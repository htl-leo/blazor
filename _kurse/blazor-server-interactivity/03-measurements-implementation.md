---
layout: kurs
title: "Modul 3: IoT Measurements - Drei Implementierungsansätze"
dozent: "HTL Leonding"
kurs: "Blazor Server Interactivity"
---

[← Zurück zur Kursübersicht](../blazor-server-interactivity)

# Modul 3: IoT Measurements - Drei Implementierungsansätze

## Einleitung

In diesem Modul lernen Sie drei verschiedene Ansätze kennen, um große Datenmengen in Blazor Server effizient darzustellen. Wir bauen auf dem **IoT-Szenario** auf und zeigen praktisch, wie Sie von einer einfachen Implementierung zu einer hochperformanten Lösung kommen.

**Szenario:** Eine IoT-Anwendung mit Sensoren, die kontinuierlich Messwerte erfassen. Wir haben mehrere tausend Messwerte in der Datenbank und möchten diese dem Benutzer anzeigen.

## Voraussetzungen

Sie haben bereits in Modul 1 den Clean Architecture Stack mit folgenden Komponenten aufgesetzt:

- ✅ Entity Framework Core mit SQL Server LocalDB
- ✅ MediatR für CQRS-Queries
- ✅ Repository Pattern
- ✅ Unit of Work Pattern

Ihr Projekt sollte diese Entities haben:

```csharp
// Domain/Entities/Sensor.cs
public class Sensor
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Location { get; set; } = string.Empty;
    public ICollection<Measurement> Measurements { get; set; } = [];
}

// Domain/Entities/Measurement.cs
public class Measurement
{
    public int Id { get; set; }
    public int SensorId { get; set; }
    public Sensor Sensor { get; set; } = default!;
    public double Value { get; set; }
    public DateTime Timestamp { get; set; }
}
```

---

## Schritt 1: Einfache HTML-Tabelle (Measurements Simple)

### Das Problem

Bei großen Datenmengen führt das synchrone Laden aller Datensätze zu langen Wartezeiten. Der Benutzer sieht zunächst nur "Loading..." bis alle Daten geladen und gerendert sind.

**Typisches Szenario:**
- 10.000 Messwerte in der Datenbank
- Alle werden auf einmal geladen
- Benutzer wartet 5-10 Sekunden
- Dann erscheint eine riesige Tabelle auf einmal

### Implementierung

#### 1. Query und Handler erstellen

Erstellen Sie `Application/Features/Measurements/Queries/GetAllMeasurementsSimpleQuery.cs`:

```csharp
using Application.Features.Dtos;
using MediatR;
using Application.Common.Results;

namespace Application.Features.Measurements.Queries;

public sealed record GetAllMeasurementsSimpleQuery 
    : IRequest<Result<IReadOnlyCollection<GetMeasurementWithSensorDto>>>;
```

Erstellen Sie den Handler `GetAllMeasurementsSimpleQueryHandler.cs`:

```csharp
using Application.Features.Dtos;
using Application.Interfaces;
using MediatR;
using Application.Common.Results;

namespace Application.Features.Measurements.Queries;

public sealed class GetAllMeasurementsSimpleQueryHandler(IUnitOfWork uow) 
    : IRequestHandler<GetAllMeasurementsSimpleQuery, 
                      Result<IReadOnlyCollection<GetMeasurementWithSensorDto>>>
{
    public async Task<Result<IReadOnlyCollection<GetMeasurementWithSensorDto>>> Handle(
        GetAllMeasurementsSimpleQuery request, 
        CancellationToken cancellationToken)
    {
        // Lädt ALLE Messwerte mit Sensor-Informationen
        var entities = await uow.Measurements.GetAllWithSensorAsync(cancellationToken);
        
        // Mapping zu DTOs
        var dtos = entities
            .Select(m => new GetMeasurementWithSensorDto(
                SensorName: m.Sensor.Name,
                Location: m.Sensor.Location,
                Date: m.Timestamp.ToString("dd.MM.yyyy"),
                Time: m.Timestamp.ToString("HH:mm:ss"),
                Value: m.Value.ToString("F2")
            ))
            .ToArray();

        return Result<IReadOnlyCollection<GetMeasurementWithSensorDto>>.Success(dtos);
    }
}
```

#### 2. DTO definieren

Erstellen Sie `Application/Features/Dtos/GetMeasurementWithSensorDto.cs`:

```csharp
namespace Application.Features.Dtos;

public sealed record GetMeasurementWithSensorDto(
    string SensorName,
    string Location,
    string Date,
    string Time,
    string Value
);
```

#### 3. Repository-Methode hinzufügen

In `Infrastructure/Persistence/Repositories/MeasurementRepository.cs`:

```csharp
public async Task<IReadOnlyCollection<Measurement>> GetAllWithSensorAsync(
    CancellationToken cancellationToken = default)
{
    return await _context.Measurements
        .Include(m => m.Sensor)
        .OrderByDescending(m => m.Timestamp)
        .ToListAsync(cancellationToken);
}
```

#### 4. Razor Component erstellen

Erstellen Sie `Components/Pages/MeasurementsSimple.razor`:

```razor
@page "/measurements-simple"
@using Application.Features.Dtos

<PageTitle>Measurements Simple</PageTitle>

<h3>Measurements - Simple Approach</h3>

@if (_error is not null)
{
    <div class="alert alert-danger">@_error</div>
}
else if (_items is null)
{
    <div class="alert alert-info">Loading...</div>
}
else
{
    <p>Anzahl: @_items.Count Messwerte</p>
    
    <table class="table table-striped table-hover">
        <thead>
            <tr>
                <th>Sensor</th>
                <th>Location</th>
                <th>Date</th>
                <th>Time</th>
                <th style="text-align:right">Value</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var item in _items)
            {
                <tr>
                    <td>@item.SensorName</td>
                    <td>@item.Location</td>
                    <td>@item.Date</td>
                    <td>@item.Time</td>
                    <td style="text-align:right">@item.Value</td>
                </tr>
            }
        </tbody>
    </table>
}
```

#### 5. Code-Behind erstellen

Erstellen Sie `Components/Pages/MeasurementsSimple.razor.cs`:

```csharp
using Application.Features.Dtos;
using Application.Features.Measurements.Queries;
using MediatR;
using Microsoft.AspNetCore.Components;

namespace Blazor.Server.Components.Pages;

public partial class MeasurementsSimple : ComponentBase
{
    [Inject] 
    private IMediator Mediator { get; set; } = default!;

    private IReadOnlyCollection<GetMeasurementWithSensorDto>? _items;
    private string? _error;

    protected override async Task OnInitializedAsync()
    {
        try
        {
            // Blockiert, bis ALLE Daten geladen sind
            var result = await Mediator.Send(new GetAllMeasurementsSimpleQuery());
            
            if (result.IsSuccess && result.Value is not null)
            {
                _items = result.Value;
            }
            else
            {
                _error = result.Message ?? "Unknown error";
            }
        }
        catch (Exception ex)
        {
            _error = ex.Message;
        }
    }
}
```

### Test der Implementierung

1. Starten Sie die Anwendung: `dotnet run`
2. Navigieren Sie zu `/measurements-simple`
3. Beobachten Sie:
   - Lange "Loading..." Anzeige
   - Dann erscheint plötzlich die komplette Tabelle
   - Bei vielen Daten kann der Browser kurz "hängen"

### Analyse der Nachteile

❌ **Lange Ladezeit**: Benutzer sieht lange "Loading..." (5-10 Sekunden bei 10.000 Einträgen)  
❌ **Hoher Speicherverbrauch**: Alle Daten werden im Server- und Client-Speicher gehalten  
❌ **Langsames Rendering**: Browser muss große HTML-Tabelle auf einmal rendern  
❌ **Schlechte User Experience**: Keine Rückmeldung während des Ladens  
❌ **SignalR-Bottleneck**: Ein großes Payload wird über SignalR übertragen

**Console-Ausgabe (Server):**
```
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (1,234ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT [m].[Id], [m].[SensorId], [m].[Value], [m].[Timestamp], [s].[Name], [s].[Location]
      FROM [Measurements] AS [m]
      INNER JOIN [Sensors] AS [s] ON [m].[SensorId] = [s].[Id]
      ORDER BY [m].[Timestamp] DESC
```

**Performance-Zahlen:**
- Datenbankabfrage: ~1-2 Sekunden
- DTO-Mapping: ~0.5 Sekunden
- SignalR-Transfer: ~2-3 Sekunden
- Browser-Rendering: ~1-2 Sekunden
- **Gesamt: 5-8 Sekunden**

---

## Schritt 2: Streaming für schnellere Latenz (Measurements Stream)

### Die Lösung

Statt alle Daten auf einmal zu laden, verwenden wir **IAsyncEnumerable** für ein schrittweises Laden und Rendern.

Wie es technisch funktioniert:
- Backend liefert einen asynchronen Datenstrom: `IAsyncEnumerable<T>` (hier per MediatR `CreateStream(...)`).
- Die Komponente iteriert mit `await foreach` und fügt die Elemente sukzessive in eine Liste ein.
- Sichtbare UI-Updates passieren erst, wenn `StateHasChanged()` aufgerufen wird – deshalb bündeln wir Updates (z. B. alle 50 Elemente), um SignalR‑Traffic zu reduzieren.
- EF Core liest die Daten sequenziell (über `AsAsyncEnumerable()`), wodurch der Peak‑Speicher sinkt – aber die Gesamtmenge bleibt gleich, wenn keine Filterung aktiv ist.

Wichtige Eigenschaften des Streamings:
- Geringere Time‑to‑First‑Row: Erste Ergebnisse erscheinen nach wenigen 100 ms.
- Fortschrittsgefühl: Nutzer kann bereits lesen/scrollen, während weitere Daten eintreffen.
- Steuerbar: Über Batch‑Größe (z. B. 50/100) lässt sich die Renderfrequenz feinjustieren.

Grenzen und Trade‑offs:
- Es werden trotzdem alle Datensätze übertragen, sofern nicht gefiltert/gepaged wird → Server‑ und Netzwerk‑Last bleiben hoch.
- Mehr DOM‑Diffs durch häufigere Re‑Renders (wir entschärfen das durch Batch‑Updates).
- Streaming beendet sich bei Verbindungsabbruch; beim erneuten Verbinden muss der Stream neu gestartet werden.

Best Practices:
- UI‑Updates in Batches (50–200): `StateHasChanged()` nicht für jedes Element.
- `CancellationToken` sauber unterstützen (Komponente kann Stream abbrechen).
- `AsNoTracking()` und DTO‑Projektion verwenden, um Overhead zu minimieren.
- Für 40k+ Datensätze: Streaming nur für „frühe Sichtbarkeit“ nutzen; für effizientes Browsen weiterhin Server‑seitiges Paging vorziehen.

**Prinzip:**
```
Traditionell:  [Lade alles] ----5s----> [Zeige alles]
Streaming:     [Zeige ersten Batch] -> [Zeige mehr] -> [Zeige mehr] -> ...
               ↑ 0.5s                  ↑ +0.3s        ↑ +0.3s
```

### Implementierung

#### 1. Streaming Query erstellen

Erstellen Sie `GetAllMeasurementsStreamQuery.cs`:

```csharp
using Application.Features.Dtos;
using MediatR;

namespace Application.Features.Measurements.Queries;

public sealed record GetAllMeasurementsStreamQuery 
    : IStreamRequest<GetMeasurementWithSensorDto>;
```

#### 2. Stream Handler implementieren

Erstellen Sie `GetAllMeasurementsStreamQueryHandler.cs`:

```csharp
using System.Runtime.CompilerServices;
using Application.Features.Dtos;
using Application.Interfaces;
using MediatR;

namespace Application.Features.Measurements.Queries;

public sealed class GetAllMeasurementsStreamQueryHandler(IUnitOfWork uow) 
    : IStreamRequestHandler<GetAllMeasurementsStreamQuery, GetMeasurementWithSensorDto>
{
    public async IAsyncEnumerable<GetMeasurementWithSensorDto> Handle(
        GetAllMeasurementsStreamQuery request, 
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        // Streamt Messwerte einzeln (oder in kleinen Batches)
        await foreach (var m in uow.Measurements.StreamAllWithSensorAsync(cancellationToken))
        {
            yield return new GetMeasurementWithSensorDto(
                SensorName: m.Sensor.Name,
                Location: m.Sensor.Location,
                Date: m.Timestamp.ToString("dd.MM.yyyy"),
                Time: m.Timestamp.ToString("HH:mm:ss"),
                Value: m.Value.ToString("F2")
            );
        }
    }
}
```

#### 3. Repository Streaming-Methode

In `MeasurementRepository.cs`:

```csharp
public async IAsyncEnumerable<Measurement> StreamAllWithSensorAsync(
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    var measurements = _context.Measurements
        .Include(m => m.Sensor)
        .OrderByDescending(m => m.Timestamp)
        .AsAsyncEnumerable();

    await foreach (var measurement in measurements.WithCancellation(cancellationToken))
    {
        yield return measurement;
    }
}
```

#### 4. Razor Component mit Streaming

Erstellen Sie `Components/Pages/MeasurementsStream.razor`:

```razor
@page "/measurements-stream"
@using Application.Features.Dtos

<PageTitle>Measurements Stream</PageTitle>

<h3>Measurements - Streaming Approach</h3>

@if (_error is not null)
{
    <div class="alert alert-danger">@_error</div>
}
else if (_isLoading)
{
    <div class="alert alert-info">Loading first measurements...</div>
}

@if (_items.Count > 0)
{
    <p>Anzahl: @_items.Count Messwerte @(_isStreamingComplete ? "" : "(wird geladen...)")</p>
    
    <table class="table table-striped table-hover">
        <thead>
            <tr>
                <th>Sensor</th>
                <th>Location</th>
                <th>Date</th>
                <th>Time</th>
                <th style="text-align:right">Value</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var item in _items)
            {
                <tr>
                    <td>@item.SensorName</td>
                    <td>@item.Location</td>
                    <td>@item.Date</td>
                    <td>@item.Time</td>
                    <td style="text-align:right">@item.Value</td>
                </tr>
            }
        </tbody>
    </table>
}
```

#### 5. Code-Behind mit Streaming-Logik

Erstellen Sie `Components/Pages/MeasurementsStream.razor.cs`:

```csharp
using Application.Features.Dtos;
using Application.Features.Measurements.Queries;
using MediatR;
using Microsoft.AspNetCore.Components;

namespace Blazor.Server.Components.Pages;

public partial class MeasurementsStream : ComponentBase, IDisposable
{
    [Inject] 
    private IMediator Mediator { get; set; } = default!;

    private readonly List<GetMeasurementWithSensorDto> _items = [];
    private bool _isLoading = true;
    private bool _isStreamingComplete = false;
    private string? _error;
    private readonly CancellationTokenSource _cts = new();
    private bool _disposed;

    protected override Task OnInitializedAsync()
    {
        // Startet Streaming im Hintergrund (Fire-and-Forget)
        _ = LoadStreamAsync(_cts.Token);
        return Task.CompletedTask;
    }

    private async Task LoadStreamAsync(CancellationToken ct)
    {
        try
        {
            int batchCounter = 0;
            
            // Verwendet MediatR.CreateStream für IAsyncEnumerable
            await foreach (var dto in Mediator.CreateStream(
                new GetAllMeasurementsStreamQuery(), ct))
            {
                _items.Add(dto);
                
                // Nach dem ersten Batch: "Loading..." ausblenden
                if (_isLoading && _items.Count > 0)
                {
                    _isLoading = false;
                }

                // UI-Update alle 50 Datensätze (Performance-Optimierung)
                batchCounter++;
                if (batchCounter % 50 == 0)
                {
                    await InvokeAsync(StateHasChanged);
                    Console.WriteLine($"[Stream] Received {_items.Count} measurements...");
                }
            }
            
            // Finales UI-Update
            _isStreamingComplete = true;
            await InvokeAsync(StateHasChanged);
            Console.WriteLine($"[Stream] Complete! Total: {_items.Count} measurements");
        }
        catch (OperationCanceledException)
        {
            // Normal beim Verlassen der Seite
            Console.WriteLine("[Stream] Cancelled by user");
        }
        catch (Exception ex)
        {
            _error = ex.Message;
            await InvokeAsync(StateHasChanged);
            Console.WriteLine($"[Stream] Error: {ex.Message}");
        }
    }

    // Cleanup bei Component-Disposal
    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        
        if (disposing)
        {
            _cts.Cancel();
            _cts.Dispose();
        }
        
        _disposed = true;
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
}
```

### Test der Streaming-Implementierung

1. Navigieren Sie zu `/measurements-stream`
2. Beobachten Sie:
   - Nach ~0.5s erscheinen die ersten Zeilen
   - Die Tabelle wird kontinuierlich erweitert
   - Alle 50 Einträge gibt es ein Update
   - Benutzer kann sofort scrollen und lesen

**Console-Ausgabe:**
```
[Stream] Received 50 measurements...
[Stream] Received 100 measurements...
[Stream] Received 150 measurements...
...
[Stream] Complete! Total: 10000 measurements
```

### Vorteile des Streamings

✅ **Sofortiges Feedback**: Erste Daten erscheinen fast sofort (~0.5s statt 5s)  
✅ **Progressive Rendering**: Tabelle wird schrittweise gefüllt  
✅ **Bessere Perceived Performance**: Benutzer sieht sofort Ergebnisse  
✅ **Batch-Updates**: `StateHasChanged()` nur alle 50 Einträge → weniger SignalR-Traffic  
✅ **Abbrechbar**: Benutzer kann während des Ladens wegnavigieren (CancellationToken)

### Nachteile bleiben bestehen

❌ **Immer noch viele Daten**: Alle Datensätze werden letztendlich geladen  
❌ **Keine Filterung**: Benutzer sieht alle Messwerte  
❌ **Keine Pagination**: Schwer navigierbar bei tausenden Zeilen  
❌ **Memory**: Irgendwann sind alle Daten im Speicher

**Performance-Zahlen:**
- Erste Anzeige: ~0.5 Sekunden ✅
- Streaming-Dauer gesamt: ~3-4 Sekunden
- SignalR-Messages: ~200 (statt 1 große) ✅
- User Experience: **Deutlich besser!** ✅

---

## Schritt 3: MudBlazor DataGrid mit Server-seitigem Paging

### Die ultimative Lösung

Die beste Performance erreichen wir durch:
1. **Server-seitiges Paging**: Nur die aktuelle Seite wird geladen (z.B. 50 Einträge)
2. **Filterung**: Benutzer kann nach Location und Sensor filtern
3. **MudBlazor DataGrid**: Professionelle UI-Komponente mit built-in Features

> Hinweis bei 40.000+ Datensätzen:
> - Verwenden Sie konsequent Server-seitiges Paging (typisch 50 Zeilen pro Seite)
> - Laden Sie nur die Spalten, die Sie anzeigen (DTO-Projektion)
> - Vermeiden Sie `StateHasChanged()` pro Datensatz; UI-Updates in Batches
> - Stellen Sie sicher, dass passende Indizes existieren (siehe unten)

**Prinzip:**
```
Simple:    Lade 10.000 Einträge → Zeige 10.000 Einträge
Streaming: Lade 10.000 Einträge → Zeige progressiv
Paging:    Lade 50 Einträge → Zeige 50 Einträge ← Perfekt!
```

### Implementierung

#### 1. MudBlazor installieren

```powershell
cd YourProject.Blazor.Server
dotnet add package MudBlazor
```

In `Program.cs`:

```csharp
builder.Services.AddMudServices();
```

In `App.razor` (im `<head>`):

```html
<link href="https://fonts.googleapis.com/css?family=Roboto:300,400,500,700&display=swap" rel="stylesheet" />
<link href="_content/MudBlazor/MudBlazor.min.css" rel="stylesheet" />
```

Am Ende von `<body>`:

```html
<script src="_content/MudBlazor/MudBlazor.min.js"></script>
```

#### 2. Paging Query erstellen

Erstellen Sie `GetMeasurementsBySensorIdPagedQuery.cs`:

```csharp
using Application.Features.Dtos;
using MediatR;
using Application.Common.Results;

namespace Application.Features.Measurements.Queries;

public sealed record GetMeasurementsBySensorIdPagedQuery(
    int SensorId,
    int Page,
    int PageSize
) : IRequest<PagedResult<GetMeasurementDto>>;
```

#### 3. PagedResult Helper-Klasse

Verwenden Sie die bereits vorhandene Klasse `PagedResult<T>` aus dem Application Layer (`Application.Common.Results`). Fügen Sie in Ihren Dateien das Using hinzu:

```csharp
using Application.Common.Results;
```

#### 4. Paged Query Handler

Erstellen Sie `GetMeasurementsBySensorIdPagedQueryHandler.cs`:

```csharp
using Application.Features.Dtos;
using Application.Interfaces;
using MediatR;
using Application.Common.Results;

namespace Application.Features.Measurements.Queries;

public sealed class GetMeasurementsBySensorIdPagedQueryHandler(IUnitOfWork uow)
    : IRequestHandler<GetMeasurementsBySensorIdPagedQuery, PagedResult<GetMeasurementDto>>
{
    public async Task<PagedResult<GetMeasurementDto>> Handle(
        GetMeasurementsBySensorIdPagedQuery request, 
        CancellationToken cancellationToken)
    {
    var page = request.Page < 1 ? 1 : request.Page;
    var pageSize = request.PageSize < 1 ? 50 : request.PageSize; // 40k+ Datensätze: 50 ist ein guter Default
        var skip = (page - 1) * pageSize;
        
        // Nur die Gesamtanzahl für den Sensor
        var total = await uow.Measurements.CountBySensorIdAsync(
            request.SensorId, cancellationToken);
        
        // Nur die aktuelle Seite laden (z.B. 50 Einträge)
        var items = await uow.Measurements.GetBySensorIdPagedAsync(
            request.SensorId, skip, pageSize, cancellationToken);
        
        var dtos = items.Select(i => new GetMeasurementDto(
            i.Id, i.SensorId, i.Value, i.Timestamp)).ToList();
            
        return PagedResult<GetMeasurementDto>.Success(dtos, total, page, pageSize);
    }
}
```

#### 5. Repository Paging-Methoden

In `MeasurementRepository.cs`:

```csharp
public async Task<int> CountBySensorIdAsync(
    int sensorId, 
    CancellationToken cancellationToken = default)
{
    return await _context.Measurements
        .Where(m => m.SensorId == sensorId)
        .CountAsync(cancellationToken);
}

public async Task<IReadOnlyCollection<Measurement>> GetBySensorIdPagedAsync(
    int sensorId, 
    int skip, 
    int take, 
    CancellationToken cancellationToken = default)
{
    return await _context.Measurements
        .Where(m => m.SensorId == sensorId)
        .AsNoTracking() // 40k+ Datensätze: Tracking vermeiden
        .OrderByDescending(m => m.Timestamp)
        .Skip(skip)
        .Take(take)
        .ToListAsync(cancellationToken);
}
```

> Optional: Direkt im Repository in DTOs projizieren, um Transfer und Materialisierung zu reduzieren:
>
> ```csharp
> return await _context.Measurements
>     .Where(m => m.SensorId == sensorId)
>     .OrderByDescending(m => m.Timestamp)
>     .Skip(skip)
>     .Take(take)
>     .Select(m => new GetMeasurementDto(m.Id, m.SensorId, m.Value, m.Timestamp))
>     .AsNoTracking()
>     .ToListAsync(cancellationToken);
> ```

#### SQL Server: Index-Empfehlungen für hohe Performance

Für Abfragen nach Sensor und absteigendem Zeitstempel empfiehlt sich ein zusammengesetzter Index:

```sql
CREATE INDEX IX_Measurements_Sensor_Timestamp
ON dbo.Measurements (SensorId, Timestamp DESC)
INCLUDE (Value);
```

Dieser Index beschleunigt sowohl `COUNT BY SensorId` als auch das Paging (`ORDER BY Timestamp DESC OFFSET .. FETCH`).

#### 6. Sensors Query für Dropdown

Erstellen Sie `GetAllSensorsQuery.cs`:

```csharp
using Application.Features.Dtos;
using MediatR;
using Application.Common.Results;

namespace Application.Features.Sensors.Queries;

public sealed record GetAllSensorsQuery 
    : IRequest<Result<IReadOnlyCollection<SensorDto>>>;
```

Handler:

```csharp
public sealed class GetAllSensorsQueryHandler(IUnitOfWork uow)
    : IRequestHandler<GetAllSensorsQuery, Result<IReadOnlyCollection<SensorDto>>>
{
    public async Task<Result<IReadOnlyCollection<SensorDto>>> Handle(
        GetAllSensorsQuery request, 
        CancellationToken cancellationToken)
    {
        var sensors = await uow.Sensors.GetAllAsync(cancellationToken);
        
        var dtos = sensors
            .Select(s => new SensorDto(s.Id, s.Name, s.Location))
            .ToArray();
            
        return Result<IReadOnlyCollection<SensorDto>>.Success(dtos);
    }
}
```

#### 7. MudBlazor Razor Component

Erstellen Sie `Components/Pages/MeasurementsMudDataGrid.razor`:

```razor
@page "/measurements-mud-datagrid"
@page "/measurements-paged"
@using Application.Features.Dtos
@using MudBlazor

<PageTitle>Measurements - MudDataGrid</PageTitle>

<h3>Measurements - Server-side Paging with MudBlazor</h3>

<!-- Filterbereich -->
<MudPaper Class="pa-4 mb-4">
    <MudStack Row="true" Spacing="2" AlignItems="AlignItems.Center">
        <!-- Location Dropdown -->
        <MudSelect T="string" 
                   Label="Location" 
                   Dense="true" 
                   Clearable="true"
                   Value="@_selectedLocation" 
                   ValueChanged="OnLocationChanged" 
                   Style="min-width: 220px">
            @foreach (var loc in _locations)
            {
                <MudSelectItem T="string" Value="@loc">@loc</MudSelectItem>
            }
        </MudSelect>

        <!-- Sensor Name Dropdown (abhängig von Location) -->
        <MudSelect T="string" 
                   Label="Sensor Name" 
                   Dense="true" 
                   Clearable="true"
                   Disabled="@string.IsNullOrWhiteSpace(_selectedLocation)"
                   Value="@_selectedName" 
                   ValueChanged="OnNameChanged" 
                   Style="min-width: 220px">
            @foreach (var nm in _availableNames)
            {
                <MudSelectItem T="string" Value="@nm">@nm</MudSelectItem>
            }
        </MudSelect>

        @if (!string.IsNullOrWhiteSpace(_selectedLocation) && 
             !string.IsNullOrWhiteSpace(_selectedName))
        {
            <MudText Typo="Typo.body2">
                Selected: @_selectedLocation - @_selectedName
            </MudText>
        }
    </MudStack>
</MudPaper>

@if (!string.IsNullOrWhiteSpace(_error))
{
    <MudAlert Severity="Severity.Error" Class="mb-4">@_error</MudAlert>
}

<!-- MudDataGrid mit Server-seitigem Paging -->
<MudPaper Class="pa-2">
    <MudDataGrid T="GetMeasurementDto"
                 @ref="_grid"
                 Bordered="true"
                 Dense="true"
                 Hover="true"
                 ServerData="LoadServerData"
                 FixedHeader="true"
                 Height="600px"
                 PageSizeOptions="new int[] { 10, 20, 50, 100 }"
                 RowsPerPage="50">
        <Columns>
            <TemplateColumn T="GetMeasurementDto" Title="Date">
                <CellTemplate Context="ctx">
                    @ctx.Item.Timestamp.ToLocalTime().ToString("dd.MM.yyyy")
                </CellTemplate>
            </TemplateColumn>
            <TemplateColumn T="GetMeasurementDto" Title="Time">
                <CellTemplate Context="ctx">
                    @ctx.Item.Timestamp.ToLocalTime().ToString("HH:mm:ss")
                </CellTemplate>
            </TemplateColumn>
            <TemplateColumn T="GetMeasurementDto" Title="Value">
                <CellTemplate Context="ctx">
                    <div style="text-align:right">
                        @ctx.Item.Value.ToString("F2")
                    </div>
                </CellTemplate>
            </TemplateColumn>
        </Columns>
        <PagerContent>
            @if (_totalItems > 0)
            {
                <MudDataGridPager T="GetMeasurementDto" />
            }
        </PagerContent>
    </MudDataGrid>
</MudPaper>
```

#### 8. Code-Behind mit Filterlogik

Erstellen Sie `Components/Pages/MeasurementsMudDataGrid.razor.cs`:

```csharp
using Application.Features.Dtos;
using Application.Features.Measurements.Queries;
using Application.Features.Sensors.Queries;
using MediatR;
using Microsoft.AspNetCore.Components;
using MudBlazor;

namespace Blazor.Server.Components.Pages;

public partial class MeasurementsMudDataGrid : ComponentBase
{
    [Inject] 
    private IMediator Mediator { get; set; } = default!;

    private MudDataGrid<GetMeasurementDto>? _grid;

    // Filter State
    private string? _selectedLocation;
    private string? _selectedName;

    private readonly HashSet<string> _locations = [];
    private readonly List<string> _availableNames = [];
    private readonly List<(int Id, string Location, string Name)> _allSensors = [];

    private string? _error;
    private int _totalItems;

    protected override async Task OnInitializedAsync()
    {
        await LoadSensorsAsync();
    }

    // Lädt alle Sensoren für die Dropdowns
    private async Task LoadSensorsAsync()
    {
        var result = await Mediator.Send(new GetAllSensorsQuery());
        
        if (!result.IsSuccess || result.Value is null)
        {
            _error = result.Message ?? "Fehler beim Laden der Sensoren";
            return;
        }

        _allSensors.Clear();
        _locations.Clear();
        
        foreach (var s in result.Value)
        {
            _allSensors.Add((s.Id, s.Location, s.Name));
            _locations.Add(s.Location);
        }
        
        UpdateAvailableNames();
    }

    // Aktualisiert verfügbare Sensor-Namen basierend auf Location
    private void UpdateAvailableNames()
    {
        _availableNames.Clear();
        
        if (!string.IsNullOrWhiteSpace(_selectedLocation))
        {
            _availableNames.AddRange(
                _allSensors
                    .Where(s => s.Location == _selectedLocation)
                    .Select(s => s.Name)
                    .Distinct()
                    .OrderBy(n => n));
        }
        
        if (!_availableNames.Contains(_selectedName ?? string.Empty))
        {
            _selectedName = null;
        }
    }

    // Event Handler für Location-Änderung
    private async Task OnLocationChanged(string? newLocation)
    {
        _selectedLocation = newLocation;
        UpdateAvailableNames();
        _totalItems = 0;
        
        if (_grid is not null)
            await _grid.ReloadServerData();
        
        StateHasChanged();
    }

    // Event Handler für Name-Änderung
    private async Task OnNameChanged(string? newName)
    {
        _selectedName = newName;
        _totalItems = 0;
        
        if (_grid is not null)
            await _grid.ReloadServerData();
        
        StateHasChanged();
    }

    // Server-seitiges Laden der Daten für MudDataGrid
    private async Task<GridData<GetMeasurementDto>> LoadServerData(
        GridState<GetMeasurementDto> state)
    {
        try
        {
            // Keine Auswahl? Leere Daten zurückgeben
            if (string.IsNullOrWhiteSpace(_selectedLocation) || 
                string.IsNullOrWhiteSpace(_selectedName))
            {
                _totalItems = 0;
                return new GridData<GetMeasurementDto> 
                { 
                    Items = [], 
                    TotalItems = 0 
                };
            }

            // Sensor-ID ermitteln
            var sensorId = _allSensors
                .FirstOrDefault(s => s.Location == _selectedLocation && 
                                     s.Name == _selectedName).Id;
            
            if (sensorId == 0)
            {
                _totalItems = 0;
                return new GridData<GetMeasurementDto> 
                { 
                    Items = [], 
                    TotalItems = 0 
                };
            }

            // MudDataGrid verwendet 0-basierte Pages
            var page = state.Page + 1; // Backend ist 1-basiert
            var pageSize = state.PageSize > 0 ? state.PageSize : 20;

            Console.WriteLine($"[Paging] Loading page {page}, pageSize {pageSize} for sensor {sensorId}");

            // Query mit Paging senden
            var result = await Mediator.Send(
                new GetMeasurementsBySensorIdPagedQuery(sensorId, page, pageSize));
            
            if (!result.IsSuccess || result.Value is null)
            {
                _error = result.Message ?? "Fehler beim Laden der Messwerte";
                _totalItems = 0;
                return new GridData<GetMeasurementDto> 
                { 
                    Items = [], 
                    TotalItems = 0 
                };
            }

            var data = result.Value;
            _totalItems = data.TotalCount;
            
            Console.WriteLine($"[Paging] Loaded {data.Items.Count} items, total {data.TotalCount}");
            
            return new GridData<GetMeasurementDto>
            {
                Items = data.Items,
                TotalItems = data.TotalCount
            };
        }
        catch (Exception ex)
        {
            _error = ex.Message;
            _totalItems = 0;
            Console.WriteLine($"[Paging] Error: {ex.Message}");
            
            return new GridData<GetMeasurementDto> 
            { 
                Items = [], 
                TotalItems = 0 
            };
        }
    }
}
```

### Test der Paging-Implementierung

1. Navigieren Sie zu `/measurements-mud-datagrid`
2. Wählen Sie eine Location aus (z.B. "Wien")
3. Wählen Sie einen Sensor (z.B. "Temperature_01")
4. Beobachten Sie:
   - Daten erscheinen **sofort** (~100-300ms)
   - Nur 20 Einträge werden geladen
   - Navigation zwischen Seiten ist **blitzschnell**
   - Filterung funktioniert perfekt

**Console-Ausgabe:**
```
[Paging] Loading page 1, pageSize 20 for sensor 3
Executed DbCommand (12ms) [Parameters=[@__sensorId_0='3'], CommandType='Text']
SELECT COUNT(*) FROM [Measurements] WHERE [SensorId] = @__sensorId_0

Executed DbCommand (5ms) [Parameters=[@__sensorId_0='3', @__p_1='0', @__p_2='20'], CommandType='Text']
SELECT [Id], [SensorId], [Value], [Timestamp]
FROM [Measurements]
WHERE [SensorId] = @__sensorId_0
ORDER BY [Timestamp] DESC
OFFSET @__p_1 ROWS FETCH NEXT @__p_2 ROWS ONLY

[Paging] Loaded 20 items, total 5432
```

### Vorteile der Paging-Lösung

✅ **Minimaler Datentransfer**: Nur 20-100 Zeilen pro Request (statt 10.000!)  
✅ **Schnelle Ladezeiten**: < 100ms pro Seite  
✅ **Filterung**: Benutzer findet schnell relevante Daten  
✅ **Professionelle UI**: MudBlazor bietet moderne Komponenten  
✅ **Skalierbar**: Funktioniert auch mit Millionen Datensätzen  
✅ **Server-seitiges Paging**: Datenbank macht die schwere Arbeit mit `OFFSET/FETCH`

---

## Performance-Vergleich

| Aspekt | Simple | Streaming | MudDataGrid + Paging |
|--------|--------|-----------|---------------------|
| **Initiale Ladezeit** | 5-10s | 0.5-1s | 0.1-0.3s ⭐ |
| **Datentransfer** | Alle (~5MB) | Alle (~5MB) | Pro Seite (~20KB) ⭐ |
| **Memory Server** | Hoch | Hoch | Niedrig ⭐ |
| **Memory Client** | Hoch | Hoch | Niedrig ⭐ |
| **SignalR Messages** | 1 groß | ~200 klein | 1 pro Seite ⭐ |
| **Perceived Performance** | Schlecht | Gut | Sehr gut ⭐ |
| **Skalierbarkeit** | ❌ | ⚠️ | ✅ ⭐ |
| **User Experience** | ❌ | ✅ | ✅✅ ⭐ |

---

## Wichtige Konzepte

### 1. SignalR und Blazor Server Rendering

- Jede UI-Interaktion = SignalR-Roundtrip zum Server
- `StateHasChanged()` triggert DOM-Diff-Berechnung
- Batch-Updates reduzieren SignalR-Traffic (z.B. nur alle 50 Items)

**Beispiel:**
```csharp
// ❌ SCHLECHT: Nach jedem Item
foreach (var item in items)
{
    _list.Add(item);
    await InvokeAsync(StateHasChanged); // 10.000 SignalR-Calls!
}

// ✅ GUT: Batch-Updates
for (int i = 0; i < items.Count; i++)
{
    _list.Add(items[i]);
    if (i % 50 == 0)
        await InvokeAsync(StateHasChanged); // Nur 200 SignalR-Calls
}
```

### 2. IAsyncEnumerable für Streaming

```csharp
// MediatR unterstützt Streaming mit CreateStream
await foreach (var item in Mediator.CreateStream(query, cancellationToken))
{
    // Verarbeite Items einzeln, während sie ankommen
    ProcessItem(item);
}
```

**Wichtig:** 
- `yield return` im Handler ermöglicht Streaming
- `[EnumeratorCancellation]` für abbrechbare Streams
- `IStreamRequestHandler<TRequest, TResponse>` statt `IRequestHandler`

### 3. Server-seitiges Paging mit OFFSET/FETCH

```sql
-- Efficient SQL mit OFFSET/FETCH
SELECT * FROM Measurements 
WHERE SensorId = @sensorId
ORDER BY Timestamp DESC
OFFSET @skip ROWS
FETCH NEXT @pageSize ROWS ONLY;
```

**Entity Framework Core:**
```csharp
return await _context.Measurements
    .Where(m => m.SensorId == sensorId)
    .OrderByDescending(m => m.Timestamp)
    .Skip(skip)      // OFFSET
    .Take(pageSize)  // FETCH NEXT
    .ToListAsync(cancellationToken);
```

### 4. Component Lifecycle für Cleanup

```csharp
public class MyComponent : ComponentBase, IDisposable
{
    private CancellationTokenSource _cts = new();
    
    protected override async Task OnInitializedAsync()
    {
        // Starte Background-Task mit Cancellation
        _ = LoadDataAsync(_cts.Token);
    }
    
    public void Dispose()
    {
        _cts.Cancel();  // Wichtig: Stoppe Background-Task!
        _cts.Dispose();
    }
}
```

---

## Best Practices

### ✅ DO

- **Paging verwenden** bei großen Datenmengen (> 100 Einträge)
- **Filterung implementieren** für bessere UX
- **Batch-Updates** bei Streaming (nicht jedes Item rendern)
- **Dispose-Pattern** bei Background-Tasks implementieren
- **CancellationToken** für abbrechbare Operationen verwenden
- **Repository-Pattern** für testbare Datenzugriffe nutzen
- **DTOs verwenden** statt Domain-Entities im UI

### ❌ DON'T

- **Große Collections** direkt in Components halten (> 1000 Items)
- **Alle Daten laden** wenn nur Teile benötigt werden
- **Blocking Calls** in `OnInitializedAsync` (verwenden Sie `async/await`)
- **StateHasChanged()** in Loops ohne Batching aufrufen
- **Memory Leaks** durch fehlende Disposal riskieren
- **Prerendering-Probleme** ignorieren (siehe Modul 2)

---

## Zusammenfassung

In diesem Modul haben Sie drei Implementierungsansätze kennengelernt:

### 🔴 Simple Approach
- **Gut für**: < 100 Einträge
- **Probleme**: Lange Ladezeiten, hoher Speicherverbrauch
- **Use Case**: Kleine Datenmengen, keine Filterung nötig

### 🟡 Streaming Approach
- **Gut für**: 100-1000 Einträge, wenn alle angezeigt werden sollen
- **Vorteile**: Bessere Perceived Performance, progressive Anzeige
- **Use Case**: Reports, Exports, Data-Migration-Anzeigen

### 🟢 Paging + Filtering (Recommended)
- **Gut für**: Beliebig viele Einträge (Millionen!)
- **Vorteile**: Minimaler Datentransfer, schnell, skalierbar
- **Use Case**: Produktive Anwendungen mit großen Datenmengen

### Was Sie gelernt haben:

✅ **Problem erkennen**: Lange Ladezeiten bei großen Datenmengen analysieren  
✅ **Streaming nutzen**: `IAsyncEnumerable` für progressive Anzeige einsetzen  
✅ **Paging implementieren**: Server-seitiges Paging für optimale Performance  
✅ **Filterung hinzufügen**: Benutzer können relevante Daten finden  
✅ **UI-Komponenten nutzen**: MudBlazor für professionelle Oberflächen  
✅ **Performance optimieren**: SignalR-Traffic minimieren mit Batch-Updates

### Nächste Schritte

Im produktiven Einsatz können Sie erweitern:
- **Sortierung**: Spalten-Header anklicken für Sort
- **Export**: Excel/CSV-Export der gefilterten Daten
- **Real-time Updates**: SignalR für Live-Aktualisierung
- **Virtualisierung**: `Virtualize` Component für extrem große Listen
- **Caching**: Server-seitiges Caching für häufige Queries

---

**Quellcode:** Das vollständige Projekt `Iot_MudDataGrid` finden Sie im Workspace!

---

[← Zurück zu Modul 2](../02-lifecycle-events) | [Zurück zur Kursübersicht](../blazor-server-interactivity)
