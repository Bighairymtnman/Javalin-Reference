# Javalin Advanced Concepts

## Table of Contents
- [Custom Plugins](#custom-plugins)
- [WebSockets](#websockets)
- [Server-Sent Events](#server-sent-events)
- [Custom Serialization](#custom-serialization)
- [Advanced Middleware](#advanced-middleware)
- [Security Features](#security-features)

## Custom Plugins

```java
// Custom Plugin Implementation
public class MetricsPlugin implements Plugin {
    private final Map<String, Integer> requestCounts = new ConcurrentHashMap<>();
    
    @Override
    public void apply(Javalin app) {
        // Add metrics endpoint
        app.get("/metrics", ctx -> ctx.json(requestCounts));
        
        // Track all requests
        app.before(ctx -> {
            String path = ctx.path();
            requestCounts.merge(path, 1, Integer::sum);
        });
    }
}

// Plugin Usage
Javalin app = Javalin.create(config -> {
    config.plugins.register(new MetricsPlugin());
    config.plugins.enableCors(cors -> {
        cors.add(CorsPluginConfig::anyHost);
    });
});

// Custom Configuration Plugin
public class DatabasePlugin implements Plugin {
    private final DataSource dataSource;
    
    public DatabasePlugin(DataSource dataSource) {
        this.dataSource = dataSource;
    }
    
    @Override
    public void apply(Javalin app) {
        app.attribute("dataSource", dataSource);    // Make available to handlers
    }
}
```

## WebSockets

```java
// WebSocket Handler
public class ChatHandler {
    private static final Set<WsContext> clients = ConcurrentHashMap.newKeySet();
    
    public static void onConnect(WsContext ctx) {
        clients.add(ctx);                           // Add new client
        System.out.println("Client connected: " + ctx.getSessionId());
    }
    
    public static void onMessage(WsContext ctx) {
        String message = ctx.message();             // Get message from client
        
        // Broadcast to all connected clients
        clients.forEach(client -> {
            if (client.session.isOpen()) {
                client.send(message);               // Send to each client
            }
        });
    }
    
    public static void onClose(WsContext ctx) {
        clients.remove(ctx);                       // Remove disconnected client
        System.out.println("Client disconnected");
    }
}

// WebSocket Configuration
app.ws("/chat", ws -> {
    ws.onConnect(ChatHandler::onConnect);          // Client connects
    ws.onMessage(ChatHandler::onMessage);          // Message received
    ws.onClose(ChatHandler::onClose);              // Client disconnects
    ws.onError(ctx -> System.err.println("WebSocket error"));
});

// Advanced WebSocket with Authentication
app.ws("/secure-chat", ws -> {
    ws.onConnect(ctx -> {
        String token = ctx.queryParam("token");
        if (!isValidToken(token)) {
            ctx.closeSession(1008, "Invalid token");
            return;
        }
        clients.add(ctx);
    });
});
```

## Server-Sent Events

```java
// SSE Handler for Real-time Updates
public class NotificationHandler {
    public static void streamNotifications(Context ctx) {
        ctx.contentType("text/event-stream");
        ctx.header("Cache-Control", "no-cache");
        
        // Send periodic updates
        ScheduledExecutorService executor = Executors.newSingleThreadScheduledExecutor();
        executor.scheduleAtFixedRate(() -> {
            try {
                String data = "data: " + getCurrentTime() + "\n\n";
                ctx.result(data);                   // Send SSE data
            } catch (Exception e) {
                executor.shutdown();                // Stop on error
            }
        }, 0, 1, TimeUnit.SECONDS);
    }
}

// SSE Route
app.get("/notifications", NotificationHandler::streamNotifications);

// Advanced SSE with Custom Events
app.get("/events", ctx -> {
    ctx.contentType("text/event-stream");
    
    // Send named events
    String event = "event: userUpdate\n" +
                  "data: {\"userId\": 123, \"status\": \"online\"}\n\n";
    ctx.result(event);
});
```

## Custom Serialization

```java
// Custom JSON Mapper
public class CustomJsonMapper implements JsonMapper {
    private final ObjectMapper objectMapper;
    
    public CustomJsonMapper() {
        this.objectMapper = new ObjectMapper();
        objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        objectMapper.setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
    }
    
    @Override
    public String toJsonString(Object obj, Type type) {
        try {
            return objectMapper.writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
    }
    
    @Override
    public <T> T fromJsonString(String json, Type targetType) {
        try {
            return objectMapper.readValue(json, objectMapper.constructType(targetType));
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
    }
}

// Apply Custom Mapper
Javalin app = Javalin.create(config -> {
    config.jsonMapper(new CustomJsonMapper());
});

// Custom Response Wrapper
public class ApiResponse<T> {
    private final boolean success;
    private final T data;
    private final String message;
    
    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(true, data, null);
    }
    
    public static <T> ApiResponse<T> error(String message) {
        return new ApiResponse<>(false, null, message);
    }
}

// Usage in Controllers
app.get("/users", ctx -> {
    List<User> users = userService.getAllUsers();
    ctx.json(ApiResponse.success(users));          // Wrapped response
});
```

## Advanced Middleware

```java
// Rate Limiting Middleware
public class RateLimitMiddleware {
    private final Map<String, List<Long>> requestTimes = new ConcurrentHashMap<>();
    private final int maxRequests = 100;
    private final long timeWindow = 60000; // 1 minute
    
    public void rateLimit(Context ctx) {
        String clientIp = ctx.ip();
        long currentTime = System.currentTimeMillis();
        
        requestTimes.computeIfAbsent(clientIp, k -> new ArrayList<>());
        List<Long> times = requestTimes.get(clientIp);
        
        // Remove old requests outside time window
        times.removeIf(time -> currentTime - time > timeWindow);
        
        if (times.size() >= maxRequests) {
            ctx.status(429).result("Rate limit exceeded");
            return;
        }
        
        times.add(currentTime);
    }
}

// Caching Middleware
public class CacheMiddleware {
    private final Map<String, CacheEntry> cache = new ConcurrentHashMap<>();
    
    public void cacheResponse(Context ctx) {
        String cacheKey = ctx.method() + ":" + ctx.path();
        CacheEntry entry = cache.get(cacheKey);
        
        if (entry != null && !entry.isExpired()) {
            ctx.result(entry.getData());               // Return cached response
            ctx.header("X-Cache", "HIT");
            return;
        }
        
        // Store response for caching (simplified)
        ctx.attribute("cacheKey", cacheKey);
    }
}

// Request Validation Middleware
public class ValidationMiddleware {
    public static void validateJson(Context ctx) {
        try {
            String body = ctx.body();
            if (body.isEmpty()) {
                ctx.status(400).result("Request body required");
                return;
            }
            
            // Validate JSON format
            new ObjectMapper().readTree(body);
        } catch (Exception e) {
            ctx.status(400).result("Invalid JSON format");
        }
    }
}
```

## Security Features

```java
// JWT Authentication
public class JwtAuth {
    private static final String SECRET = "your-secret-key";
    
    public static String generateToken(User user) {
        return Jwts.builder()
            .setSubject(user.getUsername())
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + 86400000)) // 24 hours
            .signWith(SignatureAlgorithm.HS256, SECRET)
            .compact();
    }
    
    public static void authenticate(Context ctx) {
        String token = ctx.header("Authorization");
        
        if (token == null || !token.startsWith("Bearer ")) {
            ctx.status(401).result("Missing or invalid token");
            return;
        }
        
        try {
            Claims claims = Jwts.parser()
                .setSigningKey(SECRET)
                .parseClaimsJws(token.substring(7))
                .getBody();
            
            ctx.attribute("username", claims.getSubject());
        } catch (JwtException e) {
            ctx.status(401).result("Invalid token");
        }
    }
}

// Role-Based Access Control
public class RoleMiddleware {
    public static Handler requireRole(String requiredRole) {
        return ctx -> {
            String userRole = ctx.attribute("userRole");
            if (!requiredRole.equals(userRole)) {
                ctx.status(403).result("Insufficient permissions");
                return;
            }
        };
    }
}

// CORS Configuration
app.before(ctx -> {
    ctx.header("Access-Control-Allow-Origin", "*");
    ctx.header("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS");
    ctx.header("Access-Control-Allow-Headers", "Content-Type, Authorization");
});

// Security Headers
app.after(ctx -> {
    ctx.header("X-Content-Type-Options", "nosniff");
    ctx.header("X-Frame-Options", "DENY");
    ctx.header("X-XSS-Protection", "1; mode=block");
});

// Input Sanitization
public class SecurityUtils {
    public static String sanitizeInput(String input) {
        return input.replaceAll("[<>\"']", "");     // Basic XSS prevention
    }
    
    public static boolean isValidEmail(String email) {
        return email.matches("^[A-Za-z0-9+_.-]+@(.+)$");
    }
}
```
