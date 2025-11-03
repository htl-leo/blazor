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

### Projektstruktur

Das Projekt `Iot_MudDataGrid` folgt der Clean Architecture:

```
Iot_MudDataGrid/
├── Domain/              # Entities, Interfaces, Specifications
├── Application/         # CQRS Queries/Commands, DTOs
├── Infrastructure/      # EF Core, Repositories, UnitOfWork
└── Blazor.Server/       # UI-Layer (Razor Components)
```

---

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

1. [**Modul 1: Setup auf Clean Architecture Stack**](01-clean-architecture-setup) - Aufbau einer Blazor Server Anwendung auf einem bestehenden Clean Architecture Stack
2. [**Modul 2: Page Lifecycle Events & Prerendering**](02-lifecycle-events) - Detaillierte Untersuchung der Lifecycle Events und des Prerendering-Konzepts
3. [**Modul 3: IoT Measurements - Drei Implementierungsansätze**](03-measurements-implementation) - Praktische Übung mit drei Performance-Optimierungsansätzen

---

**Kontakt:** kurse@htl-absolventen.at
