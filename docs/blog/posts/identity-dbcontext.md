---
date: 2026-04-21
hide:
  - toc
categories:
    - ASP.NET Core
    - ASP.NET Core Identity
    - EF Core
---

# **Look behind DbContext in ASP.NET Core Identity**

In this post I'll explain what's going on under the hood when you create your `ApplicationDbContext` using ASP.NET Core Identity in EF Core. <!-- more --> We'll cover where do the **AspNetUsers** and **AspNetRoles** tables actually come from and what really happens when you're inherit from `IdentityDbContext<ApplicationUser>`.

Look at the basic `ApplicationDbContext` for the to do list project.

```csharp
public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) : base(options) { }

    public DbSet<TodoList> TodoLists => Set<TodoList>();

    public DbSet<TodoItem> TodoItems => Set<TodoItem>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        builder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
    }
}

```

Can you tell what exactly happens within the chain of inheritance? What path does the `ModelBuilder builder` take? What does that simple inheritance from `IdentityDbContext<ApplicationUser>` actually brings to the table?

> Keep that statement in mind: We're passing the `ModelBuilder builder` object into this entire chain of inheritance, so EF Core builds out the entire schema. Didn't click? Keep reading :)

```
IdentityDbContext<ApplicationUser>
    ↑
ApplicationDbContext
```

Well, let's dive in!

---

**ASP.NET Core Identity** is  a membership system that provides common features like user registration, login, role management, and claims-based authentication.

For all this to become a reality, we need a Users table and a Roles table - along with several supporting tables. So, in addition to registering `DbSet` properties to interact with respective tables, our `ApplicationDbContext` must provide us with the entire schema and their respective columns:

- Roles tables
    - `AspNetRoles`
    - `AspNetRoleClaims`
    - `AspNetUserRoles`
- Users tables
    - `AspNetUsers`
    - `AspNetUserClaims`
    - `AspNetUserLogins`
    - `AspNetUserTokens`
    - `AspNetUserPasskeys`

So, **Identity should somehow create those tables for us**, right? Right :)

lets look at how it'll create them for us.

---

First of all, following the inheritance chain:

```
IdentityDbContext<ApplicationUser>
    ↑
ApplicationDbContext<IdentityUser>
```

Navigate to the source code of the:

```
IdentityDbContext<ApplicationUser> 
```

We'll get into `IdentityDbContext.cs`. For now, just understand that `IdentityDbContext` class is responsible for generating the Roles tables and their respective columns for our `IdentityUser`.

> The common misconception is that `IdentityDbContext` handles **only** roles. In reality `IdentityDbContext` is responsible for **extending** the user-centric schema with roles.

That file includes 5 versions of the `IdentityDbContext` class. At the top of the file you'll see non-generic

```
IdentityDbContext
```

This is the most basic `IdentityDbContext` class, you'd use this version in case you don't want to set your custom columns into the `IdentityUser`. 

But in our example we're using the generic version: 

```
IdentityDbContext<TUser>
```

Notice that non-generic `IdentityDbContext` and generic `IdentityDbContext<TUser>` both inherit from

```
IdentityDbContext<TUser, TRole, TKey>
```

But each of them handles the types of base class differently. 

The non-generic `IdentityDbContext` 

```
IdentityDbContext : IdentityDbContext<IdentityUser, IdentityRole, string>
```

sets the default type for `IdentityUser`, `IdentityRole` and primary key type as `string`. 

Conversely, the generic `IdentityDbContext<TUser>` 

```csharp
IdentityDbContext<TUser> : IdentityDbContext<TUser, IdentityRole, string> 
where TUser : IdentityUser
// ☝️ Set the default type for TUser

```

sets the generic type for `TUser` that must be a type of `IdentityUser`, so we can pass our own implementation of `IdentityUser` as we did right at the beginning of this post:

```csharp
//                Our custom user of type IdentityUser     👇
public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
   // ...
}
```

Look carefully at the code in this file, you'll see that the chain of inheritance looks like this: 

```
   ... (We'll dive deeper into that chain in a second)
    ↑
IdentityDbContext<TUser, TRole, TKey, TUserClaim, TUserRole, TUserLogin, TRoleClaim, TUserToken, TUserPasskey>
    ↑
IdentityDbContext<TUser, TRole, TKey, TUserClaim, TUserRole, TUserLogin, TRoleClaim, TUserToken>
    ↑
IdentityDbContext<TUser, TRole, TKey>
    ↑ 
IdentityDbContext<TUser> & IdentityDbContext
    ↑
ApplicationDbContext (which inherits from IdentityDbContext<TUser>)
```

**But why do we need to have all of those variations of the `IdentityDbContext` class? Can't we just have non-generic one?**

-> The point of having different variations of the `IdentityDbContext` class is the ability to customize our roles in the way we want.

Each variation of `IdentityDbContext` adds more specificity:

- `IdentityDbContext<TUser, TRole, TKey, TUserClaim, TUserRole, TUserLogin, TRoleClaim, TUserToken, TUserPasskey>` — the most abstract, allows you to replace all tables with your own implementations (we'll examine that class in a second).

- `IdentityDbContext<TUser, TRole, TKey, TUserClaim, TUserRole, TUserLogin, TRoleClaim, TUserToken>` - sets the standard type for (`TUserPasskey`), but leaves you free to customize `TUser, TRole, TKey, TUserClaim, TUserRole ...`

- `IdentityDbContext<TUser, TRole, TKey>` — 
sets the standard types for relationships (`UserRole`, `Claim`, `Token`), but leaves you free to customize `<TUser, TRole, TKey>`.

- `IdentityDbContext<TUser>` — sets the standard `IdentityRole` and string as a key, leaving freedom only for the custom TUser.

- `IdentityDbContext (non-generic)` — only default implementation.

Now, let's check the most flexible `IdentityDbContext` class (the top class in the chain above) and its inner logic.

If you follow the implementation of the `OnModelCreating()` from the initial code:

```csharp
public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
    // ...

    protected override void OnModelCreating(ModelBuilder builder)
    {
        //👇 Dive here
        base.OnModelCreating(builder);
        builder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
    }
}
```

The chain looks like this: 

> ✅ - `OnModelCreating()` method is present 

> ❌ - `OnModelCreating()` method is **not** present

```
       ... (We'll dive deeper into that chain in a second)
        ↑
✅ IdentityDbContext<TUser, TRole, TKey, TUserClaim, TUserRole, TUserLogin, TRoleClaim, TUserToken, TUserPasskey>
        ↑
❌ IdentityDbContext<TUser, TRole, TKey, TUserClaim, TUserRole, TUserLogin, TRoleClaim, TUserToken>
        ↑
❌ IdentityDbContext<TUser, TRole, TKey> 
        ↑ 
❌ IdentityDbContext<TUser> 
        ↑
✅ ApplicationDbContext
```

You'll land in the topmost class with an override - back to our most flexible `IdentityDbContext`:

```
IdentityDbContext<TUser, TRole, TKey, TUserClaim, TUserRole, TUserLogin, TRoleClaim, TUserToken, TUserPasskey>
```

Notice that it has a logic for those methods: 
- `OnModelCreating()` 
    - calls `base.OnModelCreating(builder)`, passing the builder object up to the parent
- `OnModelCreatingVersion3()`
- `OnModelCreatingVersion2()`
- `OnModelCreatingVersion1()`

Keep in mind that all of those methods are being overriden by this class, meaning its base class `IdentityUserContext` - defines the `virtual` versions of these methods, providing the base implementation.

**But why do we need to have 3 different versions of the same `OnModelCreatingVersion_()`? Can't we just use one version?**

-> These three versions of `OnModelCreatingVersion_()` are all about versioning the Identity database schema to avoid breaking older applications during upgrades.

Imagine you created your database schema several years ago using an older version of Identity. Later, you upgrade your project to a newer version with updated `DbContext` types.

By keeping separate schema versions, Identity ensures that older databases continue to work correctly even after update.

By default Identity uses the latest `OnModelCreatingVersion3()` version inside of the `IdentityDbContext<...>` parent class - `IdentityUserContext` 

> Although it is technically possible to target an earlier version, using the latest one is strongly recommended for new projects. We'll take a look at the source code of `IdentityUserContext` to understand how the schema version is selected in a second.

Now, lets examine that topmost `IdentityDbContext<...>` class.

> I've intentionally ommitted most of the source code, as the key part is the `OnModelCreating()` and `OnModelCreatingVersion3()` logic inside of it.

```csharp
public abstract class IdentityDbContext<TUser, TRole, TKey, TUserClaim, ...> 
: IdentityUserContext<TUser, TKey, TUserClaim, ...>
//   ☝️ Parent class
{
    //...
    
    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);// 👈 Delegate to the parent
    }

    internal override void OnModelCreatingVersion3(ModelBuilder builder)
    {
        base.OnModelCreatingVersion3(builder); // 👈 Delegate to the parent and perform its own logic below

        builder.Entity<TUser>(b =>
        {
            // fluent API that mutates the builder object.
        });

        // ...
    }
}
```

First, notice that in `OnModelCreating(ModelBuilder builder)`, this class simply delegates the execution to the base implementation:

```csharp
base.OnModelCreating(builder); 
//☝️ Delegate to the parent
```

The same pattern appears in `OnModelCreatingVersion3(ModelBuilder builder)`:

```csharp
 internal override void OnModelCreatingVersion3(ModelBuilder builder)
    {
        base.OnModelCreatingVersion3(builder);
        
        // Perform its own implementation after the parent logic
    }
```

> As well as `OnModelCreatingVersion3()` other versions: 2 and 1 are delegating the execution to the parent.

This is important as it shows that the actual version-selection logic and core configuration are defined in the base class (`IdentityUserContext`), while `IdentityDbContext` only extends that configuration providing the roles schema.

Now, before diving into the `IdentityUserContext` class, lets revisit our chain of inheritance and to understand where `IdentityDbContext` fits with its parent classs - `IdentityUserContext`

```
   ...
    ↑
IdentityUserContext<TUser, TKey, TUserClaim, TUserLogin, TUserToken, TUserPasskey>
    ↑
IdentityDbContext<TUser, TRole, TKey, TUserClaim, TUserRole, TUserLogin, TRoleClaim, TUserToken, TUserPasskey>
    ↑
IdentityDbContext<TUser, TRole, TKey, TUserClaim, TUserRole, TUserLogin, TRoleClaim, TUserToken>
    ↑
IdentityDbContext<TUser, TRole, TKey>
    ↑ 
IdentityDbContext<TUser> & IdentityDbContext
    ↑
ApplicationDbContext
```

---

Now, lets move one level up and examine the parent class - `IdentityUserContext`. 

As the name suggests, this class is responsible for configuring the **user-related** part of the Identity schema. In contrast, `IdentityDbContext` **builds on top of it and adds support for roles** and related entities.

When you navigate to the source code of `IdentityUserContext`, you'll find multiple generic variations of the class. Similar to `IdentityDbContext`, they form an internal inheritance chain:

```
   👇 Pparent class
 DbContext
    ↑
IdentityUserContext<TUser, TKey, TUserClaim, TUserLogin, TUserToken, TUserPasskey>
    ↑
IdentityUserContext<TUser, TKey, TUserClaim, TUserLogin, TUserToken>
    ↑
IdentityUserContext<TUser, TKey>
    ↑
IdentityUserContext<TUser>
```

Now, lets take a look at the implementation of the 
- `OnModelCreating()`
- `OnModelCreatingVersion()` (which contains version-selection logic) 
- `OnModelCreatingVersion3()`

> As in the previous examples, most of the source code is omitted. The focus here is on OnModelCreating() and the version-selection logic.

First, lets examine `OnModelCreating()`:

```csharp
protected override void OnModelCreating(ModelBuilder builder)
{
    var version = GetStoreOptions()?.SchemaVersion ?? IdentitySchemaVersions.Version1;
    OnModelCreatingVersion(builder, version);
}
```

This line defines which schema version Identity will use:

```csharp
var version = GetStoreOptions()?.SchemaVersion 
              ?? IdentitySchemaVersions.Version1;
```

If no schema version is explicitly configured, Identity falls back to a default value and ultimately uses the latest supported schema version (currently Version3).

This means that in most cases, you do not need to manually specify the schema version — the framework will automatically select the appropriate one.

Next, let's examine `OnModelCreatingVersion()`, where the internal version-selection logic is implemented:

```csharp
internal virtual void OnModelCreatingVersion(ModelBuilder builder, Version schemaVersion)
{
    if (schemaVersion >= IdentitySchemaVersions.Version3)
    {
        OnModelCreatingVersion3(builder);
    }
    else if (schemaVersion >= IdentitySchemaVersions.Version2)
    {
        OnModelCreatingVersion2(builder);
    }
    else
    {
        OnModelCreatingVersion1(builder);
    }
}
```

Here, the schemaVersion value (resolved in the previous step) determines which version-specific method will be executed. By default (and in our case), it leads to a call to `OnModelCreatingVersion3(builder)`.

Since `OnModelCreatingVersion3()` is a **`virtual`** method, the actual implementation that runs first is the `override` in `IdentityDbContext`:

```csharp
internal override void OnModelCreatingVersion3(ModelBuilder builder)
{
    base.OnModelCreatingVersion3(builder); // 👈 Delegate to the parent

    // ...
}
```

This means execution flows back to `IdentityDbContext`, where additional configuration is applied on top of the base identity schema.

When `OnModelCreatingVersion3()` is executed, the call to `base.OnModelCreatingVersion3()` ensures that the base class (`IdentityUserContext`) contributes its configuration to the same `ModelBuilder builder` object.

`IdentityUserContext` adds the core **user-related** entities, while `IdentityDbContext` extends the same model with **role-related** entities such as roles, user roles, and role claims.

After the base class logic is executed, `IdentityDbContext` continues execution and applies its own configuration on top of the existing model.

In this way, the same `ModelBuilder` instance is incrementally configured as it **"flows"** through the inheritance chain of contexts.

---

 This entire process demonstrates how the `ModelBuilder` is progressively configured through multiple layers of Identity before being finalized into a metadata model, which EF Core later uses to generate SQL for the application.

 I strongly recommend to read the article about [EF Core’s Internal Data Representation](https://medium.com/@a.kago1988/the-model-metadata-graph-entity-frameworks-intermediate-representation-ir-590397e00e51)

 That's it for today, thanks for reading :)