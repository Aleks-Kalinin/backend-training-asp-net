# 30 — Build, CI & Docker

> Build properties, CLI verbs, container images and the CI pipeline.

## 1. Repository layout

```
Directory.Build.props        shared MSBuild properties for every project
Directory.Packages.props     central package version management
.editorconfig                formatting + analyzer severities
FileConverter.sln
src/ ...                     see 21-architecture-solution-layout
tests/ ...
docker/
  Dockerfile
  docker-compose.yml
  docker-compose.override.yml
openapi/v1.json              committed contract snapshot (see 28)
.github/workflows/ci.yml
```

`Directory.Build.props` sets, for all projects:

```xml
<TargetFramework>net10.0</TargetFramework>
<Nullable>enable</Nullable>
<ImplicitUsings>enable</ImplicitUsings>
<TreatWarningsAsErrors>true</TreatWarningsAsErrors>
<EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
<AnalysisLevel>latest-recommended</AnalysisLevel>
```

Central Package Management (`ManagePackageVersionsCentrally=true`) means a package version appears
exactly once in the repository. Renovate or Dependabot updates that one file.

## 2. Command mapping

Common tasks, with their npm equivalents for contributors arriving from the previous
implementation:

| Node | .NET |
| ---  | --- | 
| `npm run start:dev` | `dotnet watch --project src/FileConverter.Api` |
| `npm run build` | `dotnet build -c Release` |
| `npm test` | `dotnet test` |
| `npm run lint` / `format` | `dotnet format --verify-no-changes` |
| `npm run migration:run` | `dotnet run --project src/FileConverter.Cli -- db migrate` |
| `npm run migration:generate` | `dotnet ef migrations add <Name> -p src/FileConverter.Infrastructure -s src/FileConverter.Api` |
| `npm run admin:bootstrap -- --email x` | `dotnet run --project src/FileConverter.Cli -- admin bootstrap --email x` |
| `npm run e2e:seed` | `dotnet run --project src/FileConverter.Cli -- e2e seed` |
| `npm run start:e2e` | `dotnet run --project src/FileConverter.Api -c Release` with `DOTNET_ENVIRONMENT=E2E` |

`FileConverter.Cli` uses `System.CommandLine` and shares the application's DI container, so a verb
is a thin wrapper over the same services the API uses — no duplicated logic, no raw SQL scripts.

## 3. Docker

Multi-stage build, `mcr.microsoft.com/dotnet/sdk:10.0` to build and
`mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled` to run.

- Runs as a non-root user; chiseled image has no shell, which is the point.
- `dotnet restore` as its own layer keyed on the `.csproj` and `Directory.Packages.props` files, so
  dependency restore caches across source changes.
- `ASPNETCORE_HTTP_PORTS=8080`, no HTTPS inside the container — TLS terminates at the ingress.
- Health check hitting `/health/live`.
- **SkiaSharp needs native dependencies.** The chiseled image lacks them; add
  `SkiaSharp.NativeAssets.Linux.NoDependencies` to the API project, or fall back to the
  non-chiseled `aspnet:10.0-noble` image. Verify this in CI with a smoke test that converts an
  SVG inside the built image — it is the most likely thing to break only in the container.

### 3.1 Compose

| Service | Image | Profile | Notes |
| ---  | --- | --- | --- | 
| `api` | built locally | default | depends on `postgres` |
| `postgres` | `postgres:17` | default | named volume, `citext` created by migration |
| `mailpit` | `axllent/mailpit` | `test` | SMTP 1025, UI/API 8025 — [19](../19-test-infrastructure/spec.md) |
| `redis` | `redis:7` | `multi-instance` | only needed for the distributed cache/notifier |
| `minio` | `minio/minio` | `storage` | S3-compatible target for result storage |

`docker compose --profile test up` reproduces [19](../19-test-infrastructure/spec.md).

## 4. CI pipeline

GitHub Actions, on push and pull request:

1. `dotnet restore --locked-mode` (lock files committed).
2. `dotnet format --verify-no-changes` — formatting is a build failure, not a review comment.
3. `dotnet build -c Release` with warnings as errors.
4. `dotnet test` with coverage collection; Testcontainers uses the runner's Docker daemon.
5. `dotnet ef migrations has-pending-model-changes` — fails if the model and migrations drift.
6. OpenAPI snapshot diff — fails if `openapi/v1.json` is stale ([28](../28-openapi-and-versioning/spec.md) §5).
7. `docker build` plus the SkiaSharp smoke test from §3.
8. Vulnerability scan: `dotnet list package --vulnerable --include-transitive`, failing on
   moderate or higher.

Branch protection requires all of the above. Nothing deploys from this pipeline — deployment is out
of scope for these specs.

## 5. Local developer hooks

Husky.Net pre-commit: `dotnet format` on staged files only, plus a `gitleaks` scan. Keep it to
seconds — anything slower gets bypassed, and a bypassed hook is worse than none.

## 6. Decisions

| # | Decision | Rationale | Rejected |
| ---  | --- | --- | --- | 
| 30.1 | Central Package Management | One version per package across seven projects; upgrades are a one-line diff | Per-project `PackageReference` versions (drift) |
| 30.2 | `TreatWarningsAsErrors` from day one | Retrofitting it onto a warning-rich codebase never happens | Warnings as warnings |
| 30.3 | Chiseled runtime image | No shell, no package manager, small attack surface | `aspnet:10.0` full (larger); Alpine (musl friction with SkiaSharp) |
| 30.4 | `System.CommandLine` CLI sharing the app's DI | [17](../17-user-role-assignment/spec.md)'s bootstrap and [19](../19-test-infrastructure/spec.md)'s seed must use the same hashing, role lookup and guards as the API | Standalone SQL scripts (would duplicate password hashing and the last-admin rule) |
| 30.5 | Committed lock files + `--locked-mode` | Reproducible restores; a transitive bump cannot land silently | Floating restore |
| 30.6 | OpenAPI snapshot in CI | The contract is reviewed like code; this is the control for the whole class of defect in [18](../18-api-contract/spec.md) | Trusting runtime generation |
| 30.7 | Compose profiles rather than separate files | [19](../19-test-infrastructure/spec.md) already specifies a `test` profile for Mailpit; extending the pattern keeps one file | `docker-compose.test.yml` fan-out |

## 7. Risks

- **SkiaSharp native assets.** The single most likely CI/production divergence. The §3 smoke test
  is not optional. If it proves fragile, switch the base image before switching the library.
- **Testcontainers in CI.** Requires a Docker daemon; on constrained runners, fall back to service
  containers (see [29](../29-testing-strategy/spec.md) §5).
- **Strict formatting + warnings-as-errors** can block unrelated work when an analyzer version
  changes. Pin analyzer versions through Central Package Management, like everything else.
