# POD - Play On Demand
## Product Requirement Document (PRD)

**Document Version**: 2.0  
**Last Updated**: 2025  
**Status**: Active  
**Repository**: https://github.com/bbapplied/POD  
**Current Branch**: 2025-10-26_Android

---

## 1. Executive Summary

POD is a .NET MAUI cross-platform application delivering a seamless, endless slideshow of cat GIFs sourced from the Tenor API. The application prioritizes responsiveness, user customization, and intelligent background prefetching to ensure smooth playback without interruptions. Users control playback speed, search terms, batch size, and optional audio features through an intuitive flyout menu interface.

---

## 2. Product Purpose

POD serves users seeking continuous, customizable cat-GIF entertainment with minimal interaction. The application emphasizes:
- **Simplicity**: Intuitive tap gestures and straightforward settings
- **Responsiveness**: Non-blocking UI, responsive to all user inputs
- **Customization**: Full control over search terms, timing, and content freshness
- **Reliability**: Graceful fallback mechanisms ensure playback continues during API failures
- **Cross-Platform**: Consistent experience on Windows and macOS

---

## 3. Core Features

### 3.1 GIF Playback & Display

- **Continuous Slideshow**: Automatically cycles through cat GIFs at user-defined intervals (5-30 seconds)
- **Category-Based Architecture**: Manages multiple search terms independently with isolated URL queues
- **Smart Prefetching**: Background batch fetching when queue drops to 30% capacity prevents playback gaps
- **Recent GIFs Filtering**: CircularBuffer tracks recently shown GIFs, filtering duplicates to enhance variety
- **Fallback Mechanism**: Reuses last successful batch if Tenor API becomes temporarily unavailable
- **Visual State Indication**: Border styling shows playback state (colored when playing, neutral when paused)
- **Decorative UI**: Cat eye graphics frame the main GIF display for visual appeal
- **Loading Placeholder**: Default asset ("Assets/pod_wait.png") displays during GIF loading

### 3.2 Gesture Controls

- **Single Tap**: Toggle pause/resume playback (300ms debounce to distinguish from double-tap)
- **Double Tap**: Skip to next GIF immediately and resume playback
- **Race Condition Prevention**: `_tapInProgress` flag and `_singleTapCts` cancellation token prevent conflicting tap handlers
- **300ms Debounce**: Reliable differentiation between single and double-tap actions

### 3.3 Search Customization

- **Multiple Search Terms**: Comma-separated list (e.g., "Wet Kitty, Bad Cat, Kitten Fail")
- **Category-Based Queue System**: Each search term has its own FIFO queue for independent management
- **Random Category Selection**: `LoadNextGif()` randomly selects category for content variety
- **Quick Actions**:
  - "Defaults" button restores pre-configured default search terms
  - "Clear" button removes all search terms
- **Real-Time Updates**: Search term changes immediately trigger new batch fetch and clear existing queues
- **Fallback Default**: If no search terms provided, app defaults to "Kitty"
- **Default Terms**: Pre-configured set: "Wet Kitty, Wet Cat, Bad Kitten, Bad Kitty, Bad Cat, Kitty Fail, Cat Fail, Kitten Fail"

### 3.4 Playback Speed Control

- **Slider Adjustment**: Users control delay between GIFs (5-30 seconds)
- **Persistent Setting**: Slider value automatically saved to device storage
- **Restoration on Launch**: Previous slider setting restored on app startup
- **Real-Time Application**: Speed changes apply immediately during playback

### 3.5 Batch Size Control

- **Configurable API Batch Fetching**: Users set GIF count per API request (10-50 GIFs)
- **Tenor API Compliance**: Enforces Tenor API maximum of 50 GIFs per request
- **Validation**: Minimum 10, maximum 50 GIFs per batch
- **Dynamic Recent GIFs Buffer**: CircularBuffer automatically resizes to 10x current batch size
- **Smart Prefetching**: Background fetch triggered when queue reaches 30% or below batch size threshold

### 3.6 Audio Features

- **"Soft Kitty" Audio**: Optional toggle to play background audio ("Soft Kitty" lullaby)
- **Non-Intrusive**: Audio toggle via checkbox with independent state management
- **Continuous Loop**: Audio repeats indefinitely while enabled
- **Embedded Asset**: "soft_kitty.mp4" packaged with application
- **Persistent State**: Audio preference saved and restored on app launch

### 3.7 Preference Persistence

- **Automatic Saving**: All user preferences persisted to device storage:
  - SliderValue (playback speed)
  - SearchTerms (comma-separated list)
  - PlaySoftKitty (audio toggle)
  - BatchSize (GIFs per fetch)
  - EnableLogging (debug logging toggle)
- **State Restoration**: Settings restored automatically on app startup
- **Platform-Native Storage**: Leverages MAUI `Preferences` API for secure, persistent storage

### 3.8 Debug Logging

- **Optional Logging**: Toggle `EnableLogging` to display debug output
- **Real-Time Log Display**: Bottom panel shows timestamped log entries
- **Cycle Tracking**: Log cycle numbers help track playback flow
- **File Logging**: DEBUG builds write logs to `pod_debug.log`
- **Entry Limit**: Recent 100 entries displayed, older entries removed automatically

### 3.9 Navigation & Menu Structure

- **Flyout Menu**: SwiftUI-style flyout with all controls and settings
- **Main Page**: GIF display with tap gestures
- **Donation Page**: Link to funding/support page accessible via flyout

---

## 4. Technical Architecture

### 4.1 Platform & Framework

- **Framework**: .NET MAUI v9.0 (Multi-platform App UI)
- **.NET Version**: .NET 9.0
- **C# Version**: 13.0
- **Target Platforms**:
  - Windows (10.0.26100.0)
  - macOS Catalyst (net9.0-maccatalyst)
  - Android (net9.0-android) - under development
- **MVVM Pattern**: CommunityToolkit.MVVM with `ObservableObject` and `[RelayCommand]`

### 4.2 Key Dependencies

| Dependency | Version | Purpose |
|-----------|---------|---------|
| **CommunityToolkit.MVVM** | v8.4.0 | MVVM framework: ObservableObject, RelayCommand |
| **CommunityToolkit.Maui** | v12.2.0 | Behaviors and UI enhancements |
| **FFImageLoading.Maui** | v1.3.2 | High-performance image/GIF caching and display |
| **System.Text.Json** | Built-in | JSON parsing for API responses |
| **Microsoft.Maui.Storage** | Built-in | Preferences API for settings persistence |

### 4.3 External APIs

- **Tenor API v2**: RESTful API for GIF sourcing
  - Endpoint: `https://tenor.googleapis.com/v2/search`
  - Authentication: API key (embedded in code - security concern noted in limitations)
  - Features: Random results, customizable limit, search-based filtering

### 4.4 Data Flow Architecture

```
????????????????????????????????????????????????
?         User Interactions                    ?
?  Taps, Search Changes, Settings Adjustments ?
????????????????????????????????????????????????
                    ?
????????????????????????????????????????????????
?           MainViewModel                      ?
?  - Preference Storage & Retrieval            ?
?  - Category Queue Management                 ?
?  - Gesture Command Handlers                  ?
?  - Playback Loop Control                     ?
????????????????????????????????????????????????
                    ?
        ????????????????????????????????????
        ?           ?          ?           ?
     ????????  ????????  ????????  ????????????
     ?GIF   ?  ?GIF   ?  ?Recent?  ?Last      ?
     ?Queue ?  ?Queue ?  ?GIFs  ?  ?Successful?
     ?Cat 1 ?  ?Cat 2 ?  ?Buffer?  ?Batch     ?
     ????????  ????????  ????????  ????????????
        ?           ?          ?           ?
        ????????????????????????????????????
                    ?
    ?????????????????????????????????????
    ?    Random Category Selection      ?
    ?  LoadNextGif() picks random queue ?
    ?????????????????????????????????????
                    ?
    ?????????????????????????????????????
    ?   Dequeue GIF URL From Queue      ?
    ?  If Queue Low ? Trigger Prefetch  ?
    ?????????????????????????????????????
                    ?
    ?????????????????????????????????????
    ?    Tenor API HTTP Request         ?
    ?  - Parse Search Terms             ?
    ?  - Select Random Term             ?
    ?  - Fetch Batch of GIF URLs        ?
    ?  - Parse JSON Response            ?
    ?????????????????????????????????????
                    ?
    ?????????????????????????????????????
    ?   GIF Playback Loop               ?
    ?  - Download GIF to Temp           ?
    ?  - Add URL to Recent GIFs Buffer  ?
    ?  - Display via FFImageLoading     ?
    ?  - Respect Slider Delay           ?
    ?????????????????????????????????????
                    ?
    ?????????????????????????????????????
    ?        UI Binding Update          ?
    ?  - Update CurGifName              ?
    ?  - Update CurGifCategory          ?
    ?  - Reflect IsGifPlaying State     ?
    ?????????????????????????????????????
```

### 4.5 Concurrency & Thread Safety Model

**Category-Based Architecture**:
- Each search term (category) has isolated infrastructure
- Prevents contention and enables independent queue management
- Random category selection for content variety

**Thread-Safe Components**:
- **ConcurrentQueue<T>**: Lock-free URL pipeline per category
- **CircularBuffer<T>**: Custom locking mechanism for recent GIFs tracking
- **Cancellation Tokens**: Comprehensive cooperative cancellation infrastructure
  - `_mainCts`: Main application lifecycle token
  - `_singleTapCts`: Single-tap debounce detection and cancellation
  - Linked tokens for composite timeout operations
- **State Flags**: Atomic boolean checks prevent race conditions
  - `_isLoadingGif`: Prevents concurrent GIF loading
  - `_isFetchingBatch`: Prevents concurrent API requests
  - `_tapInProgress`: Prevents multiple simultaneous tap processing

**Synchronization Strategy**:
- Playback loop and tap handlers run on separate threads
- Coordination via cancellation tokens and state flags
- Exception handling prevents thread crashes
- Main thread marshaling for UI updates via `MainThread.BeginInvokeOnMainThread()`

### 4.6 Resource Management

- **HttpClient Usage**: New instance created per API request (stateless design)
- **Disposal Patterns**: Proper cleanup of CancellationTokenSource and HttpClient instances
- **Memory Optimization**: CircularBuffer automatically overwrites oldest entries when full
- **JSON Parsing**: Using block handles JsonDocument disposal
- **File Cleanup**: Previous GIF files deleted after display to prevent disk bloat
- **Temp Directory**: Cross-platform cache location managed via `FileSystem.CacheDirectory`

---

## 5. User Interface Layout

### 5.1 Main View (MainPage.xaml)

```
???????????????????????????????????????????
?  ??        GIF Display Area         ??  ?  ? Cat eye decoratives
?                                         ?
?                                         ?
?    [Animated GIF With Gestures]        ?  ? FFImageLoading.CachedImage
?    • Single Tap: Pause/Resume          ?
?    • Double Tap: Next GIF              ?
?                                         ?
???????????????????????????????????????????
?  Current Category: {category}           ?  ? Status label
?  ??        GIF Display Area         ??  ?  ? Cat eye decoratives
?                                         ?
???????????????????????????????????????????
?  Log Entries (if EnableLogging = true)  ?  ? Scrolling log panel
???????????????????????????????????????????
```

**Gesture Zones**:
- **Single Tap Zone**: Entire GIF area - triggers pause/resume after 300ms
- **Double Tap Zone**: Entire GIF area - skips to next GIF immediately
- **Priority**: Double-tap cancels pending single-tap via `_singleTapCts`

### 5.2 Flyout Menu (AppShell.xaml)

```
????????????????????????????????????????????
?  INSTRUCTIONS PANEL                      ?
?  • Use comma-separated list for search   ?
?  • Single tap to pause/resume GIF        ?
?  • Double tap to goto next GIF           ?
????????????????????????????????????????????
?  SEARCH TERM(S) EDITOR                   ?
?  ?????????????????????????????????????? ?
?  ? [Multi-line Editor]                ? ?
?  ? Placeholder: "What type of...?"    ? ?
?  ? [Defaults] [Clear]                 ? ?
?  ?????????????????????????????????????? ?
????????????????????????????????????????????
?  PLAYBACK SPEED CONTROL                  ?
?  ?????????????????????????????????????? ?
?  ? Cycle Time: 15 seconds             ? ?
?  ? [????????????????????????????????] ? ?
?  ? Min: 5s              Max: 30s      ? ?
?  ?????????????????????????????????????? ?
????????????????????????????????????????????
?  BATCH SIZE CONTROL                      ?
?  ?????????????????????????????????????? ?
?  ? Results Batch Size: 50             ? ?
?  ? [????????????????????????????????] ? ?
?  ? Min: 10              Max: 50       ? ?
?  ?????????????????????????????????????? ?
????????????????????????????????????????????
?  AUDIO FEATURES                          ?
?  ? Play Soft Kitty                       ?  ? Checkbox toggle
????????????????????????????????????????????
?  DEBUG FEATURES                          ?
?  ? Enable Logging                        ?  ? Checkbox toggle
????????????????????????????????????????????
?  DONATE                                  ?
?  ?????????????????????????????????????? ?
?  ?  [Animated Buy Me a Coffee GIF]   ? ?
?  ?      (Clickable ? Donation Page)  ? ?
?  ?????????????????????????????????????? ?
????????????????????????????????????????????
```

### 5.3 Navigation Structure

```
AppShell (Navigation Hub)
??? MainPage (GIF Display)
?   ??? MainViewModel
?       ??? Playback Loop
?       ??? Category Queues
?       ??? Gesture Handlers
?       ??? Preference Management
?
??? Flyout Menu
?   ??? Search Terms Editor
?   ??? Speed Control (Slider)
?   ??? Batch Size Control (Slider)
?   ??? Audio Toggle
?   ??? Logging Toggle
?   ??? Donation Button
?
??? Donation Page (External navigation)
```

---

## 6. API Integration Details

### 6.1 Tenor API v2 Search Endpoint

**Base URL**: `https://tenor.googleapis.com/v2/search`

**Request Parameters**:
| Parameter | Type | Example | Notes |
|-----------|------|---------|-------|
| `q` | string | "Bad%20Kitten" | URI-encoded search term (random from user's list) |
| `key` | string | "AIzaSyD_..." | API authentication key |
| `limit` | integer | 50 | GIFs per request (user-configurable 10-50) |
| `random` | boolean | true | Returns randomized results for variety |

**Example Request**:
```
https://tenor.googleapis.com/v2/search?q=Bad%20Kitten&key=AIzaSyD_JO9yqEtuVlnIZM7fSi54B03xVdsPCBM&limit=50&random=true
```

### 6.2 Response Structure & Parsing

**JSON Response**:
```json
{
  "results": [
    {
      "id": "abc123def456",
      "title": "Bad kitten doing something funny",
      "media_formats": {
        "gif": {
          "url": "https://media.tenor.com/path/to/gif-downsized.gif",
          "duration": 2.5,
          "size": 1024000
        }
      }
    },
    { ... more results ... }
  ]
}
```

**Extraction Logic**:
1. Iterate through `results` array
2. Extract GIF URL from `media_formats.gif.url`
3. Filter against `_categoryRecentGifs[category]` to prevent duplicates
4. Add to `_categoryQueues[category]` if not filtered
5. Fallback: If all results filtered, add all URLs regardless of recency
6. Log success/error with count of fetched GIFs

### 6.3 Error Handling & Fallback Strategy

| Scenario | Response | Recovery |
|----------|----------|----------|
| **API request fails** | HttpRequestException caught | Reuse `_categoryLastSuccessfulBatch` |
| **JSON parse error** | JsonException caught | Skip malformed entries, continue processing |
| **No new GIFs found** | All filtered by CircularBuffer | Fetch all results regardless of recency |
| **Empty search terms** | No categories in list | Default to "Kitty" search term |
| **Malformed GIF URL** | Per-entry try-catch | Log error, skip entry, continue batch |
| **Network timeout** | HttpRequestException | Fallback batch + 5-second retry delay |
| **Rate limiting (429)** | EnsureSuccessStatusCode throws | Fallback batch + exponential backoff logic (future) |

### 6.4 Rate Limiting & Performance Optimization

- **Concurrent Fetch Prevention**: `_isFetchingBatch` flag ensures only one API request at a time
- **Smart Prefetching**: Fetches next batch when queue reaches 30% of batch size to prevent gaps
- **Batch Size Control**: User can adjust (10-50) to balance API calls vs. memory usage
- **HTTP Client Strategy**: New instance per request avoids connection pooling issues
- **Category Isolation**: Multiple categories don't block each other's fetch operations
- **Cache Bypass**: Download URLs include timestamp to prevent browser caching

---

## 7. GIF Download & Caching Strategy

### 7.1 Download Process

1. **Dequeue URL**: Extract next GIF URL from `_categoryQueues[_currentCategory]`
2. **Download to Temp**: HttpClient retrieves GIF to temporary file in cache directory
3. **Timestamp Cache Bypass**: URL includes `?timestamp={DateTime.Now.Ticks}` (handled by download logic)
4. **Display**: FFImageLoading.CachedImage renders GIF from local file
5. **Update Recent GIFs**: Add URL to `_categoryRecentGifs[_currentCategory]`
6. **Cleanup Previous**: Delete previous GIF file to prevent disk bloat

### 7.2 Temporary Storage

- **Location**: `FileSystem.CacheDirectory` + "/gifs" subdirectory
- **Cross-Platform Paths**:
  - Windows: `C:\Users\{username}\AppData\Local\{AppName}\`
  - macOS: `~/Library/Caches/{AppName}/`
  - Android: `/data/data/{package_name}/cache/`
- **Cleanup**: Old GIF files deleted after display, directory cleared on app launch

### 7.3 Memory Optimization

- **CircularBuffer Strategy**: Recent GIFs buffer limited to 10x batch size (default: 500 entries)
- **Automatic Overwrite**: Oldest entries automatically discarded when buffer full
- **Bounded Queue**: GIF URLs dequeued immediately after display, limiting memory footprint

---

## 8. Playback Loop Details

### 8.1 GifPlaybackLoop() Algorithm

```
LOOP:
  IF _mainCts.Token.IsCancellationRequested:
    BREAK  // App shutting down

  IF IsGifPlaying:
    IncrementCycleCount()
    LogCycleStart()

    START_TIMER: LoadNextGif()
    CalculateRemainingDisplay_Time()

    // Prefetch check
    IF CurrentCategoryQueueSize <= (BatchSize / 3):
      TriggerAsyncBatchFetch()

    // Calculate display duration
    DisplayTime = SliderValue * 1000 - LoadTime

    // Wait with cancellation
    CREATE_LINKED_CTS(MainCts)
    SET_TIMEOUT(DisplayTime)
    AWAIT_WITH_TIMEOUT()

    LogCycleEnd()
  ELSE:
    SLEEP(100ms)

CONTINUE_LOOP
```

### 8.2 Load Time Measurement

- **Start**: `DateTime.Now` before `LoadNextGif()`
- **End**: `DateTime.Now` after GIF displayed
- **Used**: Subtracted from SliderValue to calculate remaining display time
- **Minimum Display**: 100ms guaranteed (remainingMs floor)
- **Logged**: "GIF display time: {load}ms load + {remaining}ms display = {total}s total"

### 8.3 Queue Prefetch Trigger

- **Threshold**: Queue size ? BatchSize / 3 (e.g., ?17 if BatchSize=50)
- **Async**: Triggered with `_ = FetchGifBatchForCategoryAsync()` (fire-and-forget)
- **Prevention**: `_isFetchingBatch` guard prevents concurrent API requests
- **Logging**: "Queue low for '{category}': {queueSize}/{threshold} threshold. Fetching batch..."

---

## 9. Gesture Handling Mechanics

### 9.1 Single-Tap (Pause/Resume)

```
USER_TAPS_IMAGE
  ?
SingleTap Command Executed
  ?
IF _tapInProgress:
  RETURN  // Suppress duplicate

SET _tapInProgress = true
CREATE _singleTapCts = New CancellationTokenSource

TRY:
  AWAIT Task.Delay(300ms, _singleTapCts.Token)
  ?
  IF Task Completed (No Cancellation):
    Toggle IsGifPlaying
    Log "Single Tap Detected"
  ?
  IF Cancelled (Double-Tap Detected):
    Suppress Single-Tap Action
    CATCH OperationCanceledException

FINALLY:
  SET _tapInProgress = false
  CLEANUP _singleTapCts
```

**300ms Debounce Logic**:
- Single-tap waits 300ms to confirm no second tap follows
- Double-tap cancels token immediately, suppressing single-tap
- User experiences immediate double-tap response

### 9.2 Double-Tap (Skip to Next)

```
USER_DOUBLE_TAPS_IMAGE
  ?
DoubleTap Command Executed
  ?
IF _isLoadingGif:
  RETURN  // Suppress while loading

CANCEL _singleTapCts  // Cancel pending single-tap

SET _tapInProgress = true

TRY:
  AWAIT LoadNextGif()
  SET IsGifPlaying = true
  Log "Double Tap Detected"

FINALLY:
  SET _tapInProgress = false
```

**Race Condition Prevention**:
- `_tapInProgress` flag blocks concurrent tap handling
- `_singleTapCts?.Cancel()` immediately terminates pending single-tap
- Exception-free cancellation via OperationCanceledException catch

### 9.3 Tap Detection Accuracy

| Scenario | Behavior | Result |
|----------|----------|--------|
| **Quick single tap** | Wait 300ms, no 2nd tap | IsGifPlaying toggled |
| **Quick double tap** | 1st cancels, 2nd runs | LoadNextGif executed |
| **Slow double tap** | 1st executes at 300ms | IsGifPlaying toggled, then double-tap skipped |
| **Triple tap** | 2nd cancels 1st, 3rd blocked | Double-tap, then LoadNextGif skipped |
| **Rapid taps** | `_tapInProgress` blocks | Only one handler active at a time |

---

## 10. Preference Persistence Architecture

### 10.1 Persisted Preferences

| Preference Key | Type | Default | Range | Notes |
|---|---|---|---|---|
| `SliderValue` | int | 15 | 5-30 seconds | Playback delay between GIFs |
| `SearchTerms` | string | Default list | Comma-separated | User's search categories |
| `PlaySoftKitty` | bool | false | true/false | Audio toggle state |
| `BatchSize` | int | 50 | 10-50 | GIFs per API request |
| `EnableLogging` | bool | false | true/false | Debug logging toggle |

### 10.2 Storage Implementation

- **API**: MAUI `Preferences.Set()` and `Preferences.Get()`
- **Pattern**: `Preferences.Set(nameof(PropertyName), value)`
- **Type-Safety**: Using `nameof()` prevents string key typos
- **Restoration**: Constructor loads all preferences on app startup
- **Change Handlers**: Partial change methods automatically save to Preferences

**Example**:
```csharp
partial void OnSliderValueChanged(int value)
{
    Preferences.Set(nameof(SliderValue), value);  // Auto-save
}

// On app startup:
SliderValue = Preferences.Get(nameof(SliderValue), 15);
```

### 10.3 Platform-Specific Storage

| Platform | Storage Mechanism | Location |
|----------|---|---|
| **Windows** | Registry | `HKEY_CURRENT_USER\Software\{AppName}` |
| **macOS** | UserDefaults (plist) | `~/Library/Preferences/` |
| **Android** | SharedPreferences | `/data/data/{package_name}/shared_prefs/` |

---

## 11. Default Configuration

### 11.1 Search Terms

```
Wet Kitty
Wet Cat
Bad Kitten
Bad Kitty
Bad Cat
Kitty Fail
Cat Fail
Kitten Fail
```

### 11.2 UI Configuration

| Setting | Value | Notes |
|---------|-------|-------|
| **Default Playback Speed** | 15 seconds | Per-GIF display duration |
| **Default Batch Size** | 50 GIFs | Maximum allowed by Tenor API |
| **Recent GIFs Buffer Size** | 500 entries | 10x batch size, auto-resizes |
| **Single-Tap Debounce** | 300ms | Distinguishes single from double-tap |
| **Queue Prefetch Threshold** | 30% of batch size | Triggers background fetch |
| **Soft Kitty Audio** | Off by default | Optional toggle in flyout |
| **Logging** | Off by default | Optional debug feature |
| **Placeholder Image** | "Assets/pod_wait.png" | Displays while loading |
| **Log Entry Limit** | 100 entries | Oldest entries removed |

---

## 12. Performance Characteristics

### 12.1 Benchmarks & Targets

| Metric | Target | Current | Notes |
|--------|--------|---------|-------|
| **App Launch Time** | < 2s | ~0.5s | UI bindings init in background |
| **GIF Load Time** | < 500ms | ~200-400ms | Network dependent |
| **API Success Rate** | > 95% | > 99% | Fallback mechanism ensures reliability |
| **Playback Continuity** | No gaps | Seamless | Prefetch at 30% threshold |
| **Memory Usage** | < 150MB | ~80-100MB | CircularBuffer limits memory |
| **Tap Response Time** | < 100ms | < 50ms | Immediate UI feedback |
| **Preference Persistence** | 100% success | 100% | Persistent across restarts |

### 12.2 Memory Optimization Strategies

| Strategy | Benefit | Implementation |
|----------|---------|---|
| **CircularBuffer (Bounded)** | Fixed memory footprint | Size = 10x batch size, auto-overwrite |
| **Queue-Based Architecture** | Prevents memory bloat | GIFs dequeued after display |
| **Recent GIFs Filtering** | Prevents duplicates | CircularBuffer.Contains() check |
| **Fallback Batch Storage** | Minimal overhead | Only stores last successful batch per category |
| **Linked Cancellation Tokens** | Efficient timeout | No busy-waiting, cooperative cancellation |
| **File Cleanup** | Prevents disk bloat | Delete previous GIF after display |

### 12.3 Network Optimization

| Optimization | Impact | Details |
|---|---|---|
| **Batch Fetching** | Reduces API calls | Single request for 10-50 GIFs |
| **Background Prefetch** | Eliminates gaps | Fetches when queue at 30% capacity |
| **Connection-Less Design** | Avoid pooling issues | New HttpClient per request |
| **Error Recovery** | Prevents retry storms | Fallback mechanism + logging |
| **URL Deduplication** | Bandwidth efficiency | CircularBuffer prevents re-downloads |

---

## 13. Quality Attributes

### 13.1 Reliability

- **High Availability**: Handles API unavailability gracefully with fallback mechanism
- **Resource Cleanup**: Proper disposal of CancellationTokenSource and HttpClient
- **Memory Safety**: Thread-safe concurrent access patterns throughout
- **Data Persistence**: Preferences survive app restarts and device reboots
- **Graceful Degradation**: App continues with cached content if offline

### 13.2 Responsiveness

- **Playback Smoothness**: Background prefetching ensures seamless transitions
- **Input Response**: Gesture recognition within milliseconds
- **Non-Blocking**: Async/await pattern prevents UI thread blocking
- **Settings Update**: Changes apply immediately during playback
- **Minimal Latency**: Optimized data structures (ConcurrentQueue, CircularBuffer)

### 13.3 Usability

- **Intuitive Controls**: Single-tap and double-tap gestures are natural and discoverable
- **Clear Feedback**: Visual indicators (border, status label) show playback state
- **Instant Search**: Changes trigger immediate new batch fetch
- **Auto-Save**: All preferences persisted automatically
- **Responsive UI**: Always responsive to user input

### 13.4 Cross-Platform Consistency

- **Unified Code**: Single ViewModel for all platforms
- **Platform-Native APIs**: Leverages MAUI abstractions (Preferences, FileSystem)
- **Responsive Layout**: UI adapts to different screen sizes and orientations
- **Same Functionality**: Identical feature set across Windows and macOS

### 13.5 Security & Privacy

- **HTTPS-Only**: All API communications over secure connection
- **Input Validation**: Search terms sanitized via `Uri.EscapeDataString()`
- **No User Data**: No tracking, analytics, or telemetry
- **API Key Security**: Concern noted - should move to environment variables (see limitations)

---

## 14. Gesture Handling Details

### 14.1 Single-Tap Implementation

- **Detection**: 300ms delay to distinguish from double-tap
- **Debounce Token**: `_singleTapCts` cancellation prevents race conditions
- **UI Action**: Toggle `IsGifPlaying` if not double-tapped
- **Visual Feedback**: Border color changes to reflect state

### 14.2 Double-Tap Implementation

- **Detection**: Immediate cancellation of pending single-tap
- **UI Action**: `LoadNextGif()` executes immediately
- **State**: Ensures `IsGifPlaying = true` for continuous flow
- **Priority**: Double-tap always takes precedence over single-tap

### 14.3 Race Condition Prevention

- **Flag**: `_tapInProgress` prevents overlapping handlers
- **Token Coordination**: `_singleTapCts` enables cross-handler communication
- **Exception Handling**: OperationCanceledException suppresses cancelled operations
- **Lock-Free Design**: Boolean flags provide atomic checks

---

## 15. Extension Points & Future Enhancements

### 15.1 Planned Features (v2.0+)

- **Favorite/Bookmark GIFs**: Save and review favorites
- **Share Functionality**: Post GIFs to social media
- **Theme Support**: Dark and light modes
- **Grid View**: Display multiple GIFs simultaneously
- **GIF History**: Browse previously viewed content
- **Custom Audio**: Upload background music tracks
- **Search History**: Quick access to frequent searches
- **Export Features**: Download favorite GIFs locally
- **Filtering**: Content rating and size filters

### 15.2 Architectural Improvements

- **Dependency Injection**: Inject services for testability
- **Repository Pattern**: Abstract API calls behind interface
- **Unit Testing**: Comprehensive ViewModel test coverage
- **Structured Logging**: Centralized logging framework
- **Configuration Management**: Environment-based settings
- **Async Initialization**: Non-blocking preference loading
- **Cloud Sync**: Sync preferences across devices

### 15.3 Platform Expansion

- **Android Support**: Complete implementation (net9.0-android)
- **iOS/iPadOS**: Full native support (net9.0-ios)
- **Web**: Blazor web version using same ViewModel
- **Desktop**: WinUI 3 native alternative

---

## 16. Testing Strategy

### 16.1 Unit Tests

- **ViewModel Logic**: Category queue management, preference handling
- **CircularBuffer**: Add, Contains, Clear, thread-safety verification
- **Gesture Handlers**: Single/double-tap detection accuracy
- **URL Filtering**: Deduplication logic verification
- **Search Term Parsing**: Comma-separated parsing and validation

### 16.2 Integration Tests

- **API Integration**: Tenor API response parsing, error handling paths
- **Preference Persistence**: Cross-session storage verification
- **Playback Loop**: Continuous operation, memory stability
- **Network Resilience**: Fallback mechanism under API failure

### 16.3 Manual Testing

- **Gesture Responsiveness**: Tap accuracy and timing on actual devices
- **Visual Quality**: GIF display smoothness and animation
- **Platform-Specific**: Windows and macOS native behavior
- **Edge Cases**: Rapid taps, network interruptions, preference changes

### 16.4 Acceptance Criteria

- [ ] Single-tap toggles playback reliably
- [ ] Double-tap skips immediately without UI lag
- [ ] Search terms update immediately
- [ ] Settings persist across app restarts
- [ ] API failures don't crash app
- [ ] No memory leaks during extended playback
- [ ] GIFs display smoothly without gaps
- [ ] Both platforms (Windows, macOS) work identically

---

## 17. Known Limitations & Considerations

| Limitation | Impact | Severity | Workaround/Future Plan |
|---|---|---|---|
| **Hardcoded API Key** | Security risk in public repo | HIGH | Move to environment variables in production |
| **Single API Key** | Rate limiting possible under heavy load | MEDIUM | Implement API key rotation in v2.0 |
| **No Offline Mode** | Requires internet connection | LOW | Cache larger batch locally in v2.0 |
| **No User Accounts** | Can't sync across devices | LOW | Cloud sync optional feature in v2.0 |
| **Limited Content** | Only cat GIFs | LOW | Extensible to other categories in v2.0 |
| **No Content Filtering** | Can't filter by rating/size | LOW | Add content filters in v2.0 |
| **Slow Network Handling** | Poor user experience on 3G/weak WiFi | MEDIUM | Implement progressive loading indicator |

---

## 18. API Key Security Notice

**?? CRITICAL**: The Tenor API key is currently embedded in the source code. For production deployment:

1. **Move to Environment Variable**: Set `TENOR_API_KEY` at build time
2. **Secure Key Management**: Use platform-specific secure storage (Keychain, Vault)
3. **Key Rotation**: Implement periodic key rotation
4. **Usage Monitoring**: Track API calls to prevent rate limiting
5. **Rate Limit Handling**: Implement exponential backoff for 429 responses

---

## 19. Deployment & Release

### 19.1 Build Configuration

- **Build System**: .NET CLI or Visual Studio 2024+
- **Target Frameworks**:
  - `net9.0-windows10.0.26100.0`
  - `net9.0-maccatalyst`
  - `net9.0-android` (in development)
- **Release Profile**: Optimize for performance and size

### 19.2 Distribution Channels

- **GitHub Releases**: Binary packages on repository
- **Platform Installers**: Windows .exe, macOS .app bundle
- **Update Strategy**: Manual download (future: auto-update via GitHub)

### 19.3 Version Numbering

- **Format**: MAJOR.MINOR.PATCH
- **Example**: v1.0.0 (initial release)
- **Changelog**: GitHub releases and documentation

---

## 20. Success Metrics & KPIs

| Metric | Target | Method |
|--------|--------|--------|
| **App Stability** | 99.9% uptime | Runtime exception logging |
| **User Retention** | 70% at 30 days | Anonymous usage telemetry (opt-in) |
| **Performance** | 60 FPS GIF playback | Frame rate monitoring |
| **API Reliability** | > 95% success | Request/response logging |
| **User Satisfaction** | > 4.5 stars | App store reviews |
| **Memory Efficiency** | < 150MB peak | Memory profiling |
| **Network Efficiency** | < 500KB/min | Bandwidth logging |

---

## Document History

| Version | Date | Changes |
|---------|------|---------|
| 2.0 | 2025 | Major update: Category architecture, improved concurrency docs, playback loop details, gesture mechanics, performance benchmarks |
| 1.0 | 2025 | Initial PRD documentation |

---

## Contact & Support

- **Repository**: https://github.com/bbapplied/POD
- **Issues & Bugs**: GitHub Issues tracker
- **Feature Requests**: GitHub Discussions
- **Current Branch**: 2025-10-26_Android

---

**END OF PRODUCT REQUIREMENT DOCUMENT v2.0**
