---
layout: kurs
title: "Blazor Server Interactivity"
dozent: "HTL Leonding"
dauer: "8 Stunden"
level: "Fortgeschritten"
---

Ein umfassender Kurs über Blazor Server und die Entwicklung interaktiver Webanwendungen mit .NET.

## Überblick: Was ist Blazor Server Interactivity?

**Blazor Server** ist ein Framework von Microsoft, das Teil der ASP.NET Core-Plattform ist und es Entwicklern ermöglicht, interaktive Webanwendungen mit C# statt JavaScript zu erstellen. Es repräsentiert einen paradigmatischen Ansatz in der modernen Webentwicklung, bei dem die Programmierlogik vollständig in C# geschrieben werden kann.

### Grundkonzept

Blazor Server ermöglicht die Entwicklung von Single-Page-Applications (SPAs) mit folgenden Kernmerkmalen:

- **C#-basierte Entwicklung**: Sowohl Frontend als auch Backend werden in C# geschrieben
- **Komponentenbasierte Architektur**: UI wird in wiederverwendbare Razor-Komponenten aufgeteilt
- **Echtzeit-Interaktivität**: Benutzerinteraktionen werden in Echtzeit verarbeitet
- **Server-seitige Verarbeitung**: Die gesamte Anwendungslogik läuft auf dem Server

### Technische Funktionsweise

Die technische Architektur von Blazor Server basiert auf einem innovativen Kommunikationsmodell:

#### 1. **SignalR-Verbindung**

Blazor Server nutzt **SignalR** (eine Real-Time Communication Library von Microsoft) als Kommunikationskanal zwischen Browser und Server:

```
Browser (Client) ←─── SignalR WebSocket ───→ Server (.NET Runtime)
```

- Beim ersten Laden der Seite wird eine **persistente WebSocket-Verbindung** aufgebaut
- Diese Verbindung bleibt während der gesamten Sitzung aktiv
- Bi-direktionale Kommunikation ermöglicht schnelle Datenübertragung in beide Richtungen

#### 2. **Rendering-Prozess**

Der Rendering-Prozess läuft wie folgt ab:

**Initial Load (Erster Seitenaufruf):**
1. Browser fordert die Seite an
2. Server sendet minimales HTML-Gerüst + Blazor JavaScript-Client
3. JavaScript etabliert SignalR-Verbindung zum Server
4. Server rendert Blazor-Komponenten und sendet gerenderten HTML über SignalR
5. Browser zeigt die fertige Seite an

**Benutzerinteraktion (z.B. Button-Click):**
1. JavaScript im Browser fängt das Event ab
2. Event-Information wird über SignalR an Server gesendet
3. Server führt entsprechenden C#-Event-Handler aus
4. Komponenten-State wird aktualisiert
5. Server berechnet DOM-Diff (nur Änderungen)
6. Nur die Änderungen werden als **UI-Diff** zurück zum Browser gesendet
7. Blazor JavaScript aktualisiert den DOM mit den Änderungen

#### 3. **Component State Management**

Der Anwendungs-State wird vollständig auf dem Server gehalten:

```csharp
@page "/counter"

<h1>Counter</h1>
<p>Current count: @currentCount</p>
<button @onclick="IncrementCount">Click me</button>

@code {
    private int currentCount = 0;

    private void IncrementCount()
    {
        currentCount++;  // State lebt auf dem Server!
    }
}
```

- Jede Komponenten-Instanz hat ihren eigenen State auf dem Server
- Bei jedem Re-Render wird nur das DOM-Diff übertragen
- Der Browser hat keinen Zugriff auf den vollständigen State
#### 4. **Circuit-Konzept**

Blazor Server führt das Konzept eines **Circuits** ein:

- Ein **Circuit** repräsentiert eine aktive Benutzersitzung
  - SignalR-Verbindung
Ein praxisorientierter Kurs über Blazor Server Interactivity mit Fokus auf Performance-Optimierung bei großen Datenmengen.

## Voraussetzungen

Dieser Kurs baut auf dem Vorgängerkurs zu **Clean Architecture** auf, in dem folgende Konzepte behandelt wurden:

- **Clean Architecture Prinzipien** (Domain, Application, Infrastructure, Presentation Layers)
- **CQRS-Pattern** (Command Query Responsibility Segregation) mit MediatR
- **Repository-Pattern** mit Unit of Work
- **Entity Framework Core** als ORM

### Demo-Datenmodell

Die Anwendung arbeitet mit IoT-Messdaten:
- **Sensoren** mit Location und Name
- **Messwerte** (Measurements) mit Timestamp und numerischem Value
- Große Datenmengen (mehrere tausend Datensätze)
  - Service-Instanzen (DI Container)
  - Event-Handler
- Circuit wird beim Verbindungsabbruch automatisch nach Timeout (Standard: 3 Minuten) aufgeräumt
---
#### 5. **Vorteile der Server-seitigen Verarbeitung**
## Übungsablauf
**Security:**
In dieser Übung entwickeln wir in drei Schritten eine performante Blazor Server Anwendung zur Darstellung von IoT-Messdaten.
- Keine Übertragung großer Datenmengen zum Client
### Projektstruktur
- Voller Zugriff auf .NET-Ökosystem und NuGet-Packages
Das Projekt `Iot_MudDataGrid` folgt der Clean Architecture:
- Jede Benutzerinteraktion erfordert Server-Roundtrip
```
Iot_MudDataGrid/
├── Domain/              # Entities, Interfaces, Specifications
├── Application/         # CQRS Queries/Commands, DTOs
├── Infrastructure/      # EF Core, Repositories, UnitOfWork
└── Blazor.Server/       # UI-Layer (Razor Components)
```
- Server muss viele gleichzeitige Circuits verwalten
---

## Schritt 1: Einfache HTML-Tabelle (Measurements Simple)

### Problem

Bei großen Datenmengen führt das synchrone Laden aller Datensätze zu langen Wartezeiten. Der Benutzer sieht zunächst nur "Loading..." bis alle Daten geladen und gerendert sind.

### Implementation

**Query Handler (Application Layer):**

```csharp
public sealed class GetAllMeasurementsSimpleQueryHandler(IUnitOfWork uow) 
  : IRequestHandler<GetAllMeasurementsSimpleQuery, Result<IReadOnlyCollection<GetMeasurementWithSensorDto>>>
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

**Razor Component (MeasurementsSimple.razor):**

```cshtml
@page "/measurements-simple"
@using Application.Features.Dtos

@if (_error is not null)
{
  <text>@_error</text>
}
else if (_items is null)
{
  <text>Loading...</text>
}
else
{
  <!DOCTYPE html>
  <html>
  <head>
    <meta charset="utf-8" />
    <title>Measurements Simple</title>
  </head>
  <body>
    <table border="1" cellpadding="4" cellspacing="0">
      <thead>
        <tr>
          <th>Sensor</th>
          <th>Location</th>
          <th>Date</th>
          <th>Time</th>
          <th>Value</th>
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
  </body>
  </html>
}
```

**Code-Behind (MeasurementsSimple.razor.cs):**

```csharp
public partial class MeasurementsSimple : ComponentBase
{
  [Inject] private IMediator Mediator { get; set; } = default!;

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

### Nachteile

❌ **Lange Ladezeit**: Benutzer sieht lange "Loading..."  
❌ **Hoher Speicherverbrauch**: Alle Daten werden im Speicher gehalten  
❌ **Langsames Rendering**: Browser muss große HTML-Tabelle auf einmal rendern  
❌ **Schlechte User Experience**: Keine Rückmeldung während des Ladens

---

## Schritt 2: Streaming für schnellere Latenz (Measurements Stream)

### Lösung

Statt alle Daten auf einmal zu laden, verwenden wir **IAsyncEnumerable** für Streaming. Die ersten Datensätze werden sofort angezeigt, während weitere im Hintergrund nachgeladen werden.

### Implementation

**Query Handler mit Streaming:**

```csharp
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

**Razor Component (MeasurementsStream.razor):**

```cshtml
@page "/measurements-stream"

<PageTitle>Measurements</PageTitle>

@if (_error is not null)
{
  <text>@_error</text>
}
else if (_isLoading)
{
  <text>Loading...</text>
}
else
{
  <table border="1" cellpadding="4" cellspacing="0">
    <thead>
      <tr>
        <th>Sensor</th>
        <th>Location</th>
        <th>Date</th>
        <th>Time</th>
        <th>Value</th>
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

**Code-Behind mit Streaming-Logik:**

```csharp
public partial class MeasurementsStream : ComponentBase, IDisposable
{
  private readonly List<GetMeasurementWithSensorDto> _items = [];
  private bool _isLoading = true;
  private string? _error;
  private readonly CancellationTokenSource _cts = new();
  private bool _disposed;

  [Inject] private IMediator Mediator { get; set; } = default!;

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
      int i = 0;
      // Verwendet MediatR.CreateStream für IAsyncEnumerable
      await foreach (var dto in Mediator.CreateStream(
        new GetAllMeasurementsStreamQuery(), ct))
      {
        _items.Add(dto);
                
        // Nach dem ersten Batch: "Loading..." ausblenden
        if (_isLoading && _items.Count > 0)
          _isLoading = false;

        // UI-Update alle 50 Datensätze (Performance-Optimierung)
        if (++i % 50 == 0)
        {
          await InvokeAsync(StateHasChanged);
          Console.WriteLine($"Received {_items.Count} measurements...");
        }
      }
      // Finales UI-Update
      await InvokeAsync(StateHasChanged);
    }
    catch (OperationCanceledException)
    {
      // Normal beim Verlassen der Seite
    }
    catch (Exception ex)
    {
      _error = ex.Message;
      await InvokeAsync(StateHasChanged);
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

### Vorteile

✅ **Sofortiges Feedback**: Erste Daten erscheinen fast sofort  
✅ **Progressive Rendering**: Tabelle wird schrittweise gefüllt  
✅ **Bessere Perceived Performance**: Benutzer sieht sofort Ergebnisse  
✅ **Batch-Updates**: `StateHasChanged()` nur alle 50 Einträge → weniger SignalR-Traffic

### Nachteile

❌ **Immer noch viele Daten**: Alle Datensätze werden geladen  
❌ **Keine Filterung**: Benutzer sieht alle Messwerte  
❌ **Keine Pagination**: Schwer navigierbar bei tausenden Zeilen

---

## Schritt 3: MudBlazor DataGrid mit Paging und Filterung

### Lösung

Die beste Performance erreichen wir durch:
1. **Server-seitiges Paging**: Nur die aktuelle Seite wird geladen
2. **Filterung**: Benutzer kann nach Location und Sensor filtern
3. **MudBlazor DataGrid**: Professionelle UI-Komponente mit built-in Features

### Implementation

**Paged Query Handler:**

```csharp
public sealed class GetMeasurementsBySensorIdPagedQueryHandler(IUnitOfWork uow)
  : IRequestHandler<GetMeasurementsBySensorIdPagedQuery, PagedResult<GetMeasurementDto>>
{
  public async Task<PagedResult<GetMeasurementDto>> Handle(
    GetMeasurementsBySensorIdPagedQuery request, 
    CancellationToken cancellationToken)
  {
    var page = request.Page < 1 ? 1 : request.Page;
    var pageSize = request.PageSize < 1 ? 50 : request.PageSize;
    var skip = (page - 1) * pageSize;
        
    // Nur die Gesamtanzahl für den Sensor
    var total = await uow.Measurements.CountBySensorIdAsync(
      request.SensorId, cancellationToken);
        
    // Nur die aktuelle Seite laden (z.B. 20 Einträge)
    var items = await uow.Measurements.GetBySensorIdPagedAsync(
      request.SensorId, skip, pageSize, cancellationToken);
        
    var dtos = items.Select(i => new GetMeasurementDto(
      i.Id, i.SensorId, i.Value, i.Timestamp)).ToList();
            
    return PagedResult<GetMeasurementDto>.Success(dtos, total, page, pageSize);
  }
}
```

**Razor Component mit MudBlazor (MeasurementsMudDataGrid.razor):**

```cshtml
@page "/measurements-mud-datagrid"
@page "/measurements-paged"
@using Application.Features.Dtos
@using MudBlazor

<PageTitle>Measurements MudDataGrid</PageTitle>

<!-- Filterbereich -->
<MudPaper Class="pa-4" Style="width:50%; margin: 0 auto;">
  <MudStack Row="true" Spacing="2" AlignItems="AlignItems.Center">
    <!-- Location Dropdown -->
    <MudSelect T="string" Label="Location" Dense="true" Clearable="true"
           Value="@_selectedLocation" ValueChanged="OnLocationChanged" 
           Style="min-width: 220px">
      @foreach (var loc in _locations)
      {
        <MudSelectItem T="string" Value="@loc">@loc</MudSelectItem>
      }
    </MudSelect>

    <!-- Sensor Name Dropdown (abhängig von Location) -->
    <MudSelect T="string" Label="Name" Dense="true" Clearable="true"
           Disabled="@string.IsNullOrWhiteSpace(_selectedLocation)"
           Value="@_selectedName" ValueChanged="OnNameChanged" 
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
  <MudAlert Severity="Severity.Error" Class="mt-4">@_error</MudAlert>
}

<!-- MudDataGrid mit Server-seitigem Paging -->
<MudPaper Class="pa-2 mt-4" Style="width:50%; margin: 0 auto;">
  <MudDataGrid T="GetMeasurementDto"
         @ref="_grid"
         Bordered="true"
         Dense="true"
         Hover="true"
         ServerData="LoadServerData"
         PageSizeOptions="new int[] { 10, 20, 50, 100 }"
         RowsPerPage="20">
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

**Code-Behind mit Filterlogik:**

```csharp
public partial class MeasurementsMudDataGrid : ComponentBase
{
  [Inject] private IMediator Mediator { get; set; } = default!;

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
      return new GridData<GetMeasurementDto> 
      { 
        Items = [], 
        TotalItems = 0 
      };
    }
  }
}
```

### Vorteile

✅ **Minimaler Datentransfer**: Nur 20-100 Zeilen pro Request  
✅ **Schnelle Ladezeiten**: < 100ms pro Seite  
✅ **Filterung**: Benutzer findet schnell relevante Daten  
✅ **Professionelle UI**: MudBlazor bietet moderne Komponenten  
✅ **Skalierbar**: Funktioniert auch mit Millionen Datensätzen  
✅ **Server-seitiges Paging**: Datenbank macht die schwere Arbeit

---

## Performance-Vergleich

| Aspekt | Simple | Streaming | MudDataGrid + Paging |
|--------|--------|-----------|---------------------|
| **Initiale Ladezeit** | 5-10s | 0.5-1s | 0.1-0.3s |
| **Datentransfer** | Alle (~5MB) | Alle (~5MB) | Pro Seite (~20KB) |
| **Memory Server** | Hoch | Hoch | Niedrig |
| **Memory Client** | Hoch | Hoch | Niedrig |
| **SignalR Messages** | 1 groß | Viele klein | 1 pro Seite |
| **Perceived Performance** | Schlecht | Gut | Sehr gut |
| **Skalierbarkeit** | ❌ | ⚠️ | ✅ |

---

## Wichtige Konzepte

### 1. SignalR und Blazor Server Rendering

- Jede UI-Interaktion = SignalR-Roundtrip zum Server
- `StateHasChanged()` triggert DOM-Diff-Berechnung
- Batch-Updates reduzieren SignalR-Traffic

### 2. IAsyncEnumerable für Streaming

```csharp
// MediatR unterstützt Streaming
await foreach (var item in Mediator.CreateStream(query, cancellationToken))
{
  // Verarbeite Items einzeln
}
```

### 3. Server-seitiges Paging

```sql
-- Efficient SQL mit OFFSET/FETCH
SELECT * FROM Measurements 
WHERE SensorId = @sensorId
ORDER BY Timestamp DESC
OFFSET @skip ROWS
FETCH NEXT @pageSize ROWS ONLY;
```

### 4. Component Lifecycle

- `OnInitializedAsync()`: Erste Initialisierung
- `Dispose()`: Cleanup (wichtig für Streaming!)
- `StateHasChanged()`: Manueller Re-Render trigger
- Reconnection-Logik erforderlich

### Vergleich: Blazor Server vs. Blazor WebAssembly
## Best Practices

### ✅ DO

- **Paging verwenden** bei großen Datenmengen
- **Filterung implementieren** für bessere UX
- **Batch-Updates** bei Streaming (nicht jedes Item)
- **Dispose-Pattern** bei Background-Tasks
- **CancellationToken** für abbrechbare Operationen
- **Repository-Pattern** für testbare Datenzugriffe

### ❌ DON'T

- **Große Collections** direkt in Components halten
- **Alle Daten laden** wenn nur Teile benötigt werden
- **Blocking Calls** in `OnInitializedAsync`
- **StateHasChanged()** in Loops ohne Batching
- **Memory Leaks** durch fehlende Disposal

---

## Zusammenfassung

In dieser Übung haben Sie gelernt:

1. **Problem erkennen**: Lange Ladezeiten bei großen Datenmengen
2. **Streaming nutzen**: `IAsyncEnumerable` für progressive Anzeige
3. **Paging implementieren**: Server-seitiges Paging für optimale Performance
4. **Filterung hinzufügen**: Benutzer können relevante Daten finden
5. **UI-Komponenten nutzen**: MudBlazor für professionelle Oberflächen

### Nächste Schritte
| Aspekt | Blazor Server | Blazor WebAssembly |
- Sortierung implementieren
- Export-Funktionen (Excel, CSV)
- Real-time Updates mit SignalR
- Virtualisierung für extrem große Listen
- Circuit und Session State
**Quellcode:** `Iot_MudDataGrid` im Workspace
- State Container Pattern
- Scoped vs. Singleton Services

### Modul 4: SignalR-Integration (1,5 Stunden)
- Verbindungsmanagement
- Reconnection-Strategien
- Circuit Handler
- Performance-Optimierung

### Modul 5: Fortgeschrittene Themen (1 Stunde)
- JavaScript Interop
- Custom Event Arguments
- Prerendering
- Deployment und Hosting

---

## Voraussetzungen

- Grundkenntnisse in C# und .NET
- HTML/CSS Basics
- Verständnis von Web-Anwendungsarchitekturen
- Visual Studio 2022 oder Visual Studio Code

## Lernziele

Nach Abschluss dieses Kurses können Sie:

✅ Interaktive Webanwendungen mit Blazor Server entwickeln
✅ Das SignalR-Kommunikationsmodell verstehen und anwenden
✅ Komponentenbasierte Architekturen entwerfen
✅ State Management effektiv implementieren
✅ Performance-Optimierungen durchführen
✅ Produktionsreife Blazor Server Apps deployen

---

## Materialien

- Slides und Code-Beispiele
- Übungsaufgaben mit Lösungen
- Best Practices Checkliste
- Links zu weiterführenden Ressourcen

## Kursmodule

Dieser Kurs ist in mehrere Module unterteilt:

1. [**Modul 1: Setup auf Clean Architecture Stack**](blazor-server-interactivity/01-clean-architecture-setup) - Aufbau einer Blazor Server Anwendung auf einem bestehenden Clean Architecture Stack
2. [**Modul 2: Page Lifecycle Events & Prerendering**](blazor-server-interactivity/02-lifecycle-events) - Detaillierte Untersuchung der Lifecycle Events und des Prerendering-Konzepts

---

**Kontakt:** kurse@htl-absolventen.at
