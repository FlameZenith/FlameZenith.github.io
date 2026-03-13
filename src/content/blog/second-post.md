---
title: 'Building REST APIs with Spring Boot: A Practical Guide'
description: 'A hands-on guide to building clean, production-ready REST APIs with Spring Boot — covering project structure, best practices, and common patterns.'
pubDate: 'Mar 05 2026'
---

Spring Boot makes building Java-based REST APIs remarkably productive. But there's a difference between a working API and a **well-structured, maintainable** one. Here's what I've learned from building APIs in production.

## Project Structure That Scales

A clean project structure is the foundation of maintainable code. Here's the layered architecture I follow:

```
src/main/java/com/example/app/
├── controller/       # REST endpoints
├── service/          # Business logic
├── repository/       # Data access (JPA)
├── model/
│   ├── entity/       # Database entities
│   ├── dto/          # Data transfer objects
│   └── mapper/       # Entity ↔ DTO mappers
├── exception/        # Custom exceptions + global handler
├── config/           # Security, CORS, etc.
└── util/             # Helpers and constants
```

Each layer has a clear responsibility. Controllers handle HTTP concerns. Services contain business logic. Repositories talk to the database. **Never let a controller call a repository directly.**

## Writing Clean Controllers

A controller should be thin — just route the request, validate input, and delegate to the service layer:

```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @GetMapping("/{id}")
    public ResponseEntity<UserDTO> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }

    @PostMapping
    public ResponseEntity<UserDTO> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        UserDTO created = userService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
}
```

Key practices:
- **Use DTOs**, never expose entities directly
- **Version your API** (`/api/v1/...`)
- **Use proper HTTP status codes** — 201 for creation, 204 for deletion, 404 for not found
- **Validate input** with `@Valid` and Bean Validation annotations

## Global Exception Handling

Don't scatter try-catch blocks everywhere. Use a `@RestControllerAdvice` for centralized error handling:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));
        ErrorResponse error = new ErrorResponse(
            HttpStatus.BAD_REQUEST.value(), message
        );
        return ResponseEntity.badRequest().body(error);
    }
}
```

## Pagination Done Right

For list endpoints, always support pagination:

```java
@GetMapping
public ResponseEntity<Page<UserDTO>> getUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "createdAt") String sortBy) {
    Pageable pageable = PageRequest.of(page, size, Sort.by(sortBy).descending());
    return ResponseEntity.ok(userService.findAll(pageable));
}
```

Spring Data's `Page` object automatically includes metadata like total pages, total elements, and navigation info.

## Essential Practices

1. **Use profiles** — `application-dev.yml`, `application-prod.yml` for environment-specific config
2. **Externalize secrets** — never hardcode database passwords or API keys
3. **Add health checks** — Spring Actuator gives you `/health`, `/info`, `/metrics` for free
4. **Write integration tests** — use `@SpringBootTest` with `TestRestTemplate` or `MockMvc`
5. **Document your API** — SpringDoc/OpenAPI generates Swagger docs automatically

Spring Boot gives you a lot out of the box. The art is in keeping things simple, layered, and consistent. **Your future self will thank you.**
