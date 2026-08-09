---
date: 2026-06-06
categories:
    - Deep Dive
    - ASP.NET Core
    - OpenApi
---

# **Deep dive into OpenApi in ASP.NET Core**

In this post I'll dive into the OpenAPI, we'll explore what OpenAPI is, why it was created, and how OpenAPI was integrated in ASP.NET Core. <!-- more --> By the end of this post, you’ll feel more confident about documenting your API.

Well, lets dive in!

---

## Table of Contents

1. [What OpenAPI is?](#what-openapi-is)
2. [What does it brings to the table?](#what-does-it-brings-to-the-table)
3. [How OpenAPI was integrated in ASP.NET Core?](#how-openapi-was-integrated-in-aspnet-core)
4. [OpenAPI structure and naming conventions](#openapi-structure-and-naming-conventions)
   - [OpenAPI version](#openapi-version)
   - [OpenAPI information](#openapi-information)
   - [OpenAPI servers](#openapi-servers)
   - [OpenAPI paths](#openapi-paths)
   - [OpenAPI HTTP request methods](#openapi-http-request-methods)
   - [OpenAPI tags](#openapi-tags)
   - [OpenAPI responses](#openapi-responses)
   - [OpenAPI parameters](#openapi-parameters)
5. [Customize the OpenAPI document](#customize-the-openapi-document)
   - [Use different OpenAPI document format](#use-different-openapi-document-format)
   - [Customize the OpenAPI document name](#customize-the-openapi-document-name)
   - [Customize the OpenAPI endpoint route](#customize-the-openapi-endpoint-route)
   - [Generate multiple OpenAPI documents](#generate-multiple-openapi-documents)
   - [Generate OpenAPI documents at build time](#generate-openapi-documents-at-build-time)
   - [Customize the OpenAPI document using transformers](#customize-the-openapi-document-using-transformers)
     - [Transformers structure](#transformers-structure)
     - [Transformer parameters](#transformer-parameters)
     - [Ways to register the transformers](#ways-to-register-the-transformers)
     - [Execution order for transformers](#execution-order-for-transformers)
   - [Open API generation pipeline](#open-api-generation-pipeline)
6. [Use Document transformers](#use-document-transformers)
7. [Use Operation transformers](#use-operation-transformers)
8. [Use Schema transformers](#use-schema-transformers)
9. [Generate the UI using Scalar](#generate-the-ui-using-scalar)
---

## What OpenAPI is?

Lets clarify one simple question - "What OpenAPI is?". **OpenAPI** is just a way to document HTTP APIs so it can be understood by humans as well as computers, to be readable by both humans and computers we need a proper format - **OpenAPI** creators decided to use `JSON` or `YAML` since those formats are well-known and widely used.

---

## What does it brings to the table?

**OpenAPI** is a standart, it can be compared to another standarts like **Git** for source control or **HTTP** protocol for web, it's just a way to standardize an API documentation format so potentially every human and computer will understand it reducing issues with compatibility (imagine if there was 10 various API specification standarts - which one to use? Will this library for creating UI work with that specification? What if I need to swith to another library? Will it support my current document?).

---

## How OpenAPI was integrated in ASP.NET Core?

There is three key aspects for integrating OpenAPI specification in ASP.NET Core:

- Generating information about the endpoints in the app.
- Gathering the information into a format that matches the OpenAPI schema.
- Exposing the generated OpenAPI document through a visual UI or a serialized file.

To complete 1 and 2 aspects Microsoft introduced [Microsoft.AspNetCore.OpenApi](https://www.nuget.org/packages/Microsoft.AspNetCore.OpenApi) package, the last one was delegated to the third-party libraries like [Scalar](https://scalar.com/).

Let's see a practical example to understand it more deeply, look at this simple `Program.cs` file:


```csharp
var builder = WebApplication.CreateBuilder();

builder.Services.AddOpenApi(); // 👈️

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi(); // 👈️
}

app.UseHttpsRedirection();

app.MapGet("/posts", () => "get posts");

app.Run();
```

Meaning:

`builder.Services.AddOpenApi()` - registers services required for **OpenAPI** document generation into the application's DI container.

`app.MapOpenApi();` - adds an endpoint into the application for viewing the OpenAPI document serialized into `JSON` (restricted to the `Development` environment unavailable in production).


If you run the application

```
dotnet run --launch-profile https
```

and open the `https://localhost:5001/openapi/v1.json`

_(In this case I'm using port `5001`, change the port if needed)_

You'll see this output in `JSON` format:

```json
{
  "openapi": "3.1.1",
  "info": {
    "title": "demo | v1",
    "version": "1.0.0"
  },
  "servers": [
    {
      "url": "https://localhost:5001/"
    }
  ],
  "paths": {
    "/posts": {
      "get": {
        "tags": [
          "demo"
        ],
        "responses": {
          "200": {
            "description": "OK",
            "content": {
              "text/plain": {
                "schema": {
                  "type": "string"
                }
              }
            }
          }
        }
      }
    }
  },
  "tags": [
    {
      "name": "demo"
    }
  ]
}
```

This is an **OpenAPI** documentation, third-party libraries like Scalar are utilizing it to render the UI, for the sake of simplicity it's contains 1 `GET /posts` endpoint that returns `"get posts"`.

---

## OpenAPI structure and naming conventions

You can customize every section in the OpenAPI documentation as you prefer, however, before customizing the document, I’ll first give you a clear and deep explanation of every section inside the **OpenAPI** specification and what each one is responsible for.

### OpenAPI version

```json
"openapi": "3.1.1"
```

The first key-value pair describes the version of the specification that is being used. It's important to know, that by the time I'm typing this text the latest version of **OpenAPI** specification is [v3.2.0](https://spec.openapis.org/oas/latest.html).

The fact is that the existence of version **v3.2.0+** does not guarantee that tools such as ASP.NET Core, libraries, Scalar, Swagger etc. are already supporting it. The point is that a standard like **OpenAPI** is slow-moving, and changes to the specification and its structure lead to inevitable breaking changes, which break libraries and UI generators, so don't be surprised if a library, tool or some tutorial "doesn't keep up" with specification versions.

> For more details about available OpenAPI Specification versions see [docs](https://spec.openapis.org/oas/)

### OpenAPI information

```json
  "info": {
    "title": "demo | v1",
    "version": "1.0.0"
  },
```

This value represents the `"title"` and `"version"` of your API documentation. The `"title"` shows a simple text and the `"version"` informs the consumer about the version of API he's utilizisng.

### OpenAPI servers

```json
  "servers": [
    {
      "url": "https://localhost:5001/"
    }
  ],
```
The `"servers"` key stores the URL where our API documentation is available. This URL is important as the tool for rendering the UI like Scalar will use that URL to send the requests to the endpoints, so if our documentation has 1 endpoint, and it's structure looks like this:

```json
"/posts": {
  "get": { ... }
}
```
the resulting request will be sent to the

```
GET https://localhost:5001/posts
```
The main thing to know is that **OpenAPI** documentation is built during runtime by default. Because of this, the "servers" value is dynamic and the documentation becomes "request-aware".

### OpenAPI paths

```json
"paths": {
    "/posts": {
      "get": {
        "tags": [
          "demo"
        ],
        "responses": {
          "200": {
            "description": "OK",
            "content": {
              "text/plain": {
                "schema": {
                  "type": "string"
                }
              }
            }
          }
        }
      }
    }
  },
```

This section describes all our API endpoints, each endpoint has a lot of values that are describing it in details: path, HTTP request method, tags and responses. Let's analyze this section in details with examples.

Understand that **OpenAPI** specification organizes documentation by operations, which are grouped by paths (endpoints). Currently our application has `GET https://localhost:5001/posts` endpoint, let's add `POST` endpoin to see the different **OpenAPI** documentation output:


```diff
var builder = WebApplication.CreateBuilder();

builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.UseHttpsRedirection();
app.MapGet("/posts", () => "get posts");
+ app.MapGet("/users", () => "get users"); // 👈️ Add new GET endpoint

app.Run();
```

After running the application the `"paths"` section will be updated, now it consits 2 different endpoints:

```diff
  "paths": {
    "/posts": { ... },
+   "/users": { ... }
  },
```

### OpenAPI HTTP request methods

Now let's analyze the details of the path in our **OpenAPI** document by taking the `"/post"` endpoint as an example:

```json
  "paths": {
    "/posts": {
      "get": { ... }
    },
  },
```

Each HTTP endpoint has its own method name, which determines what type of **CRUD** operation a particular endpoint belongs to: `GET`, `POST`, `PUT`, `DELETE` etc. If we add a new endpoint with the same path, but different method names we'll get 2 endpoints within the same path `"/posts"`.

Let's add a new endpoint into our application to see that in action:

```diff
var builder = WebApplication.CreateBuilder();

builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.UseHttpsRedirection();
app.MapGet("/posts", () => "get posts");
+ app.MapPost("/posts", () => "create post");
- app.MapGet("/users", () => "get users");

app.Run();
```

After the application re-run:

```json
  "paths": {
    "/posts": {
      "get": { ... },
      "post": { ... }
    }
  },

```

Now we have 2 different endpoints within the same path `"/posts"`, it shows us how **OpenAPI** groups endpoints by path with different HTTP request methods.

> Learn more about HTTP request methods [here](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods)

### OpenAPI tags

Each path has it's set of `"tags"`

```json
"/posts": {
  "get": {
    "tags": [
      "demo"
    ],
    ...
  },
}
```

`"tags"` in **OpenAPI** are used to logically group endpoints within documentation. Tools for rendering UI like Scalar are using tags to group related operations into sections, such as posts, auth, etc., to make the documentation structured and more readable. We'll see why tags are helpful when we'll render the UI using **OpenAPI** document.

### OpenAPI responses

Each endpoint in an **OpenAPI** document contains detailed information about the responses it can return:

```json
"paths": {
  "/posts": {
    "get": {
      "tags": [
        "demo"
      ],
      "responses": {
        "200": {
          "description": "OK",
          "content": {
            "text/plain": {
              "schema": {
                "type": "string"
              }
            }
          }
        }
      }
    }
  }
}
```

Since an endpoint can return multiple HTTP status codes with different response bodies, **OpenAPI** represents `"responses"` as a collection of possible responses keyed by status code.

You should be familiar with the most common HTTP status codes:

- `"200"` - success
- `"404"` - resource was not found
- `"500"` - internal server error

Each status code contains details describing the response.

In order to see how **OpenAPI** specification documents different HTTP responses we'll add those lines to our endpoint:

```diff
var builder = WebApplication.CreateBuilder();

builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.UseHttpsRedirection();

app.MapGet("/posts", () => "get posts")
+   .Produces<string>(StatusCodes.Status200OK)
+   .Produces<string>(StatusCodes.Status500InternalServerError);
- app.MapPost("/posts", () => "create post");

app.Run();
```

The resulting output will be:

```json
  "paths": {
    "/posts": {
      "get": {
        "tags": [
          "demo"
        ],
        "responses": {
          "200": {
            ...
          },
          "500": {
            ...
          }
        }
      }
    }
  },
```

Now we see that one endpoint can return different status codes and different information. Now let's analyze the details of the responses structure.

#### `"description"`

```json
"description": "OK"
```

A short human-readable explanation of the response. In this case, `200 OK` means that the request was successfully processed.

#### `"content"`

```json
"content": {
  "text/plain": {
    "schema": {
      "type": "string"
    }
  }
}
```

Describes the actual response body returned by the endpoint.

The `content` section contains:

- media type (`text/plain`, `application/json`, etc.)
- schema describing the structure of the returned data

Let's analyze `"content"` section in details:

#### `"text/plain"`

```json
"text/plain": { ... }
```

Specifies the MIME type (media type) of the response. In this example, the endpoint returns plain text. Commonly used media types are:

- `application/json`
- `text/plain`
- `application/xml`

as you can see, we can also return the output in a `JSON` format.

#### `"schema"`

```json
"schema": {
  "type": "string"
}
```

Defines the structure and data type of the response body, in this case the response body is a primitive `string` value.

### OpenAPI parameters

There is also a `"parameters"` section we didn't see, to introduce it, define a new endpoint with the following code:

```diff
var builder = WebApplication.CreateBuilder();

builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.UseHttpsRedirection();

app.MapGet("/posts", () => "get posts");
+ app.MapPut("/posts/{id}", (int id) => $"update {id} post");

app.Run();
```

After re-running the application we'll see that the structure of that new enpoint slightly differs from the previous endpoints.

```diff
  "paths": {
    "/posts/{id}": {
      "put": {
        "tags": [
          "demo"
        ],
+        "parameters": [
+          {
+            "name": "id",
+            "in": "path",
+            "required": true,
+            "schema": {
+              "pattern": "^-?(?:0|[1-9]\\d*)$",
+              "type": [
+                "integer",
+                "string"
+              ],
+              "format": "int32"
+            }
+          }
+        ],
        "responses": {
          "200": {
            "description": "OK",
            "content": {
              "text/plain": {
                "schema": {
                  "type": "string"
                }
              }
            }
          }
        }
      }
    }
  },
```

Now we see a new section with `"parameters"` key. This section describes all parameters the endpoint expects from the client in order to successfully process the request.

In our example, the endpoint

```
PUT /posts/{id}
```

contains a route parameter named `{id}`. ASP.NET Core automatically detects this parameter from the route template and includes it in the generated **OpenAPI** document.

The `"name"` field represents the name of the parameter:

```json
"name": "id"
```

The `"in"` field specifies where the parameter comes from. In our case, the parameter is part of the URL path, therefore its location is `"path"`:

```json
"in": "path"
```

The `"required"` field indicates whether the parameter is mandatory. Path parameters **are always required** because the route cannot be matched without them:

```json
"required": true
```

The `"schema"` section describes the expected data type and validation rules for the parameter.

```json
"schema": {
  "pattern": "^-?(?:0|[1-9]\\d*)$",
  "type": [
    "integer",
    "string"
  ],
  "format": "int32"
}
```

In this example:

- `"format": "int32"` means the parameter should represent a 32-bit integer.
- `"pattern"` contains additional validation rules generated by ASP.NET Core.
    - `"^-?(?:0|[1-9]\\d*)$"`

        this validation rule ensures that the parameter value represents a valid integer number.

- `"type"` describes the underlying `JSON` representation of the value.

---

## Customize the OpenAPI document

Now let's see how we can customize the **OpenAPI** document, I'll start from basic customization and then we'll get into the transformers.

### Use different OpenAPI document format

If you want to use the `YAML` format instead of `JSON` you'll need to specify it in the `Program.cs` like this:


```diff
var builder = WebApplication.CreateBuilder();

builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
+   app.MapOpenApi("/openapi/{documentName}.yaml");
//               Specify the URL with .yaml  👆️
}

app.UseHttpsRedirection();

app.MapGet("/posts", () => "get posts");

app.Run();
```
Now the URL to access the document will have `.yaml` in the end of the URL:

```
https://localhost:5001/openapi/v1.yaml
```

As expected, the output will be in `YAML` format:

```yaml
openapi: '3.1.1'
info:
  title: demo | v1
  version: 1.0.0
servers:
  - url: https://localhost:5001/
paths:
  /posts:
    get:
      tags:
        - demo
      responses:
        '200':
          description: OK
          content:
            text/plain:
              schema:
                type: string
tags:
  - name: demo
```

### Customize the OpenAPI document name

Each **OpenAPI** document in your application has it's unique name

```csharp
builder.Services.AddOpenApi(); // 👈️ Default name is v1

```
The document name can be modified by passing the name as a parameter to the AddOpenApi call:

 ```csharp
 builder.Services.AddOpenApi("custom");

 ```

 Now the URL to access the resulting **OpenAPI** document will also change:

```
https://localhost:5001/openapi/custom.json

```
The resulting document will have those changeis in the `"info"` section:

```json
{
  "openapi": "3.1.1",
  "info": {
    "title": "demo | custom", 👈️ Here is our value
    "version": "1.0.0"
  },
  ...
}
```
It's important to understand, that the "demo" in the "title" was generated based on the project name, if you want to change it to something different you'll need to use transformers, we'll get to them in a minute.

### Customize the OpenAPI endpoint route

You can customize the documentation URL as shown above defining the new URL path in `.MapOpenApi()` call:

```csharp
app.MapOpenApi("/my/documentation/openapi.json");
```
And believe me or not, it will work and you'll be able to access the document.


### Generate multiple OpenAPI documents

In some cases you'll need to generate multiple **OpenAPI** documentations, because **OpenAPI** documentation will be needed for:

- Audiences, such as public and internal APIs.
- Versions of an API.
- Parts of an app, such as a frontend and backend API.

To generate multiple **OpenAPI** documents, call the `.AddOpenApi()` method once for each document, specifying a different document name:

```csharp
builder.Services.AddOpenApi("v1");
builder.Services.AddOpenApi("v2");

```

To the different documents will have different URL's:

```
https://localhost:5001/openapi/v1.json

https://localhost:5001/openapi/v2.json

```

Each invocation of `.AddOpenApi()` can specify its own set of options, so you can choose to use the same or different customizations for each **OpenAPI** document.

The framework uses the `ShouldInclude` delegate method of `OpenApiOptions` to determine which endpoints to include in each document.

To visualize it more clearly, consider the following code in your  `Program.cs` file:

```csharp
var builder = WebApplication.CreateBuilder();

// Include "backend" endpoints
builder.Services.AddOpenApi("backend", options =>
{
    options.ShouldInclude = desc => desc.GroupName == "backend";
});

// Include "frontend" endpoints
builder.Services.AddOpenApi("frontend", options =>
{
    options.ShouldInclude = desc => desc.GroupName == "frontend";
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}
app.UseHttpsRedirection();

// Add the endpoint to the "backend" group
app.MapGet("/backend/posts", () => "backend posts")
    .WithGroupName("backend");

// Add the endpoint to the frontend group
app.MapGet("/frontend/posts", () => "frontend posts")
    .WithGroupName("frontend");

app.Run();
```

Now each **OpenApi** document is accessible from it's own URL:

```
https://localhost:5001/openapi/backend.json

https://localhost:5001/openapi/frontend.json

```

### Generate OpenAPI documents at build time

By default, the OpenAPI document is generated at runtime, which means a new, up-to-date document is produced every time the application starts.

In some scenarios, it's helpful to generate the **OpenAPI** document during the app's build time, for example if **OpenAPI** documentation needs to be:

- Committed into source control.
- Used for spec-based integration testing.
- Served statically from the web server.

The [Microsoft.Extensions.ApiDescription.Server](https://www.nuget.org/packages/Microsoft.Extensions.ApiDescription.Server) package enables build-time document generation.

After your project was built:

```
dotnet build
```

The place where you can find your generated document is

```
obj/{ProjectName}.json
```

---

## Customize the OpenAPI document using transformers

Transformers provide a useful API for modifying the **OpenAPI** document with your customizations.

The common scenarios to use transformers:

- Adding parameters to all operations in a document.
- Modifying descriptions for parameters or operations.
- Adding top-level information to the **OpenAPI** document.

There are 3 categories of transformers:

|  Document | Operation   | Schema  |
|---|---|---|
|Access to the entire OpenAPI document. These can be used to make global modifications to the document.   | Apply to each individual operation. Each individual operation is a combination of path and HTTP method. These can be used to modify parameters or responses on endpoints.  | Apply to each schema in the document. These can be used to modify the schema of request or response bodies, or any nested schemas. |

Let's analyze each transformers category with examples.

---

### Transformers structure

Before creating our own transformers let's analyze their structure. We have 3 categories of transformers, hence there are 3 interfaces for each transformer:

- `IOpenApiDocumentTransformer` (Document transformer).

```csharp
class MyDocumentTransformer : IOpenApiDocumentTransformer
{
    public Task TransformAsync(OpenApiDocument document, OpenApiDocumentTransformerContext context,
        CancellationToken cancellationToken)
    {
        // ...
    }
}
```

- `IOpenApiOperationTransformer` (Operation transformer).

```csharp
class MyOperationTransformer : IOpenApiOperationTransformer
{
    public Task TransformAsync(OpenApiOperation operation, OpenApiOperationTransformerContext context, CancellationToken cancellationToken)
    {
        // ...
    }
}
```

- `IOpenApiSchemaTransformer` (Schema transformer).

```csharp
class MySchemaTransformer : IOpenApiSchemaTransformer
{
    public Task TransformAsync(OpenApiSchema schema, OpenApiSchemaTransformerContext context, CancellationToken cancellationToken)
    {
        // ...
    }
}
```

---

### Transformer parameters

Let's analyze the parameters of those transformers.

Each transformer receives three parameters that represent the current state of the OpenAPI generation pipeline.

#### `OpenApiDocument` | `OpenApiOperation` | `OpenApiSchema`

The first parameter represents the part of the **OpenAPI** document being modified.

Depending on the transformer type:

- `IOpenApiDocumentTransformer` - works with the entire `OpenApiDocument`
- `IOpenApiOperationTransformer` - works with a single `OpenApiOperation`
- `IOpenApiSchemaTransformer` - works with a single `OpenApiSchema`

This parameter is mutable, meaning transformers can modify its structure (add, remove, or update properties).


#### `OpenApiDocumentTransformerContext` | `OpenApiOperationTransformerContext` | `OpenApiSchemaTransformerContext`

Each transformer also receives a context object that provides additional metadata about the current item being processed.

For example, the context may include:

- information about the current endpoint (for operation transformers)
- information about the associated API description
- the document name being generated
- access to dependency injection services (`IServiceProvider`)

etc.

#### `CancellationToken`

The `CancellationToken` allows the transformation process to be cancelled if the request is aborted or the application is shutting down.

Basically, these three parameters allow transformers:

- inspect the current **OpenAPI** model
- access runtime metadata from ASP.NET Core
- safely modify the document during generation

---

### Ways to register the transformers

There are 3 ways to register transformers:

- using a delegate
- using an instance of interface
- using a DI

Here is the visual example with Document transformer:

```csharp
using Microsoft.AspNetCore.OpenApi;
using Microsoft.OpenApi;

var builder = WebApplication.CreateBuilder();

builder.Services.AddOpenApi(options =>
{
    // Register using a delegate
    options.AddDocumentTransformer((document, context, cancellationToken) => Task.CompletedTask);
    // Register using an instance of interface
    options.AddDocumentTransformer(new MyDocumentTransformer());
    // Register using a DI
    options.AddDocumentTransformer<MyDocumentTransformer>();
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.UseHttpsRedirection();

app.MapGet("/posts", () => "get posts");
app.Run();

// Dummy Document transformer
class MyDocumentTransformer : IOpenApiDocumentTransformer
{
    public Task TransformAsync(OpenApiDocument document, OpenApiDocumentTransformerContext context, CancellationToken cancellationToken)
    {
        // ...

        return Task.CompletedTask;
    }
}
```

---

### Execution order for transformers

Transformers are always executed in the following order:

```text
Document transformers
      ↑
Operation transformers
      ↑
Schema transformers

```
This execution order is fixed and cannot be changed.

However, the order in which you register transformers matters for transformers of the same type. Transformers of the same category are executed sequentially in the exact order they were added.

For example, if you register two document transformers:

```csharp
options.AddDocumentTransformer<MyDocumentTransformer1>();
options.AddDocumentTransformer<MyDocumentTransformer2>();
```
Then `MyDocumentTransformer1` executes first `MyDocumentTransformer2` executes second and has access to all modifications made by the first transformer. The same rule applies to Operation and Schema transformers.

Now we'll analyze each transformer type and see how they behave through practical examples.

---

### Open API generation pipeline

The entire process of the **OpenAPI** generation can be introduced like this:

```text
JSON serialization
    ↑
Transformers execution (Schema -> Operation -> Document)
    ↑
Document generation
    ↑
Operation generation
    ↑
Schema generation
    ↑
ApiDescription metadata graph
    ↑
Endpoints

```

As you can see instead of producing the final `JSON`/`YAML` document immediately, the framework gradually builds an internal object graph that represents the entire API structure. This design allows each stage of the pipeline to modify and reuse metadata generated by previous stages.

---

### Use Document transformers

Document transformers allow you to modify the generated **OpenAPI** document globally.

Unlike Operation or Schema transformers, Document transformers work with the entire `OpenApiDocument` object, which means they can customize top-level metadata, security requirements, servers, tags, and any other part of the final document.

```csharp
class MyDocumentTransformer : IOpenApiDocumentTransformer
{
    public Task TransformAsync(OpenApiDocument document, OpenApiDocumentTransformerContext context, CancellationToken cancellationToken)
    {
        // ...
    }
}
```

Document transformers receive a context object that provides additional information about the current generation process, including:

- the name of the document being generated
- the `ApiDescriptionGroups` associated with that document
- the `IServiceProvider` used during document generation

The following example demonstrates a simple Document transformer that modifies the top-level `"info"` section of the generated **OpenAPI** document:

```csharp
var builder = WebApplication.CreateBuilder();

builder.Services.AddOpenApi(options =>
{
    options.AddDocumentTransformer((document, context, cancellationToken) =>
    {
        document.Info = new()
        {
            Title = "Demo API",
            Version = "v99",
            Description = "API for testing purposes"
        };
        return Task.CompletedTask;
    });
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.UseHttpsRedirection();

app.MapGet("/posts", () => "get posts");
app.Run();
```

Since document transformers execute after all Schemas and Operations are generated, they act as the final customization layer of the **OpenAPI** generation pipeline.

Document transformers can also use services from ASP.NET Core's DI IoC container.

This is useful when the generated **OpenAPI** document depends on the current application configuration or runtime services.

The following example demonstrates a Document transformer that uses the `IAuthenticationSchemeProvider` service to check whether `JWT Bearer` authentication is registered in the application.

If a `Bearer` authentication scheme exists, the transformer adds a security scheme definition to the generated **OpenAPI** document:

```csharp
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.OpenApi;
using Microsoft.OpenApi;

var builder = WebApplication.CreateBuilder();

builder.Services.AddAuthentication().AddJwtBearer();

builder.Services.AddOpenApi(options =>
{
    options.AddDocumentTransformer<BearerSecuritySchemeTransformer>();
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.UseHttpsRedirection();

app.MapGet("/posts", () => "get posts");

app.Run();

internal sealed class BearerSecuritySchemeTransformer(IAuthenticationSchemeProvider authenticationSchemeProvider) : IOpenApiDocumentTransformer
{
    public async Task TransformAsync(OpenApiDocument document, OpenApiDocumentTransformerContext context, CancellationToken cancellationToken)
    {
        var authenticationSchemes = await authenticationSchemeProvider.GetAllSchemesAsync();
        if (authenticationSchemes.Any(authScheme => authScheme.Name == "Bearer"))
        {
            var securitySchemes = new Dictionary<string, IOpenApiSecurityScheme>
            {
                ["Bearer"] = new OpenApiSecurityScheme
                {
                    Type = SecuritySchemeType.Http,
                    Scheme = "bearer",
                    In = ParameterLocation.Header,
                    BearerFormat = "Json Web Token"
                }
            };
            document.Components ??= new OpenApiComponents();
            document.Components.SecuritySchemes = securitySchemes;
        }
    }
}
```
After re-running the application you can see the new section in the documentation:

```json
"components": {
  "securitySchemes": {
    "Bearer": {
      "type": "http",
      "scheme": "bearer",
      "bearerFormat": "Json Web Token"
    }
  }
}
```

Let's analyze this new section. The `"components"` section contains reusable objects that can be referenced across the **OpenAPI** document. In this example, it includes the "securitySchemes" section, which defines authentication methods supported by the API.

The `"securitySchemes"` object is a registry of available authentication mechanisms. Each scheme can later be applied to individual endpoints or to the entire API.

#### `"Bearer"`

This is the name of the security scheme. It acts as an identifier that can be referenced elsewhere in the **OpenAPI** document.

#### `"type": "http"`

This indicates that the authentication mechanism is based on HTTP authentication. In this case we're using the HTTP Bearer authentication.

#### `"scheme": "bearer"`

Specifies the authentication scheme used within HTTP authentication. Here it defines `Bearer` token authentication, where the token is sent in the HTTP `Authorization` header

#### `"bearerFormat"`

Provides a human-readable hint about the token format. In this case, it indicates that the bearer token is expected to be a `JSON Web Token` (JWT).

> It's important to understand that Document transformers are unique to the document instance they're associated with. By specifying something like `builder.Services.AddOpenApi("internal", options => {...});`

The `"components"` section contains globally reusable OpenAPI objects that can be referenced throughout the document instead of being duplicated multiple times.

---

### Use Operation transformers

Operation transformers allow you to modify individual **OpenAPI** operations. You can use Operation transformers when you need to:

- modify all endpoints in the application
- apply changes only to specific endpoints

```csharp
class MyOperationTransformer : IOpenApiOperationTransformer
{
    public Task TransformAsync(OpenApiOperation operation, OpenApiOperationTransformerContext context, CancellationToken cancellationToken)
    {
        // ...
    }
}
```

Operation transformers receive a context object containing:

- the **OpenAPI** document name
- the `ApiDescription` associated with the endpoint
- the `IServiceProvider`


The following example adds a `500 Internal Server Error` response to all operations in the document:

```csharp
builder.Services.AddOpenApi(options =>
{
    options.AddOperationTransformer((operation, context, cancellationToken) =>
    {
        operation.Responses ??= new OpenApiResponses();

        operation.Responses.Add("500",
            new OpenApiResponse
            {
                Description = "Internal server error"
            });

        return Task.CompletedTask;
    });
});
```

This Operation transformer will add this response to all endpoints in the document, for the sake of simplicity I specified only 1 endpoint. The **OpenAPI** document will look like this:

```diff
  // ...
  "paths": {
    "/posts": {
      "get": {
        "tags": [
          "demo"
        ],
        "responses": {
          "200": {
          ...
          },
+         "500": {
+           "description": "Internal server error"
+         }
        }
      }
    }
  },
  // ...
```

Operation transformers can also be attached to a specific endpoint instead of the entire document. The following example marks a single endpoint as deprecated in the generated **OpenAPI** document:

```csharp
app.MapGet("/old/posts", () => "get old posts")
    .AddOpenApiOperationTransformer((operation, context, cancellationToken) =>
    {
        operation.Deprecated = true;
        return Task.CompletedTask;
    });
```

The resulting document will have a marker that this endpoint is deprecated and therefore shouldn't be used:

```diff
// ...
    "/old/posts": {
      "get": {
        "tags": [
          "demo"
        ],
        "responses": {
          "200": {
            "description": "OK",
            "content": {
              "text/plain": {
                "schema": {
                  "type": "string"
                }
              }
            }
          },
        },
+       "deprecated": true
      }
    }
// ...
```  

This marker will be shown in the tools like Scalar for generating UI. This allows endpoint-specific customization without affecting the rest of the API endpoints.

---

### Use Schema transformers

Schemas are the data models that are used in request and response bodies in an **OpenAPI** document. Schema transformers are useful when a modification:

- should be made to each schema in the document
- conditionally applied to certain schemas

Schema transformers have access to a context object which contains:

- the name of the document the schema belongs to
- the `JSON` type information associated with the target schema
- `IServiceProvider` used in document generation

The following Schema transformer sets the format of `decimal` types to `decimal` instead of `double`:

```csharp
using Microsoft.AspNetCore.OpenApi;

var builder = WebApplication.CreateBuilder();

builder.Services.AddOpenApi(options => {
    // Here is out Schema transformer to set the format of decimal to 'decimal'
    options.AddSchemaTransformer((schema, context, cancellationToken) =>
    {
        if (context.JsonTypeInfo.Type == typeof(decimal))
        {
            schema.Format = "decimal";
        }
        return Task.CompletedTask;
    });
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.UseHttpsRedirection();

app.MapGet("/", () => new Body { Amount = 1.1m });

app.Run();

//           👇️ Body class with decimal Amount property
public class Body {
    public decimal Amount { get; set; }
}
```

---

## Generate the UI using Scalar

Finally, let's see how we can generate a UI using the **OpenAPI** document. For the example I'll use Scalar, as it's the most common tool for generating a UI.

First, install the [Scalar.AspNetCore](https://www.nuget.org/packages/Scalar.AspNetCore) package.

Then, add this `app.MapScalarApiReference()` line to the `Program.cs` file:

```csharp

using Scalar.AspNetCore;

var builder = WebApplication.CreateBuilder();

builder.Services.AddAuthentication().AddJwtBearer();

builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
    app.MapScalarApiReference(); // 👈️ Enable Scalar
}

app.UseHttpsRedirection();

app.MapGet("/posts", () => "get posts");

app.Run();
```

Now, simply open the documentation with this URL:

```
https://localhost:5001/scalar/
```

_(change the port if needed)_

As a result we have a beautiful UI generated using our **OpenAPI** document. It's important to know that Scalar acts like a real API consumer, so be careful when you're making some HTTP requests accessible for Scalar.

---

[Bring me back to the top!](#)

Well, we dove really deep this time, this entire post was describing **OpenAPI** specification and how ASP.NET Core works with it.

That's it for today, thanks for reading :)