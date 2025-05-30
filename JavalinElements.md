# Javalin Elements Quick Reference

## Table of Contents
- [Basic Setup](#basic-setup)
- [HTTP Methods](#http-methods)
- [Request Handling](#request-handling)
- [Response Operations](#response-operations)
- [Path Parameters](#path-parameters)
- [Context Operations](#context-operations)

## Basic Setup

```java
// Basic Javalin Application
Javalin app = Javalin.create();     // Create Javalin instance
app.start(7000);                    // Start server on port 7000

// Application with Configuration
Javalin app = Javalin.create(config -> {
    config.enableCorsForAllOrigins(); // Enable CORS
    config.enableDevLogging();        // Enable development logging
});

// Starting and Stopping
app.start();                        // Start on default port (7000)
app.start(8080);                   // Start on specific port
app.stop();                        // Stop the server

// Basic Route Definition
app.get("/", ctx -> ctx.result("Hello World!"));
```

## HTTP Methods

```java
// GET Requests
app.get("/users", ctx -> {          // Handle GET requests
    ctx.result("Get all users");
});

// POST Requests
app.post("/users", ctx -> {         // Handle POST requests
    ctx.result("Create user");
});

// PUT Requests
app.put("/users/{id}", ctx -> {     // Handle PUT requests
    ctx.result("Update user");
});

// DELETE Requests
app.delete("/users/{id}", ctx -> {  // Handle DELETE requests
    ctx.result("Delete user");
});

// PATCH Requests
app.patch("/users/{id}", ctx -> {   // Handle PATCH requests
    ctx.result("Partially update user");
});

// Multiple HTTP Methods
app.routes(() -> {                  // Group routes
    path("/api", () -> {
        get("/users", UserController::getAll);
        post("/users", UserController::create);
    });
});
```

## Request Handling

```java
// Handler Methods
app.get("/hello", this::helloHandler);

private void helloHandler(Context ctx) {
    ctx.result("Hello from handler!");
}

// Lambda Handlers
app.get("/lambda", ctx -> {         // Inline lambda handler
    String name = ctx.queryParam("name");
    ctx.result("Hello " + name);
});

// Method References
app.get("/users", UserController::getAllUsers);
app.post("/users", UserController::createUser);

// Handler with Exception Handling
app.get("/safe", ctx -> {
    try {
        // Some operation
        ctx.result("Success");
    } catch (Exception e) {
        ctx.status(500).result("Error occurred");
    }
});
```

## Response Operations

```java
// Basic Response
ctx.result("Plain text response");   // Send text response
ctx.html("<h1>HTML Response</h1>");  // Send HTML response

// JSON Response
User user = new User("John", 25);
ctx.json(user);                     // Send JSON response

// Status Codes
ctx.status(200);                    // Set status code
ctx.status(201).result("Created");  // Status with response
ctx.status(404).result("Not Found"); // Error status

// Response Headers
ctx.header("Content-Type", "application/json");
ctx.header("Cache-Control", "no-cache");

// Redirect
ctx.redirect("/new-path");          // Redirect to new path
ctx.redirect("/login", 302);        // Redirect with status code
```

## Path Parameters

```java
// Single Path Parameter
app.get("/users/{id}", ctx -> {
    String id = ctx.pathParam("id");    // Get path parameter
    ctx.result("User ID: " + id);
});

// Multiple Path Parameters
app.get("/users/{userId}/posts/{postId}", ctx -> {
    String userId = ctx.pathParam("userId");
    String postId = ctx.pathParam("postId");
    ctx.result("User: " + userId + ", Post: " + postId);
});

// Query Parameters
app.get("/search", ctx -> {
    String query = ctx.queryParam("q");     // Get query parameter
    String page = ctx.queryParam("page", "1"); // With default value
    ctx.result("Search: " + query + ", Page: " + page);
});

// Form Parameters
app.post("/submit", ctx -> {
    String name = ctx.formParam("name");    // Get form parameter
    String email = ctx.formParam("email");
    ctx.result("Name: " + name + ", Email: " + email);
});
```

## Context Operations

```java
// Request Body Operations
app.post("/data", ctx -> {
    String body = ctx.body();           // Get raw body
    User user = ctx.bodyAsClass(User.class); // Parse JSON to object
    ctx.result("Received: " + user.getName());
});

// Request Information
app.get("/info", ctx -> {
    String method = ctx.method();       // HTTP method
    String path = ctx.path();          // Request path
    String userAgent = ctx.header("User-Agent"); // Request header
    String ip = ctx.ip();              // Client IP address
});

// Session Operations
app.get("/session", ctx -> {
    ctx.sessionAttribute("user", "john"); // Set session attribute
    String user = ctx.sessionAttribute("user"); // Get session attribute
    ctx.result("Session user: " + user);
});

// Cookie Operations
app.get("/cookies", ctx -> {
    ctx.cookie("username", "john");     // Set cookie
    String username = ctx.cookie("username"); // Get cookie
    ctx.result("Cookie value: " + username);
});

// File Upload
app.post("/upload", ctx -> {
    ctx.uploadedFiles("file").forEach(file -> {
        // Process uploaded file
        String filename = file.getFilename();
        // Save file logic here
    });
});
```
```

---