---
layout: kurs
title: "Modul 1: Setup auf Clean Architecture Stack"
dozent: "HTL Leonding"
kurs: "Blazor Server Interactivity"
---

[← Zurück zur Kursübersicht](../blazor-server-interactivity)

# Modul 1: Setup auf Clean Architecture Stack

## Einleitung

In diesem Modul lernen Sie, wie Sie eine Blazor Server Anwendung auf einem bestehenden Clean Architecture Stack aufsetzen. Wir bauen auf einem vorhandenen Projekt auf, das bereits folgende Patterns implementiert hat:

- **Clean Architecture** mit klarer Trennung der Layer (Domain, Application, Infrastructure, Presentation)
- **CQRS Pattern** (Command Query Responsibility Segregation) mit MediatR
- **Repository Pattern** und **Unit of Work Pattern** für Datenzugriff

## Voraussetzungen

### Bestehender Clean Architecture Stack

Ihr Projekt sollte bereits folgende Struktur haben:

```
YourProject/
├── Domain/                          # Domain Layer
│   ├── Entities/                   # Domain-Entitäten
│   └── Interfaces/                 # Domain-Interfaces
├── Application/                     # Application Layer
│   ├── Features/                   # Feature-basierte Organisation
│   │   ├── Commands/               # Write-Operations (CQRS)
│   │   └── Queries/                # Read-Operations (CQRS)
│   └── Interfaces/                 # Application-Interfaces
├── Infrastructure/                  # Infrastructure Layer
│   ├── Persistence/                # Datenbank-Zugriff
│   │   ├── Repositories/           # Repository Implementations
│   │   └── UnitOfWork/            # Unit of Work Implementation
│   └── Services/                   # Infrastructure-Services
└── WebApi/                         # Bestehende API (optional)
```

### Wichtige NuGet-Pakete im Application Layer

```xml
<PackageReference Include="MediatR" Version="12.2.0" />
<PackageReference Include="FluentValidation" Version="11.9.0" />
```

### Wichtige NuGet-Pakete im Infrastructure Layer

```xml
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.0.0" />
<PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="8.0.0" />
<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.0.0" />
```

## Schritt 1: Blazor Server Projekt erstellen

Fügen Sie ein neues Blazor Server Projekt zu Ihrer Solution hinzu:

```powershell
dotnet new blazorserver -n YourProject.Blazor.Server -f net8.0
dotnet sln add YourProject.Blazor.Server
```

## Schritt 2: Projekt-Referenzen hinzufügen

Fügen Sie im Blazor Server Projekt Referenzen zu den anderen Layern hinzu:

```powershell
cd YourProject.Blazor.Server
dotnet add reference ../Application
dotnet add reference ../Infrastructure
```

**Wichtig:** Wir referenzieren nicht direkt das Domain-Projekt, da wir über den Application Layer darauf zugreifen.

## Schritt 3: NuGet-Pakete installieren

Installieren Sie die notwendigen Pakete für Blazor Server:

```powershell
# MediatR für CQRS
dotnet add package MediatR

# Entity Framework Core für DI
dotnet add package Microsoft.EntityFrameworkCore

# Optional: MudBlazor für UI-Komponenten
dotnet add package MudBlazor
```

## Schritt 4: Program.cs konfigurieren

Konfigurieren Sie die `Program.cs` um alle Services zu registrieren:

```csharp
using Microsoft.EntityFrameworkCore;
using YourProject.Infrastructure.Persistence;
using YourProject.Application;
using YourProject.Infrastructure;
using MudBlazor.Services;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddRazorComponents()
    .AddInteractiveServerComponents();

// MudBlazor (optional, aber empfohlen)
builder.Services.AddMudServices();

// Database Context
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection")));

// Application Layer Services (MediatR)
builder.Services.AddMediatR(cfg => 
    cfg.RegisterServicesFromAssembly(
        typeof(Application.AssemblyReference).Assembly));

// Infrastructure Layer Services (Repositories, UnitOfWork)
builder.Services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();

// Weitere Repositories falls vorhanden
builder.Services.AddScoped<ISensorRepository, SensorRepository>();
builder.Services.AddScoped<IMeasurementRepository, MeasurementRepository>();

var app = builder.Build();

// Configure the HTTP request pipeline.
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseAntiforgery();

app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode();

app.Run();
```

### Wichtige Aspekte der Konfiguration

1. **AddRazorComponents() mit AddInteractiveServerComponents()**
   - Aktiviert Blazor Server Interactivität
   - SignalR wird automatisch konfiguriert

2. **DbContext Registration**
   - Scoped Lifetime für Entity Framework Core
   - Jede SignalR-Verbindung erhält eigenen DbContext

3. **MediatR Registration**
   - Registriert alle Queries und Commands aus dem Application Layer
   - Ermöglicht CQRS-Pattern in Blazor-Komponenten

4. **Repository & UnitOfWork Registration**
   - Generic Repository Pattern für wiederverwendbare Datenzugriffs-Logik
   - Unit of Work für transaktionale Konsistenz

## Schritt 5: appsettings.json konfigurieren

Fügen Sie die ConnectionString hinzu:

```json
{
  "ConnectionStrings": {
        "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=YourProjectDb;Trusted_Connection=true;MultipleActiveResultSets=true"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.AspNetCore.SignalR": "Information"
    }
  },
  "AllowedHosts": "*"
}
```

**Hinweis:** SignalR-Logging auf "Information" für besseres Debugging.

## Schritt 6: App.razor anpassen

Passen Sie die `App.razor` für Blazor Server an:

```razor
<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <base href="/" />
    <link rel="stylesheet" href="bootstrap/bootstrap.min.css" />
    <link rel="stylesheet" href="app.css" />
    <link rel="stylesheet" href="YourProject.Blazor.Server.styles.css" />
    
    @* Optional: MudBlazor *@
    <link href="https://fonts.googleapis.com/css?family=Roboto:300,400,500,700&display=swap" rel="stylesheet" />
    <link href="_content/MudBlazor/MudBlazor.min.css" rel="stylesheet" />
    
    <HeadOutlet />
</head>
<body>
    <Routes />
    <script src="_framework/blazor.web.js"></script>
    
    @* Optional: MudBlazor *@
    <script src="_content/MudBlazor/MudBlazor.min.js"></script>
</body>
</html>
```

## Schritt 7: Erste Komponente mit CQRS erstellen

Erstellen Sie eine Beispiel-Komponente unter `Components/Pages/Sensors.razor`:

```razor
@page "/sensors"
@using MediatR
@using YourProject.Application.Features.Sensors.Queries
@inject IMediator Mediator

<PageTitle>Sensoren</PageTitle>

<h3>Sensoren</h3>

@if (sensors == null)
{
    <p><em>Lade Daten...</em></p>
}
else if (!sensors.Any())
{
    <p>Keine Sensoren gefunden.</p>
}
else
{
    <table class="table">
        <thead>
            <tr>
                <th>ID</th>
                <th>Name</th>
                <th>Standort</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var sensor in sensors)
            {
                <tr>
                    <td>@sensor.Id</td>
                    <td>@sensor.Name</td>
                    <td>@sensor.Location</td>
                </tr>
            }
        </tbody>
    </table>
}

@code {
    private IEnumerable<SensorDto>? sensors;

    protected override async Task OnInitializedAsync()
    {
        // CQRS Query über MediatR
        var query = new GetAllSensorsQuery();
        sensors = await Mediator.Send(query);
    }
}
```

### Code-Behind Alternative (empfohlen für größere Komponenten)

Erstellen Sie `Sensors.razor.cs`:

```csharp
using Microsoft.AspNetCore.Components;
using MediatR;
using YourProject.Application.Features.Sensors.Queries;

namespace Blazor.Server.Components.Pages;

public partial class Sensors
{
    [Inject]
    private IMediator Mediator { get; set; } = default!;

    private IEnumerable<SensorDto>? sensors;

    protected override async Task OnInitializedAsync()
    {
        var query = new GetAllSensorsQuery();
        sensors = await Mediator.Send(query);
    }
}
```

Und vereinfachtes `Sensors.razor`:

```razor
@page "/sensors"

<PageTitle>Sensoren</PageTitle>

<h3>Sensoren</h3>

@if (sensors == null)
{
    <p><em>Lade Daten...</em></p>
}
else if (!sensors.Any())
{
    <p>Keine Sensoren gefunden.</p>
}
else
{
    <table class="table">
        <thead>
            <tr>
                <th>ID</th>
                <th>Name</th>
                <th>Standort</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var sensor in sensors)
            {
                <tr>
                    <td>@sensor.Id</td>
                    <td>@sensor.Name</td>
                    <td>@sensor.Location</td>
                </tr>
            }
        </tbody>
    </table>
}
```

## Schritt 8: Navigation hinzufügen

Fügen Sie in `Components/Layout/NavMenu.razor` einen Link zu Ihrer neuen Seite hinzu:

```razor
<div class="nav-item px-3">
    <NavLink class="nav-link" href="sensors">
        <span class="bi bi-speedometer2" aria-hidden="true"></span> Sensoren
    </NavLink>
</div>
```

## Best Practices für die Integration

### 1. Dependency Injection richtig nutzen

```csharp
// ✅ Richtig: Constructor Injection in Code-Behind
public partial class Sensors
{
    [Inject]
    private IMediator Mediator { get; set; } = default!;
}

// ❌ Falsch: Direkte Instanziierung
// var mediator = new Mediator(); // NIEMALS!
```

### 2. Async/Await konsequent verwenden

```csharp
// ✅ Richtig: Async Lifecycle Methods
protected override async Task OnInitializedAsync()
{
    sensors = await Mediator.Send(query);
}

// ❌ Falsch: Synchroner Code blockiert SignalR
protected override void OnInitialized()
{
    sensors = Mediator.Send(query).Result; // Deadlock-Gefahr!
}
```

### 3. DTOs verwenden statt Domain Entities

```csharp
// ✅ Richtig: DTOs in der Presentation Layer
private IEnumerable<SensorDto>? sensors;

// ❌ Falsch: Domain Entities direkt im UI
// private IEnumerable<Sensor>? sensors; // Kopplung zu hoch!
```

### 4. Error Handling implementieren

```csharp
protected override async Task OnInitializedAsync()
{
    try
    {
        var query = new GetAllSensorsQuery();
        sensors = await Mediator.Send(query);
    }
    catch (Exception ex)
    {
        // Logging über ILogger<T>
        Logger.LogError(ex, "Fehler beim Laden der Sensoren");
        // User-Feedback anzeigen
        errorMessage = "Daten konnten nicht geladen werden.";
    }
}
```

## Zusammenfassung

Sie haben nun erfolgreich:

✅ Ein Blazor Server Projekt auf Ihrem Clean Architecture Stack aufgesetzt
✅ MediatR für CQRS-Pattern integriert
✅ Repository Pattern und Unit of Work in Blazor verfügbar gemacht
✅ Eine erste Komponente mit Datenbankzugriff erstellt
✅ Best Practices für die Integration gelernt

Im nächsten Modul werden wir uns detailliert mit den **Lifecycle Events** von Blazor Server Komponenten und dem **Prerendering-Konzept** beschäftigen.

---

[Weiter zu Modul 2: Page Lifecycle Events & Prerendering →](../02-lifecycle-events)
