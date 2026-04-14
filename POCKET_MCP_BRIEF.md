# PocketMCP — Your Phone as an MCP Server

## What Is This

A Kotlin Multiplatform mobile app where you chat with Claude, and Claude can control your phone through tool use (function calling). Think of it as MCP, but the server is the device in your pocket.

You talk to Claude. Claude decides it needs to turn on your flashlight, check your battery, take a photo, or get your GPS location. It sends a tool call. Your app executes it on the device. The result goes back to Claude. Claude responds naturally.

**This is not a demo. This is what Anthropic won't build — and that's the point.**

---

## Why This Project Exists

1. Proves KMP + CMP in a real cross-platform app (Android + iOS from shared code)
2. Proves Claude API tool use / function calling in production
3. Proves clean architecture under real complexity (permissions, platform APIs, async flows)
4. Proves full-stack KMP: mobile app + Ktor backend proxy
5. Lives on your GitHub profile as the evidence behind every claim on your README

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    MOBILE APP                        │
│                (Compose Multiplatform)               │
│                                                     │
│  ┌─────────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ Presentation │  │  Domain  │  │     Data      │  │
│  │              │  │          │  │               │  │
│  │ ChatScreen   │  │ UseCases │  │ ChatRepo      │  │
│  │ ChatViewModel│  │ Models   │  │ ToolExecutor  │  │
│  │ ToolResultUI │  │ ToolDefs │  │ ApiClient     │  │
│  └──────┬───────┘  └────┬─────┘  └───────┬───────┘  │
│         │               │                │          │
│         └───────────────┴────────────────┘          │
│                         │                           │
│              ┌──────────┴──────────┐                │
│              │   Platform Tools    │                │
│              │   (expect/actual)   │                │
│              │                     │                │
│              │ FlashlightTool      │                │
│              │ BatteryTool         │                │
│              │ VibrateTool         │                │
│              │ ClipboardTool       │                │
│              │ CameraTool          │                │
│              │ LocationTool        │                │
│              │ DeviceInfoTool      │                │
│              │ NotificationTool    │                │
│              └─────────────────────┘                │
└──────────────────────┬──────────────────────────────┘
                       │ HTTPS
                       ▼
┌─────────────────────────────────────────────────────┐
│                  BACKEND PROXY                       │
│                    (Ktor Server)                     │
│                                                     │
│  POST /chat                                         │
│  - Receives messages + tool results from app        │
│  - Forwards to Claude API (API key stays here)      │
│  - Returns Claude's response (text or tool_use)     │
│  - Streams responses via SSE                        │
│                                                     │
│  Claude API key lives HERE, never on the device     │
└─────────────────────────────────────────────────────┘
                       │ HTTPS
                       ▼
              ┌─────────────────┐
              │   Claude API    │
              │   (Anthropic)   │
              └─────────────────┘
```

---

## Module Structure

```
pocket-mcp/
├── shared/                          # KMP shared module
│   ├── src/
│   │   ├── commonMain/
│   │   │   ├── domain/
│   │   │   │   ├── model/
│   │   │   │   │   ├── Message.kt           # Chat message (user/assistant/tool_result)
│   │   │   │   │   ├── ToolCall.kt          # Tool call from Claude (name, id, input)
│   │   │   │   │   ├── ToolResult.kt        # Result after executing a tool
│   │   │   │   │   └── ToolDefinition.kt    # Tool schema (name, description, parameters)
│   │   │   │   └── usecase/
│   │   │   │       ├── SendMessageUseCase.kt
│   │   │   │       └── ExecuteToolUseCase.kt
│   │   │   │
│   │   │   ├── data/
│   │   │   │   ├── repository/
│   │   │   │   │   └── ChatRepository.kt    # Manages conversation state + API calls
│   │   │   │   ├── remote/
│   │   │   │   │   └── ProxyApiClient.kt    # Ktor client → your backend proxy
│   │   │   │   └── tools/
│   │   │   │       ├── ToolRegistry.kt      # Maps tool names → implementations
│   │   │   │       ├── ToolExecutor.kt      # Dispatches tool calls to the right impl
│   │   │   │       └── tools/
│   │   │   │           ├── FlashlightTool.kt    # expect/actual
│   │   │   │           ├── BatteryTool.kt       # expect/actual
│   │   │   │           ├── VibrateTool.kt       # expect/actual
│   │   │   │           ├── ClipboardTool.kt     # expect/actual
│   │   │   │           ├── CameraTool.kt        # expect/actual
│   │   │   │           ├── LocationTool.kt      # expect/actual
│   │   │   │           ├── DeviceInfoTool.kt    # expect/actual
│   │   │   │           └── NotificationTool.kt  # expect/actual
│   │   │   │
│   │   │   └── presentation/
│   │   │       ├── ChatViewModel.kt
│   │   │       └── ChatUiState.kt
│   │   │
│   │   ├── androidMain/
│   │   │   └── tools/                  # Android implementations
│   │   │       ├── FlashlightTool.android.kt    # CameraManager
│   │   │       ├── BatteryTool.android.kt       # BatteryManager
│   │   │       ├── VibrateTool.android.kt       # VibratorManager
│   │   │       ├── ClipboardTool.android.kt     # ClipboardManager
│   │   │       ├── CameraTool.android.kt        # CameraX
│   │   │       ├── LocationTool.android.kt      # FusedLocationProvider
│   │   │       ├── DeviceInfoTool.android.kt    # Build, Settings
│   │   │       └── NotificationTool.android.kt  # NotificationManager
│   │   │
│   │   └── iosMain/
│   │       └── tools/                  # iOS implementations
│   │           ├── FlashlightTool.ios.kt        # AVCaptureDevice
│   │           ├── BatteryTool.ios.kt           # UIDevice
│   │           ├── VibrateTool.ios.kt           # UIImpactFeedbackGenerator
│   │           ├── ClipboardTool.ios.kt         # UIPasteboard
│   │           ├── CameraTool.ios.kt            # AVFoundation
│   │           ├── LocationTool.ios.kt          # CLLocationManager
│   │           ├── DeviceInfoTool.ios.kt        # UIDevice, ProcessInfo
│   │           └── NotificationTool.ios.kt      # UNUserNotificationCenter
│   │
│   └── build.gradle.kts
│
├── composeApp/                      # Compose Multiplatform UI
│   ├── src/
│   │   ├── commonMain/
│   │   │   └── ui/
│   │   │       ├── App.kt
│   │   │       ├── ChatScreen.kt           # Main chat interface
│   │   │       ├── MessageBubble.kt        # User/assistant message display
│   │   │       ├── ToolCallCard.kt         # Shows tool execution + result
│   │   │       ├── PermissionDialog.kt     # Explains why a permission is needed
│   │   │       └── theme/
│   │   │           └── Theme.kt
│   │   ├── androidMain/
│   │   └── iosMain/
│   └── build.gradle.kts
│
├── server/                          # Ktor backend proxy
│   ├── src/main/kotlin/
│   │   ├── Application.kt
│   │   ├── routes/
│   │   │   └── ChatRoutes.kt       # POST /chat endpoint
│   │   ├── service/
│   │   │   └── ClaudeService.kt    # Anthropic API client with tool definitions
│   │   └── model/
│   │       ├── ChatRequest.kt      # Messages from app
│   │       └── ChatResponse.kt     # Response to app (text or tool_use)
│   └── build.gradle.kts
│
├── build.gradle.kts                 # Root build file
├── settings.gradle.kts
├── gradle.properties
├── .github/
│   └── workflows/
│       └── ci.yml                   # Build + test on every push
└── README.md
```

---

## Tool Definitions

Each tool needs: a shared interface, a Claude tool schema (JSON), and platform implementations.

### Shared Tool Interface (commonMain)

```kotlin
interface DeviceTool {
    val name: String
    val description: String
    val parameters: JsonObject  // JSON Schema for Claude

    suspend fun execute(input: JsonObject): ToolResult
}
```

### Tool Registry (commonMain)

```kotlin
class ToolRegistry(private val tools: List<DeviceTool>) {

    fun getToolDefinitions(): List<ToolDefinition> =
        tools.map { ToolDefinition(it.name, it.description, it.parameters) }

    fun findTool(name: String): DeviceTool? =
        tools.firstOrNull { it.name == name }
}
```

### Tool List with Platform APIs

| Tool | Claude sees | What it does | Android API | iOS API |
|------|-----------|--------------|-------------|---------|
| `flashlight_toggle` | "Toggle the device flashlight on or off" | Turns torch on/off | `CameraManager.setTorchMode()` | `AVCaptureDevice.torchMode` |
| `battery_status` | "Get current battery level and charging state" | Returns battery % + charging | `BatteryManager` | `UIDevice.current.batteryLevel` |
| `vibrate` | "Vibrate the device with specified pattern" | Short/long/pattern vibration | `VibratorManager` | `UIImpactFeedbackGenerator` |
| `clipboard_write` | "Copy text to the device clipboard" | Writes to clipboard | `ClipboardManager` | `UIPasteboard.general` |
| `clipboard_read` | "Read current text from the device clipboard" | Reads clipboard | `ClipboardManager` | `UIPasteboard.general` |
| `take_photo` | "Take a photo using the device camera" | Opens camera, captures image | `CameraX` | `AVCaptureSession` |
| `get_location` | "Get the device's current GPS coordinates" | Returns lat/lon + accuracy | `FusedLocationProviderClient` | `CLLocationManager` |
| `device_info` | "Get device model, OS version, and system info" | Returns device metadata | `Build.*`, `Settings` | `UIDevice`, `ProcessInfo` |
| `set_reminder` | "Set a local notification reminder" | Schedules a notification | `NotificationManager` + `AlarmManager` | `UNUserNotificationCenter` |

---

## Claude API Integration

### How Tool Use Works (the loop)

```
1. User types "Turn on my flashlight"

2. App sends to YOUR backend proxy:
   POST /chat
   {
     "messages": [{"role": "user", "content": "Turn on my flashlight"}],
     "tools": [... all tool definitions ...]
   }

3. Backend forwards to Claude API (with your API key)

4. Claude responds with a tool_use block:
   {
     "role": "assistant",
     "content": [
       {
         "type": "tool_use",
         "id": "toolu_abc123",
         "name": "flashlight_toggle",
         "input": {"state": "on"}
       }
     ]
   }

5. Backend returns this to the app

6. App sees tool_use → finds "flashlight_toggle" in ToolRegistry
   → calls FlashlightTool.execute({"state": "on"})
   → flashlight turns on
   → returns ToolResult(success=true, message="Flashlight turned on")

7. App sends the result back to the backend:
   POST /chat
   {
     "messages": [
       {"role": "user", "content": "Turn on my flashlight"},
       {"role": "assistant", "content": [{"type": "tool_use", ...}]},
       {"role": "user", "content": [
         {"type": "tool_result", "tool_use_id": "toolu_abc123",
          "content": "Flashlight turned on successfully"}
       ]}
     ]
   }

8. Claude receives the result and responds naturally:
   "Done! I've turned on your flashlight. Let me know when
    you want me to turn it off."

9. App displays Claude's response in the chat.
```

### Backend Proxy — Claude Service

```kotlin
// server/src/main/kotlin/service/ClaudeService.kt

class ClaudeService(private val apiKey: String) {
    private val client = HttpClient(CIO) {
        install(ContentNegotiation) { json() }
    }

    suspend fun chat(
        messages: List<Message>,
        tools: List<ToolDefinition>
    ): ClaudeResponse {
        val response = client.post("https://api.anthropic.com/v1/messages") {
            header("x-api-key", apiKey)
            header("anthropic-version", "2023-06-01")
            contentType(ContentType.Application.Json)
            setBody(ChatRequest(
                model = "claude-sonnet-4-20250514",
                max_tokens = 1024,
                messages = messages,
                tools = tools
            ))
        }
        return response.body<ClaudeResponse>()
    }
}
```

---

## UI Design

```
┌────────────────────────────────┐
│  PocketMCP              [⚙️]   │  ← App bar with settings
├────────────────────────────────┤
│                                │
│  ┌──────────────────────┐      │
│  │ 🧑 Turn on my        │      │  ← User message (right-aligned)
│  │    flashlight         │      │
│  └──────────────────────┘      │
│                                │
│      ┌──────────────────────┐  │
│      │ 🤖 Using tool:       │  │  ← Assistant message (left-aligned)
│      │ ┌──────────────────┐ │  │
│      │ │ 🔦 flashlight     │ │  │  ← Tool call card (highlighted)
│      │ │    toggle → ON    │ │  │
│      │ │    ✅ Success     │ │  │
│      │ └──────────────────┘ │  │
│      │                      │  │
│      │ Done! I've turned on │  │
│      │ your flashlight.     │  │
│      └──────────────────────┘  │
│                                │
│  ┌──────────────────────┐      │
│  │ 🧑 What's my battery  │      │
│  │    at?                │      │
│  └──────────────────────┘      │
│                                │
│      ┌──────────────────────┐  │
│      │ 🤖 Using tool:       │  │
│      │ ┌──────────────────┐ │  │
│      │ │ 🔋 battery_status │ │  │
│      │ │    78% charging   │ │  │
│      │ │    ✅ Success     │ │  │
│      │ └──────────────────┘ │  │
│      │                      │  │
│      │ Your battery is at   │  │
│      │ 78% and charging.    │  │
│      └──────────────────────┘  │
│                                │
├────────────────────────────────┤
│ [  Type a message...     ] [➤] │  ← Input bar
└────────────────────────────────┘
```

---

## Task Breakdown

### Phase 1 — Foundation (Days 1-3)

- [ ] Create KMP project with Compose Multiplatform (use KMP Wizard or template)
- [ ] Set up module structure: `shared`, `composeApp`, `server`
- [ ] Add dependencies: Ktor client (shared), Ktor server, kotlinx.serialization
- [ ] Define domain models: `Message`, `ToolCall`, `ToolResult`, `ToolDefinition`
- [ ] Define `DeviceTool` interface and `ToolRegistry` in commonMain
- [ ] Build the Ktor backend proxy with POST /chat endpoint
- [ ] Implement `ClaudeService` that calls Anthropic API with tool definitions
- [ ] Test the proxy locally: send a message, get a response with tool_use

### Phase 2 — Chat UI (Days 4-5)

- [ ] Build `ChatScreen` with Compose Multiplatform
- [ ] Build `MessageBubble` composable (user vs assistant styling)
- [ ] Build `ToolCallCard` composable (shows tool name, input, result, status)
- [ ] Build `ChatViewModel` with conversation state management
- [ ] Wire up `ProxyApiClient` (Ktor client in shared) to talk to your backend
- [ ] Implement the tool use loop: send → receive tool_use → execute → send result → receive response
- [ ] Handle loading states, errors, and empty states

### Phase 3 — Easy Tools (Days 6-8)

Start with tools that need NO permissions:

- [ ] `DeviceInfoTool` — expect/actual, Android: `Build.*`, iOS: `UIDevice`
- [ ] `BatteryTool` — expect/actual, Android: `BatteryManager`, iOS: `UIDevice.batteryLevel`
- [ ] `ClipboardTool` (read + write) — expect/actual, both platforms
- [ ] `VibrateTool` — expect/actual, Android: `VibratorManager`, iOS: `UIImpactFeedbackGenerator`
- [ ] `FlashlightTool` — expect/actual, Android: `CameraManager`, iOS: `AVCaptureDevice`
- [ ] Test each tool end-to-end: user asks → Claude calls tool → phone executes → result shown

### Phase 4 — Permission Tools (Days 9-12)

Tools that require runtime permissions:

- [ ] Build a shared permission handling system (expect/actual)
- [ ] `LocationTool` — Android: `FusedLocationProviderClient` + ACCESS_FINE_LOCATION, iOS: `CLLocationManager` + requestWhenInUseAuthorization
- [ ] `CameraTool` — Android: CameraX + CAMERA permission, iOS: AVFoundation + camera usage description
- [ ] `NotificationTool` — Android: `NotificationManager` + POST_NOTIFICATIONS, iOS: `UNUserNotificationCenter` + requestAuthorization
- [ ] Build `PermissionDialog` composable that explains why the permission is needed before requesting
- [ ] Handle permission denied gracefully (tool returns error, Claude explains to user)

### Phase 5 — Polish (Days 13-15)

- [ ] Add a system prompt that tells Claude what tools are available and how to use them naturally
- [ ] Add conversation persistence (local storage with Room/SQLDelight)
- [ ] Add settings screen (backend URL config for development)
- [ ] Write unit tests for: ToolRegistry, ToolExecutor, ChatRepository, UseCases
- [ ] Write integration tests for the backend proxy
- [ ] Set up GitHub Actions CI: build Android + run tests on every push
- [ ] Write a proper README for the repo with screenshots and architecture diagram
- [ ] Add the project to your GitHub profile

---

## System Prompt for Claude

```
You are a helpful assistant running inside a mobile app called PocketMCP.
You have access to the user's device through tools. When the user asks you
to do something that involves the device, use the appropriate tool.

Be natural about it. Don't list your capabilities unless asked. Just do
what the user asks. If a tool fails (permission denied, hardware unavailable),
explain what happened clearly and suggest what the user can do.

You can chain multiple tools in one response if needed. For example, if
the user says "take a photo and tell me my battery level", use both tools.

Keep responses short and conversational. You're an assistant on their phone,
not a chatbot on a website.
```

---

## Dependencies

### shared (build.gradle.kts)
```kotlin
commonMain.dependencies {
    implementation("io.ktor:ktor-client-core:3.1.1")
    implementation("io.ktor:ktor-client-content-negotiation:3.1.1")
    implementation("io.ktor:ktor-serialization-kotlinx-json:3.1.1")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.10.1")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.8.0")
    // ViewModel
    implementation("org.jetbrains.androidx.lifecycle:lifecycle-viewmodel-compose:2.9.0")
}
androidMain.dependencies {
    implementation("io.ktor:ktor-client-okhttp:3.1.1")
    implementation("com.google.android.gms:play-services-location:21.3.0")
    implementation("androidx.camera:camera-camera2:1.4.1")
    implementation("androidx.camera:camera-lifecycle:1.4.1")
}
iosMain.dependencies {
    implementation("io.ktor:ktor-client-darwin:3.1.1")
}
```

### server (build.gradle.kts)
```kotlin
dependencies {
    implementation("io.ktor:ktor-server-core:3.1.1")
    implementation("io.ktor:ktor-server-netty:3.1.1")
    implementation("io.ktor:ktor-server-content-negotiation:3.1.1")
    implementation("io.ktor:ktor-serialization-kotlinx-json:3.1.1")
    implementation("io.ktor:ktor-client-cio:3.1.1")
    implementation("io.ktor:ktor-client-content-negotiation:3.1.1")
}
```

---

## What This Proves on Your GitHub Profile

When a recruiter clicks this repo, they see:

- **KMP** — shared code across Android and iOS, not a toy
- **Compose Multiplatform** — real UI, not platform-specific
- **expect/actual** — the KMP feature that matters, used properly across 8 tools
- **Claude API + tool use** — not just calling an API, building an agent
- **Ktor** — both as HTTP client (mobile) and server (backend)
- **Clean architecture** — domain/data/presentation, testable, modular
- **Permission handling** — real production concern, handled properly
- **CI/CD** — GitHub Actions running on every push
- **Full-stack KMP** — mobile + backend, one language, one repo

This is the repo that makes the rest of your profile believable.
