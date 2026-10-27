# 21 — Architecture & Solution Layout

## 1. Overview

- **Purpose:** Define the solution layout, the dependency rule between projects, how a feature
  module is expressed in ASP.NET Core, and where each kind of type lives.
- **Applies to:** Every other document in this directory.
- **Shape:** Ports and adapters (hexagonal). Storage, mail and job dispatch sit behind interfaces so
  a backend can be swapped without altering core domain logic, as required by
  [12](../12-transformation-storage/spec.md) and [13](../13-avatar-upload/spec.md).

## 2. Solution layout

```
FileConverter.sln
src/
  FileConverter.Domain/          entities, value objects, enums, domain policies, domain exceptions
  FileConverter.Application/     use-case services, port interfaces, DTO-independent contracts
  FileConverter.Infrastructure/  EF Core, Supabase, S3, MailKit, Hangfire, Redis adapters
  FileConverter.Api/             controllers, request/response DTOs, validators, auth handlers, OpenAPI
  FileConverter.Cli/             admin bootstrap, e2e seed, db migrate verbs
tests/
  FileConverter.UnitTests/
  FileConverter.IntegrationTests/
```

**Dependency rule.** `Domain` references nothing. `Application` references `Domain`.
`Infrastructure` and `Api` reference `Application` (and transitively `Domain`). `Infrastructure`
and `Api` never reference each other — composition happens in `Api/Program.cs`, which is the only
place allowed to see both. `Cli` references `Application` + `Infrastructure`.

Enforced by an architecture test in `FileConverter.UnitTests` (see
[29](../29-testing-strategy/spec.md) §2.4), not by convention alone.

### 2.1 Feature folders

Inside each project, organise by feature, not by technical kind:

```
FileConverter.Application/
  Users/
    UserService.cs
    Ports/IUserRepository.cs
    Models/UserProfile.cs
  Rbac/
  Auth/
  Conversion/
  Transformations/
  Storage/
```

`FileConverter.Api/Users/` holds `UsersController.cs`, `Dtos/`, `Validators/`. A feature's files
sit together.

## 3. Appendix: NestJS concept mapping

For contributors arriving from the previous Node implementation. Nothing in this section is
required reading to work on the service.

| NestJS | ASP.NET Core |
| ---  | --- | 
| `@Module({ providers, imports })` | `IServiceCollection` extension: `services.AddUsersModule(configuration)` in a feature's `DependencyInjection.cs` |
| `@Injectable()` provider | Class registered in DI, usually `AddScoped` |
| `@Controller('users')` | `[ApiController] [Route("users")] class UsersController : ControllerBase` |
| `@UseGuards(AuthGuard)` | `[Authorize]` + authentication handler |
| `@UseGuards(RbacGuard)` + `@RequirePermission(...)` | `[RequirePermission("users","read")]` + dynamic policy provider — [02](../02-rbac/spec.md) §3 |
| `ValidationPipe` + class-validator DTO | FluentValidation validator + model binding — [23](../23-validation-and-partial-updates/spec.md) |
| Custom pipe | Model binder or action filter |
| Interceptor | Middleware, or `IAsyncActionFilter`/`IAsyncResultFilter` |
| Exception filter | `IExceptionHandler` — [24](../24-error-handling/spec.md) |
| `@nestjs/config` + Joi schema | Options pattern + `ValidateOnStart` — [26](../26-configuration-and-settings/spec.md) |
| `OnModuleInit` | `IHostedService` / `IStartupFilter` |
| `EventEmitter2` (RBAC change event) | `IRbacChangeNotifier` port, in-process + Redis adapters — [02](../02-rbac/spec.md) §4 |
| `@nestjs/schedule` cron | Hangfire recurring job — [25](../25-background-jobs/spec.md) |
| TypeORM repository | EF Core `DbContext` behind an `I...Repository` port — [22](../22-persistence-and-migrations/spec.md) |
| `npm run <script>` | `dotnet run --project src/FileConverter.Cli -- <verb>` |

## 4. Port and adapter naming

- Ports live in `Application/<Feature>/Ports/` and are named `I<Thing>Repository`,
  `I<Thing>Storage`, `I<Thing>Sender`, `I<Thing>Notifier`.
- Adapters live in `Infrastructure/<Feature>/` and are named after the technology:
  `EfUserRepository`, `SupabaseAvatarStorage`, `LocalAvatarStorage`, `S3ResultStorage`,
  `MailKitEmailSender`, `HangfireJobEnqueuer`, `RedisRbacChangeNotifier`.
- Each adapter has exactly one `AddXxx` registration, selected by configuration where a driver switch is required, as for avatar
  specs require a driver switch ([19](../19-test-infrastructure/spec.md) avatar driver, [12](../12-transformation-storage/spec.md) storage provider).

## 5. Application service shape

No mediator. A use case is a method on a feature service:

```csharp
public sealed class UserProfileService(
    IUserRepository users,
    IRbacCache rbac,
    IAuditLogger audit,
    TimeProvider clock)
{
    public Task<UserProfile> GetAsync(Guid callerId, Guid targetId, CancellationToken ct);
}
```

- Primary constructors, `sealed` by default, no base classes.
- `TimeProvider` is injected everywhere instead of `DateTimeOffset.UtcNow`, so TTL and expiry
  logic (OTP 10 min, access 15 min, retention 90 d) is testable without waiting.
- Services return domain models or throw domain exceptions. They never return `IActionResult`,
  never touch `HttpContext`, and never know about DTOs.
- Controllers are thin: bind, call one service method, map to a response DTO, return.

## 6. Decisions

| # | Decision | Rationale | Rejected |
| ---  | --- | --- | --- | 
| 21.1 | .NET 10 (LTS), `net10.0` | Current LTS; `Guid.CreateVersion7`, `HybridCache`, `IExceptionHandler`, built-in OpenAPI and request timeouts all available in-box | .NET 9 (STS, shorter support) |
| 21.2 | Controller-based MVC, not Minimal APIs | The authorization model is attribute-shaped: `[RequirePermission]` and `[AllowSelf]` compose naturally with MVC filters. Minimal API endpoint filters would work but read less clearly for this surface | Minimal APIs; FastEndpoints (extra dependency, unfamiliar) |
| 21.3 | Ports and adapters, 5 source projects | [12](../12-transformation-storage/spec.md) and [13](../13-avatar-upload/spec.md) demand swappable storage; project boundaries make the dependency rule compiler-enforced rather than aspirational | Single project with folders (no enforcement); Clean Architecture with separate Contracts project (more ceremony than this size needs) |
| 21.4 | No mediator library | MediatR >= 13 is commercially licensed, and nothing here needs pipeline behaviours that DI plus a filter cannot do | MediatR (licence); Wolverine (large surface for the benefit) |
| 21.5 | Hand-written mappers for user profiles, Mapster elsewhere if useful | The field-masking rules in specs 05/15 *are* business logic; hiding them in mapping config makes them untestable | AutoMapper (commercial licence) |
| 21.6 | `TimeProvider` injected, never `DateTimeOffset.UtcNow` | Every verification flow has a TTL; tests must control the clock | Static clock wrapper |
| 21.7 | One `AddXxxModule` extension per feature | Keeps `Program.cs` readable and makes the feature list explicit | Assembly scanning (implicit, harder to trace) |

## 7. Risks

- **Over-layering.** Five projects for a training-sized app is deliberate (it is what makes the
  structure legible) but it costs indirection. Do not add more layers.
- **Service bloat.** `UserService` will attract profile view, update, deletion, role assignment and
  password logic. Split by use case group early — `UserProfileService`, `UserAdminService`,
  `UserDeletionService` — rather than growing one class past a few hundred lines.
