# Project Guidelines

## Architecture

- This repository is a .NET 8 Aspire solution. Use `AdventureWorksApp.AppHost/AppHost.cs` as the local entry point when you need the full distributed app.
- `AdventureWorksApp.AppHost` orchestrates `AdventureWorksApp.ApiService` as `apiservice` and `AdventureWorksApp.Web` as `webfrontend`. Preserve those service names unless a task explicitly requires coordinated renames.
- `AdventureWorksApp.ServiceDefaults/Extensions.cs` is the shared place for service discovery, resilience, health checks, and OpenTelemetry defaults. Put cross-service startup behavior there instead of duplicating it in each app.

## Build And Test

- Restore and build from the solution root with `dotnet restore AdventureWorksApp.sln` and `dotnet build AdventureWorksApp.sln`.
- Run the full stack locally with `dotnet run --project AdventureWorksApp.AppHost`.
- Run tests with `dotnet test AdventureWorksApp.Tests/AdventureWorksApp.Tests.csproj`.
- Prefer validating end-to-end behavior through the AppHost or the Aspire test project instead of launching the web or API projects in isolation.

## Conventions

- Keep the current `net8.0` target and top-level `Program.cs` hosting style unless the task requires a coordinated framework or hosting-model change.
- In the web project, keep API access wired through the typed `HttpClient` in `AdventureWorksApp.Web/Program.cs` and service discovery address `https+http://apiservice`; do not replace it with hard-coded localhost URLs.
- Preserve the health-check pattern based on `MapDefaultEndpoints()` in the service projects and `WithHttpHealthCheck("/health")` in the AppHost when changing startup or orchestration code.
- Tests are integration-style MSTest tests built on `Aspire.Hosting.Testing`. Add new coverage there when verifying distributed app behavior.
- Do not edit generated artifacts under `bin/`, `obj/`, `.vs/`, or NuGet-generated files unless the task is explicitly about build outputs.

## Documentation

- `README.md` currently contains only the repository title, so treat the current source as the primary reference for behavior.
- Use `AdventureWorksApp.AppHost/AppHost.cs`, `AdventureWorksApp.ServiceDefaults/Extensions.cs`, and `AdventureWorksApp.Tests/WebTests.cs` as pattern examples before introducing new structure.