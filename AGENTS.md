# AGENTS.md

Agent-specific workflow and coding conventions for this repository. For project overview, build commands, architecture, and technical details see [CLAUDE.md](CLAUDE.md).

## Naming Conventions

| Type | Pattern | Example |
|------|---------|---------|
| ViewModel | `{Feature}ViewModel` | `CallLogViewModel` |
| Screen | `{Feature}Screen` | `ContactScreen`, `CallLogFullScreen` |
| Component | `Rivo{Name}` for library-style | `RivoAvatar`, `RivoListItem`, `RivoSwipeToActionBox` |
| Repository interface | `I{Name}Repository` | `IContactsRepository` |
| DAO | `{Entity}Dao` | `PrivateContactDao` |
| Entity | `{Name}Entity` | `PrivateContactEntity` |
| Manager/Util | `{Feature}Manager` or `{Feature}Utils` | `PreferenceManager`, `ContactUtils` |

## ViewModel Pattern

```kotlin
class FeatureViewModel(
    private val repo: IFeatureRepository,
    private val contentResolver: ContentResolver
) : ViewModel() {
    private val _state = MutableStateFlow<List<Item>>(emptyList())
    val state: StateFlow<List<Item>> = _state.asStateFlow()

    private var debounceFetchJob: Job? = null

    // ContentObserver with 250ms debounce for system data changes
    private val contentObserver = object : ContentObserver(Handler(Looper.getMainLooper())) {
        override fun onChange(selfChange: Boolean) {
            debounceFetchJob?.cancel()
            debounceFetchJob = viewModelScope.launch(Dispatchers.IO) {
                delay(250)
                fetch()
            }
        }
    }

    override fun onCleared() { /* unregister observers, cancel jobs */ }
}
```

Inject in Compose with `koinActivityViewModel<FeatureViewModel>()`. Singletons with `koinInject<PreferenceManager>()`.

## Adding a New Screen

1. Create `view/screen/FeatureScreen.kt`
2. Annotate with `@Destination<RootGraph>` + `@Composable`
3. First parameter: `navigator: DestinationsNavigator`
4. Optional params become type-safe navigation args
5. KSP auto-generates `FeatureScreenDestination` — navigate via:
   ```kotlin
   navigator.navigate(FeatureScreenDestination(arg = "value"))
   ```
6. Add strings to `res/values/strings.xml`
7. Opt-in annotations go before `@Destination`:
   ```kotlin
   @OptIn(ExperimentalMaterial3Api::class)
   @Destination<RootGraph>
   @Composable
   fun FeatureScreen(navigator: DestinationsNavigator) { }
   ```

## Adding a Database Table

1. Create `modal/db/FeatureEntity.kt` with `@Entity(tableName = "...")` and `@PrimaryKey`
2. Create `modal/db/FeatureDao.kt` as `@Dao` interface
3. Add entity to `@Database(entities = [...])` in `RivoDatabase`
4. Add abstract DAO getter in `RivoDatabase`
5. Bump `version` in `@Database`, create `MIGRATION_N_N+1` in companion
6. Add migration to `ALL_MIGRATIONS` array
7. Register DAO in `AppModule.kt`: `single { get<RivoDatabase>().featureDao() }`

Complex types stored as JSON strings in entities (e.g. `phoneNumbersJson`). Entities have `toContact()` / `fromContact()` conversion methods.

## Koin DI Registration

```kotlin
// AppModule.kt
val appModule = module {
    single { get<RivoDatabase>().featureDao() }                          // DAO
    single<IFeatureRepository> { FeatureRepository(androidContext(), get()) }  // Repository
    viewModel { FeatureViewModel(get(), androidContext().contentResolver) }     // ViewModel
}
```

## Flavor-Specific Code (Play vs FOSS)

Parallel source sets with identical package paths:
- `src/play/java/com/grinch/rivo4/` — real implementations (ads, billing)
- `src/foss/java/com/grinch/rivo4/` — no-op stubs with same API surface

FOSS stubs return defaults (`false`, empty flows). Gradle auto-selects. Same imports work in both.

## Repository Data Strategy

- **System contacts/call logs**: `ContentResolver` queries against system `ContentProvider`
- **Private contacts, call notes, reminders, trash**: Room database
- **Merged results**: Repository layer combines both sources

## String Resource Naming

```xml
<string name="permission_{feature}_title">...</string>
<string name="permission_{feature}_description">...</string>
<string name="dialog_{feature}_title">...</string>
<string name="action_{verb}">...</string>
<string name="content_desc_{element}">...</string>
<string name="toast_{action}_result">...</string>
<string name="label_{type}">...</string>
```

Format args: `%1$s` (string), `%1$d` (int). Plurals via `<plurals name="...">` with `quantity="one"` and `quantity="other"`.

## Compose Conventions

- **State hoisting**: State lives in screen, components receive values + callbacks
- **Derived state**: `derivedStateOf` for computed values that depend on scroll position etc.
- **Memoization**: `remember(key1, key2) { expensive() }` for display formatting
- **Config propagation**: `CompositionLocal` for deep theme/config (e.g. `LocalCallLogTileConfig`, `LocalRivoAvatarStyle`)
- **Config classes**: Mark with `@Immutable`
- **Coroutine scope**: `rememberCoroutineScope()` for UI-triggered async work

## Component Placement

- Reusable across screens → `view/components/`
- Used in one screen only → inline in screen file
- Library-style components prefix with `Rivo` (e.g. `RivoFrostedFAB`)

## KSP Notes

- Compose Destinations and Room both use KSP
- Generated code in `build/generated/ksp/`
- Windows workaround: KSP output dirs cleaned before each build to avoid `FileAlreadyExistsException`
- `ksp.incremental = false` in `gradle.properties`
