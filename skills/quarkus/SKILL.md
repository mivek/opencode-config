---
name: quarkus
description: Conventions and patterns for Quarkus + Maven Java projects. Load when working on Quarkus codebases — REST endpoints, CDI, configuration, native builds, testing.
compatibility: opencode
metadata:
  stack: java-quarkus
  build-tool: maven
---

# Quarkus + Maven conventions

## Project structure expected

```
src/
  main/
    java/<package>/
      <Module>Resource.java        # JAX-RS endpoints
      <Module>Service.java          # @ApplicationScoped business logic
      <Module>Repository.java       # @ApplicationScoped data access
    resources/
      application.properties        # config (or .yml)
      META-INF/resources/           # static files
  test/
    java/<package>/
      <Module>ResourceTest.java     # @QuarkusTest with RestAssured
```

## Always run the wrapper

Use `./mvnw` (Maven wrapper) not `mvn` — it guarantees the right Maven version. If wrapper missing, suggest adding it.

## Standard commands

| Task | Command |
|---|---|
| Dev mode (hot reload) | `./mvnw quarkus:dev` |
| Compile | `./mvnw compile` |
| Run tests | `./mvnw test` |
| Run a specific test | `./mvnw test -Dtest=ClassName#methodName` |
| Package JAR | `./mvnw package` |
| Native build | `./mvnw package -Pnative` |
| Add an extension | `./mvnw quarkus:add-extension -Dextensions="name"` |
| Show dep tree | `./mvnw dependency:tree` |

## Conventions

### Dependency injection
- Use `@ApplicationScoped` for services/repositories (NOT `@Singleton` unless eager init needed)
- Use `@Inject` (constructor injection preferred over field injection)
- Avoid `@RequestScoped` unless you genuinely need per-request state

### REST endpoints
- Use `jakarta.ws.rs.*` annotations (NOT `javax.ws.rs.*` — that's deprecated since Quarkus 3)
- Return DTOs/records, not entities directly
- Use `@PathParam`, `@QueryParam`, `@HeaderParam` explicitly

### Configuration
- Externalize config via `application.properties` with `quarkus.*` namespace for framework, `app.*` for your custom keys
- Inject with `@ConfigProperty(name="...")` or use `@ConfigMapping` for grouped config
- Never hardcode environment-specific values

### Persistence (Panache)
- Use Panache patterns: extend `PanacheEntity` or `PanacheRepository<T>`
- Active record (PanacheEntity) for simple cases, repository pattern for complex queries
- Always use parameterized queries: `find("name = ?1", name)`, never string concatenation

### Testing
- `@QuarkusTest` for integration-level tests (boots app)
- RestAssured for HTTP tests: `given().when().get("/api/x").then().statusCode(200)`
- Use `@TestProfile` to swap config per test class
- Mock external services with `@InjectMock` (requires `quarkus-junit5-mockito`)

## Common pitfalls

- **Native image fails** : usually reflection access to a class without `@RegisterForReflection`. Add it or use `quarkus.native.additional-build-args=--initialize-at-build-time=...`
- **Dev mode doesn't reload** : check for compilation errors in terminal — silent failures mean stale state
- **Slow startup in dev** : check `quarkus.live-reload.password` and verify you're not loading huge datasets at startup
- **Tests fail with "container already running"** : `./mvnw clean test` to reset
- **Wrong jakarta import** : Quarkus 3+ requires `jakarta.*`, never `javax.*`

## When generating new code

1. Check existing extensions in `pom.xml` first — don't add what's already there
2. Match the package structure used by existing modules
3. Follow the existing pattern for the same type of file (look at neighboring `*Resource.java` files before writing a new one)
4. Add tests in the matching `src/test/java/` location
5. Mention any extension that needs to be added
