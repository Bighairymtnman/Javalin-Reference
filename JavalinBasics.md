# Javalin Basics Quick Reference

## Table of Contents
- [Application Structure](#application-structure)
- [Route Organization](#route-organization)
- [Middleware](#middleware)
- [Error Handling](#error-handling)
- [Controllers](#controllers)
- [JSON Handling](#json-handling)

## Application Structure

```java
// Basic Application Setup
public class App {
    private Javalin app;
    
    public App() {
        this.app = Javalin.create();    // Create Javalin instance
        configureRoutes();              // Set up all routes
    }
    
    private void configureRoutes() {
        app.get("/", ctx -> ctx.result("Hello World"));
        app.start(7000);               // Start server on port 7000
    }
}

// Application with Configuration
Javalin app = Javalin.create(config -> {
    config.enableCorsForAllOrigins();  // Enable CORS
    config.enableDevLogging();         // Enable request logging
});
```

## Route Organization

```java
// Grouping Related Routes
app.routes(() -> {
    path("/api", () -> {               // Group under /api prefix
        get("/users", UserController::getAll);
        post("/users", UserController::create);
        
        path("/users/{id}", () -> {    // Nested path grouping
            get("/", UserController::getById);
            delete("/", UserController::delete);
        });
    });
});

// Route with Multiple HTTP Methods
app.get("/users/{id}", ctx -> { /* GET logic */ });
app.put("/users/{id}", ctx -> { /* PUT logic */ });
app.delete("/users/{id}", ctx -> { /* DELETE logic */ });
```

## Middleware

```java
// Before Middleware (runs before route handler)
app.before("/api/*", ctx -> {
    String token = ctx.header("Authorization");
    if (token == null) {
        ctx.status(401).result("Unauthorized");
        return;                        // Stop processing
    }
});

// After Middleware (runs after route handler)
app.after(ctx -> {
    ctx.header("X-Response-Time", "100ms");  // Add response header
});

// Global Middleware
app.before(ctx -> {
    System.out.println("Request: " + ctx.method() + " " + ctx.path());
});
```

## Error Handling

```java
// Exception Handling
app.exception(IllegalArgumentException.class, (e, ctx) -> {
    ctx.status(400).result("Bad request: " + e.getMessage());
});

app.exception(SQLException.class, (e, ctx) -> {
    ctx.status(500).result("Database error");
    e.printStackTrace();               // Log for debugging
});

// HTTP Error Handling
app.error(404, ctx -> {
    ctx.result("Page not found");      // Custom 404 page
});

app.error(500, ctx -> {
    ctx.result("Internal server error");
});
```

## Controllers

```java
// Controller Class Pattern
public class UserController {
    private static UserService userService = new UserService();
    
    // Static method for route handler
    public static void getAll(Context ctx) {
        List<User> users = userService.getAllUsers();
        ctx.json(users);               // Send JSON response
    }
    
    public static void getById(Context ctx) {
        int id = Integer.parseInt(ctx.pathParam("id"));  // Get path parameter
        User user = userService.getUserById(id);
        
        if (user != null) {
            ctx.json(user);
        } else {
            ctx.status(404).result("User not found");
        }
    }
    
    public static void create(Context ctx) {
        User user = ctx.bodyAsClass(User.class);  // Parse JSON to object
        User created = userService.createUser(user);
        ctx.status(201).json(created); // 201 Created status
    }
}
```

## JSON Handling

```java
// Sending JSON Response
User user = new User("John", "john@email.com");
ctx.json(user);                        // Automatically converts to JSON

// Receiving JSON Request
User user = ctx.bodyAsClass(User.class);  // Parse JSON body to User object

// Manual JSON Operations
String jsonString = ctx.body();        // Get raw JSON string
ctx.result("{\"message\": \"success\"}");  // Send raw JSON string

// Working with Lists
List<User> users = Arrays.asList(user1, user2);
ctx.json(users);                       // Send JSON array

// Status Codes with JSON
ctx.status(201).json(newUser);         // Created
ctx.status(400).json(errorMessage);    // Bad Request
ctx.status(404).result("Not found");   // Not Found
```
