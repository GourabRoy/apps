# Journals App - Low-Level Design (LLD)

## 1. Data Layer Implementation (Room)

**Entity Definitions:**
Defines the SQLite tables. The `Note` entity relies on a Foreign Key to `Track` with `CASCADE` delete to prevent orphaned notes.

```kotlin
@Entity(tableName = "tracks")
data class TrackEntity(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val name: String,
    val createdAt: Long = System.currentTimeMillis()
)

@Entity(
    tableName = "notes",
    foreignKeys = [
        ForeignKey(
            entity = TrackEntity::class,
            parentColumns = ["id"],
            childColumns = ["trackId"],
            onDelete = ForeignKey.CASCADE
        )
    ],
    indices = [Index(value = ["trackId"])]
)
data class NoteEntity(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val trackId: Int,
    val content: String,
    val timestamp: Long = System.currentTimeMillis()
)
```

### 2.1. Data Access Object (DAO):
Reactive queries utilizing Kotlin Flow. The yearly summary uses SQLite strftime equivalent time bounding.

```kotlin
@Dao
interface JournalDao {
    @Query("SELECT * FROM tracks ORDER BY createdAt ASC")
    fun observeAllTracks(): Flow<List<TrackEntity>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertNote(note: NoteEntity)

    @Query("""
        SELECT t.name as trackName, COUNT(n.id) as noteCount 
        FROM tracks t 
        LEFT JOIN notes n ON t.id = n.trackId 
        WHERE n.timestamp BETWEEN :startOfYear AND :endOfYear 
        GROUP BY t.id
    """)
    suspend fun getYearlySummary(startOfYear: Long, endOfYear: Long): List<TrackSummaryView>
}
```

## 2. Repository Layer Implementation
Abstracts the Room implementation details, exposing data streams and suspending functions for one-shot operations.

```kotlin
class NoteRepositoryImpl(private val dao: JournalDao) : NoteRepository {
    override fun getAllTracks(): Flow<List<TrackEntity>> = dao.observeAllTracks()

    override suspend fun addNote(trackId: Int, content: String) {
        val newNote = NoteEntity(trackId = trackId, content = content)
        dao.insertNote(newNote)
    }

    override suspend fun getSummaryForYear(year: Int): List<TrackSummaryView> {
        val cal = Calendar.getInstance()
        cal.set(year, 0, 1, 0, 0, 0)
        val start = cal.timeInMillis
        cal.set(year, 11, 31, 23, 59, 59)
        val end = cal.timeInMillis
        return dao.getYearlySummary(start, end)
    }
}
```

3. Presentation Layer (ViewModel)
Implements Unidirectional Data Flow using StateFlow. viewModelScope ensures queries are cancelled if the ViewModel is cleared.

```kotlin
class NoteViewModel(private val repository: NoteRepository) : ViewModel() {
    private val _uiState = MutableStateFlow(NoteUiState())
    val uiState: StateFlow<NoteUiState> = _uiState.asStateFlow()

    init {
        viewModelScope.launch {
            repository.getAllTracks().collect { tracks ->
                _uiState.update { it.copy(tracks = tracks) }
            }
        }
    }

    fun saveNote(trackId: Int, content: String) {
        viewModelScope.launch(Dispatchers.IO) {
            repository.addNote(trackId, content)
        }
    }
}

data class NoteUiState(
    val tracks: List<TrackEntity> = emptyList(),
    val isLoading: Boolean = false
)
```

## 4. Background Task Layer (WorkManager)
The PromptWorker wakes up every 2 hours, validates the time against active hours (8 AM - 10 PM), and dispatches a system notification if the condition is met.

```kotlin
class PromptWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        val currentHour = Calendar.getInstance().get(Calendar.HOUR_OF_DAY)
        
        if (currentHour in 8..22) {
            triggerNotification(applicationContext)
        }
        return Result.success()
    }

    private fun triggerNotification(context: Context) {
        val intent = Intent(
            Intent.ACTION_VIEW,
            Uri.parse("journalsapp://add_note")
        )
        val pendingIntent = PendingIntent.getActivity(
            context, 0, intent, PendingIntent.FLAG_IMMUTABLE
        )

        val notification = NotificationCompat.Builder(context, "JOURNAL_CHANNEL_ID")
            .setSmallIcon(R.drawable.ic_note)
            .setContentTitle("Capture your thoughts")
            .setContentText("Anything you want to add to your journal?")
            .setContentIntent(pendingIntent)
            .setAutoCancel(true)
            .build()

        NotificationManagerCompat.from(context).notify(101, notification)
    }
}
```
