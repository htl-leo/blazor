---
layout: kurs
title: "Modul 2: Page Lifecycle Events & Prerendering"
dozent: "HTL Leonding"
kurs: "Blazor Server Interactivity"
---

[← Zurück zur Kursübersicht](../blazor-server-interactivity)

# Modul 2: Page Lifecycle Events & Prerendering

## Einleitung

In diesem Modul lernen Sie die **Lifecycle Events** von Blazor Server Komponenten im Detail kennen. Wir werden alle verfügbaren Lifecycle-Methoden überschreiben und auf der Konsole protokollieren, wann sie aufgerufen werden. Dabei werden Sie ein überraschendes Verhalten entdecken: **Die Seite wird beim ersten Aufruf zweimal geladen!**

Dieses Verhalten ist auf das **Prerendering-Konzept** von Blazor Server zurückzuführen, das wir detailliert erklären werden.

## Die Lifecycle Events einer Blazor Komponente

Blazor Server Komponenten durchlaufen während ihrer Existenz verschiedene Phasen. Für jede Phase gibt es spezielle Lifecycle-Methoden, die Sie überschreiben können:

### Übersicht der Lifecycle Events

```
1. SetParametersAsync()      ← Parameter werden gesetzt
2. OnInitialized()            ← Komponente wird initialisiert
3. OnInitializedAsync()       ← Async-Initialisierung
4. OnParametersSet()          ← Parameter wurden gesetzt
5. OnParametersSetAsync()     ← Async-Parameter-Verarbeitung
6. OnAfterRender()            ← Nach dem Rendern
7. OnAfterRenderAsync()       ← Async nach dem Rendern
8. Dispose()                  ← Komponente wird zerstört (wenn IDisposable)
```

## Praktische Demo: Alle Events protokollieren

Erstellen Sie eine neue Seite `Components/Pages/LifecycleDemo.razor`:

```razor
@page "/lifecycle-demo"
@implements IDisposable

<PageTitle>Lifecycle Events Demo</PageTitle>

<h3>Lifecycle Events Demo</h3>

<div class="alert alert-info">
    <p><strong>Anleitung:</strong></p>
    <ul>
        <li>Öffnen Sie die Browser-Entwicklertools (F12)</li>
        <li>Wechseln Sie zur "Console" Tab</li>
        <li>Laden Sie diese Seite neu</li>
        <li>Beobachten Sie die Ausgabe!</li>
    </ul>
</div>

<div class="card">
    <div class="card-body">
        <h5>Counter: @counter</h5>
        <button class="btn btn-primary" @onclick="IncrementCounter">Increment</button>
        <button class="btn btn-secondary" @onclick="ForceRerender">Force Re-render</button>
    </div>
</div>

<div class="mt-3">
    <h5>Lifecycle Event Log:</h5>
    <div class="border p-3" style="max-height: 400px; overflow-y: auto; background: #f8f9fa; font-family: monospace;">
        @foreach (var log in eventLog)
        {
            <div>@log</div>
        }
    </div>
</div>
```

Jetzt das Code-Behind `LifecycleDemo.razor.cs`:

```csharp
using Microsoft.AspNetCore.Components;
using Microsoft.JSInterop;

namespace YourProject.Blazor.Server.Components.Pages;

public partial class LifecycleDemo : IDisposable
{
    [Inject]
    private IJSRuntime JSRuntime { get; set; } = default!;

    private int counter = 0;
    private int renderCount = 0;
    private List<string> eventLog = new();
    
    // Eindeutige ID für diese Instanz (um Prerendering zu demonstrieren)
    private readonly Guid instanceId = Guid.NewGuid();

    // 1. SetParametersAsync - Wird als ERSTES aufgerufen
    public override async Task SetParametersAsync(ParameterView parameters)
    {
        var timestamp = DateTime.Now.ToString("HH:mm:ss.fff");
        var message = $"[{timestamp}] [{instanceId:N}] 1. SetParametersAsync()";
        
        Console.WriteLine(message);
        await LogToConsole(message);
        eventLog.Add(message);

        await base.SetParametersAsync(parameters);
    }

    // 2. OnInitialized - Synchrone Initialisierung
    protected override void OnInitialized()
    {
        var timestamp = DateTime.Now.ToString("HH:mm:ss.fff");
        var message = $"[{timestamp}] [{instanceId:N}] 2. OnInitialized()";
        
        Console.WriteLine(message);
        eventLog.Add(message);

        base.OnInitialized();
    }

    // 3. OnInitializedAsync - Asynchrone Initialisierung
    protected override async Task OnInitializedAsync()
    {
        var timestamp = DateTime.Now.ToString("HH:mm:ss.fff");
        var message = $"[{timestamp}] [{instanceId:N}] 3. OnInitializedAsync()";
        
        Console.WriteLine(message);
        await LogToConsole(message);
        eventLog.Add(message);

        // Simuliere async Operation
        await Task.Delay(100);

        await base.OnInitializedAsync();
    }

    // 4. OnParametersSet - Wird nach OnInitialized aufgerufen
    protected override void OnParametersSet()
    {
        var timestamp = DateTime.Now.ToString("HH:mm:ss.fff");
        var message = $"[{timestamp}] [{instanceId:N}] 4. OnParametersSet()";
        
        Console.WriteLine(message);
        eventLog.Add(message);

        base.OnParametersSet();
    }

    // 5. OnParametersSetAsync - Async Parameter-Verarbeitung
    protected override async Task OnParametersSetAsync()
    {
        var timestamp = DateTime.Now.ToString("HH:mm:ss.fff");
        var message = $"[{timestamp}] [{instanceId:N}] 5. OnParametersSetAsync()";
        
        Console.WriteLine(message);
        await LogToConsole(message);
        eventLog.Add(message);

        await base.OnParametersSetAsync();
    }

    // 6. OnAfterRender - Nach JEDEM Rendering
    protected override void OnAfterRender(bool firstRender)
    {
        renderCount++;
        var timestamp = DateTime.Now.ToString("HH:mm:ss.fff");
        var message = $"[{timestamp}] [{instanceId:N}] 6. OnAfterRender(firstRender: {firstRender}) - Render #{renderCount}";
        
        Console.WriteLine(message);
        // Achtung: eventLog.Add() hier würde zu Endlosschleife führen!
        // (Add → StateHasChanged → Render → OnAfterRender → Add → ...)

        base.OnAfterRender(firstRender);
    }

    // 7. OnAfterRenderAsync - Async nach dem Rendering
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        var timestamp = DateTime.Now.ToString("HH:mm:ss.fff");
        var message = $"[{timestamp}] [{instanceId:N}] 7. OnAfterRenderAsync(firstRender: {firstRender})";
        
        Console.WriteLine(message);
        await LogToConsole(message);
        
        // Nur beim ersten Render zur eventLog hinzufügen
        if (firstRender)
        {
            eventLog.Add($"{message} - ERSTES RENDER ABGESCHLOSSEN");
        }

        await base.OnAfterRenderAsync(firstRender);
    }

    // 8. Dispose - Komponente wird zerstört
    public void Dispose()
    {
        var timestamp = DateTime.Now.ToString("HH:mm:ss.fff");
        var message = $"[{timestamp}] [{instanceId:N}] 8. Dispose()";
        
        Console.WriteLine(message);
        // Kein eventLog.Add() hier - Komponente wird gerade zerstört!
    }

    // Helper Methods
    private async Task LogToConsole(string message)
    {
        try
        {
            await JSRuntime.InvokeVoidAsync("console.log", message);
        }
        catch
        {
            // Prerendering: JSInterop nicht verfügbar
            // Das ist OK, Console.WriteLine() funktioniert trotzdem
        }
    }

    private void IncrementCounter()
    {
        counter++;
        Console.WriteLine($"[{DateTime.Now:HH:mm:ss.fff}] Button clicked - Counter: {counter}");
    }

    private void ForceRerender()
    {
        Console.WriteLine($"[{DateTime.Now:HH:mm:ss.fff}] Force Re-render triggered");
        StateHasChanged();
    }
}
```

## Das Experiment durchführen

### Schritt 1: Seite starten

1. Starten Sie die Anwendung: `dotnet run`
2. Öffnen Sie den Browser
3. Öffnen Sie die Entwicklertools (F12)
4. Navigieren Sie zu `/lifecycle-demo`

### Schritt 2: Console-Ausgabe beobachten

Sie werden folgende Ausgabe sehen (die GUID ist bei jedem Lauf unterschiedlich):

```
[10:15:30.123] [a1b2c3d4...] 1. SetParametersAsync()
[10:15:30.125] [a1b2c3d4...] 2. OnInitialized()
[10:15:30.127] [a1b2c3d4...] 3. OnInitializedAsync()
[10:15:30.128] [a1b2c3d4...] 4. OnParametersSet()
[10:15:30.130] [a1b2c3d4...] 5. OnParametersSetAsync()
[10:15:30.232] [a1b2c3d4...] 6. OnAfterRender(firstRender: True) - Render #1
[10:15:30.234] [a1b2c3d4...] 7. OnAfterRenderAsync(firstRender: True)
[10:15:30.235] [a1b2c3d4...] 8. Dispose()

[10:15:30.240] [e5f6g7h8...] 1. SetParametersAsync()
[10:15:30.242] [e5f6g7h8...] 2. OnInitialized()
[10:15:30.244] [e5f6g7h8...] 3. OnInitializedAsync()
[10:15:30.245] [e5f6g7h8...] 4. OnParametersSet()
[10:15:30.247] [e5f6g7h8...] 5. OnParametersSetAsync()
[10:15:30.350] [e5f6g7h8...] 6. OnAfterRender(firstRender: True) - Render #1
[10:15:30.352] [e5f6g7h8...] 7. OnAfterRenderAsync(firstRender: True)
```

### Überraschung: Zwei verschiedene Instance-IDs! 🤔

Beachten Sie:
- Die komplette Lifecycle-Sequenz wird **zweimal** durchlaufen
- Es gibt **zwei unterschiedliche Instance-IDs** (a1b2c3d4... und e5f6g7h8...)
- Die erste Instanz wird mit `Dispose()` zerstört
- Erst die zweite Instanz bleibt aktiv

**Warum passiert das?** → **Prerendering!**

## Das Prerendering-Konzept

### Was ist Prerendering?

**Prerendering** (auch Server-Side Rendering oder SSR genannt) ist ein Konzept, bei dem die Seite **auf dem Server vorgerendert** wird, bevor die SignalR-Verbindung aufgebaut ist.

### Der Ablauf im Detail

#### Phase 1: Prerendering (Erste Instanz)

```
1. Browser sendet HTTP-Request an Server
2. Server erstellt Komponenten-Instanz
3. Alle Lifecycle Events werden durchlaufen
4. Server rendert HTML
5. HTML wird zum Browser geschickt
6. Komponenten-Instanz wird ZERSTÖRT (Dispose!)
7. Browser zeigt HTML an (statisch, noch nicht interaktiv)
```

**Zeitpunkt:** ~0-100ms nach Request

**Zweck:** 
- Schnelles Initial-Rendering
- Bessere SEO (Suchmaschinen sehen fertiges HTML)
- Bessere Perceived Performance

#### Phase 2: SignalR-Verbindung (Zweite Instanz)

```
8. Browser lädt blazor.web.js
9. JavaScript etabliert SignalR-Verbindung
10. Server erstellt NEUE Komponenten-Instanz
11. Alle Lifecycle Events werden ERNEUT durchlaufen
12. Server sendet nur Render-Diffs über SignalR
13. Diese Instanz bleibt aktiv (kein Dispose!)
14. Seite ist jetzt INTERAKTIV
```

**Zeitpunkt:** ~100-500ms nach Request

**Zweck:**
- Interaktivität aktivieren
- SignalR-Verbindung für Echtzeit-Updates
- Event-Handling aktivieren

### Visualisierung

```
Timeline beim ersten Laden:

0ms     [HTTP Request]
        ↓
50ms    [Prerender] Instanz 1: SetParameters → OnInit → ... → OnAfterRender → Dispose
        ↓
100ms   [HTML zum Browser]
        Browser zeigt Seite (statisch)
        ↓
200ms   [SignalR WebSocket aufbaut]
        ↓
250ms   [Interactive] Instanz 2: SetParameters → OnInit → ... → OnAfterRender
        ↓
300ms   [Seite ist interaktiv!]
```

## Probleme durch Prerendering

### Problem 1: JSInterop funktioniert nicht beim Prerendering

```csharp
protected override async Task OnInitializedAsync()
{
    try
    {
        // ❌ Fehler beim Prerendering!
        await JSRuntime.InvokeVoidAsync("console.log", "Hello");
    }
    catch (InvalidOperationException ex)
    {
        // JavaScript interop calls cannot be issued at this time.
        // This is because the component is being prerendered.
    }
}
```

**Lösung:** Nur in `OnAfterRenderAsync` mit `firstRender`-Check:

```csharp
protected override async Task OnAfterRenderAsync(bool firstRender)
{
    if (firstRender)
    {
        // ✅ Funktioniert! SignalR ist aktiv
        await JSRuntime.InvokeVoidAsync("console.log", "Hello");
    }
}
```

### Problem 2: Doppelte Datenbank-Abfragen

```csharp
protected override async Task OnInitializedAsync()
{
    // ❌ Wird zweimal aufgerufen!
    // 1x beim Prerendering
    // 1x beim Interactive Rendering
    sensors = await Mediator.Send(new GetAllSensorsQuery());
}
```

**Auswirkung:**
- Doppelte Last auf Datenbank
- Verschwendete Ressourcen
- Längere Ladezeiten

### Problem 3: State geht verloren

```csharp
private int counter = 0;

protected override void OnInitialized()
{
    counter = 42; // ❌ Geht beim Prerendering verloren!
    // Die Prerender-Instanz wird mit Dispose() zerstört
}
```

**Bei der zweiten Instanz ist `counter` wieder `0`!**

## Prerendering deaktivieren (wenn nötig)

Manchmal ist Prerendering nicht gewünscht. So deaktivieren Sie es:

### Option 1: Pro Komponente

In der `@page` Directive in der `.razor`-Datei:

```razor
@page "/lifecycle-demo"
@rendermode InteractiveServer
```

**Achtung:** Diese Direktive muss in neueren Blazor-Versionen in der aufrufenden Komponente gesetzt werden!

### Option 2: Global in Program.cs

```csharp
// Statt:
app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode();

// Verwenden:
app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode(options => 
    {
        options.RootComponents.Add<App>("#app");
    });
```

### Option 3: Spezifische Komponenten ausschließen

In `App.razor`:

```razor
<Routes @rendermode="InteractiveServer" />
```

**Aber:** Dann wird nichts mehr prerendert!

### Option 4: Selektiv mit RenderModeAttribute

In `_Imports.razor`:

```razor
@using Microsoft.AspNetCore.Components.Web
```

In der Komponente:

```csharp
@attribute [RenderModeInteractiveServer(prerender: false)]
```

## Best Practices für Prerendering

### ✅ DO: Prerendering bewusst nutzen

```csharp
protected override async Task OnInitializedAsync()
{
    // Daten laden ist OK - wird gecacht
    sensors = await Mediator.Send(new GetAllSensorsQuery());
}

protected override async Task OnAfterRenderAsync(bool firstRender)
{
    if (firstRender)
    {
        // JS-Calls nur hier
        await JSRuntime.InvokeVoidAsync("initializeMap", "mapDiv");
    }
}
```

### ✅ DO: firstRender-Parameter nutzen

```csharp
protected override async Task OnAfterRenderAsync(bool firstRender)
{
    if (firstRender)
    {
        // Einmalige Initialisierung
        await SetupComponent();
    }
}
```

### ✅ DO: State zwischen Prerender und Interactive teilen

```csharp
// In Program.cs
builder.Services.AddScoped<StateContainer>();

// In Komponente
[Inject]
private StateContainer State { get; set; } = default!;

protected override void OnInitialized()
{
    State.Counter = 42; // Wird zwischen Instanzen geteilt
}
```

### ❌ DON'T: JS Interop vor OnAfterRender

```csharp
protected override async Task OnInitializedAsync()
{
    // ❌ Fehler beim Prerendering!
    await JSRuntime.InvokeVoidAsync("alert", "Hello");
}
```

### ❌ DON'T: Browser-spezifische APIs vor OnAfterRender

```csharp
protected override void OnInitialized()
{
    // ❌ window, document, localStorage nicht verfügbar!
    // var width = await JSRuntime.InvokeAsync<int>("getWindowWidth");
}
```

### ❌ DON'T: Teure Operationen zweimal ausführen

```csharp
protected override async Task OnInitializedAsync()
{
    // ❌ Wird zweimal ausgeführt!
    await Task.Delay(5000); // 5 Sekunden Wartezeit × 2 = 10 Sekunden!
}
```

**Besser:**

```csharp
protected override async Task OnAfterRenderAsync(bool firstRender)
{
    if (firstRender)
    {
        // ✅ Nur einmal
        await Task.Delay(5000);
        StateHasChanged();
    }
}
```

## Zusammenfassung

Sie haben gelernt:

✅ **Lifecycle Events**: Alle 7 Lifecycle-Methoden und ihre Reihenfolge
✅ **Prerendering**: Warum die Seite zweimal geladen wird
✅ **Zwei Instanzen**: Prerender-Instanz wird zerstört, Interactive-Instanz bleibt
✅ **JSInterop**: Nur in `OnAfterRenderAsync` mit `firstRender`-Check verwenden
✅ **Best Practices**: Wie man Prerendering optimal nutzt
✅ **Deaktivierung**: Wie man Prerendering ausschaltet (falls nötig)

### Die wichtigsten Erkenntnisse

1. **Prerendering ist Standard** in Blazor Server - die erste Instanz dient nur zum HTML-Generieren
2. **Zwei separate Instanzen** - mit eigenen Lifecycle-Durchläufen
3. **firstRender-Parameter** ist der Schlüssel zum Unterscheiden der Phasen
4. **JSInterop nur nach erstem Render** - vorher ist JavaScript nicht verfügbar

Im nächsten Modul werden wir uns mit **State Management** und **Component Communication** beschäftigen.

---

[← Zurück zu Modul 1](../blazor-server-interactivity/01-clean-architecture-setup) | [Zurück zur Kursübersicht](../blazor-server-interactivity)

**Kontakt:** kurse@htl-absolventen.at
