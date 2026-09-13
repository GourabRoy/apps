# Journals App - High-Level Design (HLD)

The application implements the **Model-View-ViewModel (MVVM)** architectural pattern combined with **Clean Architecture** principles. This ensures a unidirectional data flow (UDF), separates business logic from UI rendering, and guarantees offline-first data persistence.

## Core System Components

1. **UI Layer (Jetpack Compose):** A declarative UI system that consumes state via `StateFlow` and emits user intent events to the ViewModel. It handles navigation and rendering.
2. **Presentation Layer (ViewModel):** Retains UI state across configuration changes, manages background coroutine scopes (`viewModelScope`), and translates raw repository data into UI-specific data models.
3. **Repository Layer:** Acts as the single source of truth. It abstracts the data sources (Room database) and handles complex data transformations (e.g., mapping raw DB rows to summary objects).
4. **Data Layer (Room SQLite):** Local persistence mechanism utilizing DAOs (Data Access Objects) to perform CRUD operations with asynchronous `Flow` streams for real-time reactivity.
5. **Background Task Layer (WorkManager):** An OS-managed job scheduler responsible for executing the 2-hour notification loop independent of the app's lifecycle, utilizing `CoroutineWorker` for non-blocking execution.

## Component Interaction Diagram

```mermaid
graph TD
    %% UI Layer
    subgraph UI[UI Layer - Jetpack Compose]
        HomeScreen[Home / Track List]
        NoteDialog[Quick Add Note Sheet]
        SummaryScreen[Yearly Summary Dashboard]
    end

    %% Presentation Layer
    subgraph ViewModels[Presentation Layer]
        NoteVM[NoteViewModel]
        SummaryVM[SummaryViewModel]
    end

    %% Repository Layer
    subgraph Repositories[Repository Layer]
        NoteRepo[NoteRepositoryImpl]
    end

    %% Data Layer
    subgraph Data[Data Layer - Room]
        AppDB[(AppDatabase SQLite)]
        TrackDao[TrackDao]
        NoteDao[NoteDao]
    end

    %% Background Layer
    subgraph Background[Background Task Layer]
        Worker[PromptWorker]
        NotifMgr[NotificationManager]
    end

    %% UI to ViewModel (Intents)
    HomeScreen -->|Get Tracks, Add Track| NoteVM
    NoteDialog -->|Save Note, Auto-timestamp| NoteVM
    SummaryScreen -->|Fetch Year| SummaryVM

    %% ViewModel to Repository (Requests & Flows)
    NoteVM <-->|StateFlow<List<Track>>| NoteRepo
    SummaryVM <-->|Suspend Fetch| NoteRepo

    %% Repository to Data (DAO interactions)
    NoteRepo <-->|Flow Queries| TrackDao
    NoteRepo <-->|Flow Queries| NoteDao
    TrackDao <--> AppDB
    NoteDao <--> AppDB

    %% Background interactions
    AppDB -.->|App Start / Re-schedule| Worker
    Worker -->|Check Time 8-22h| NotifMgr
    NotifMgr -.->|Deep Link PendingIntent| NoteDialog
