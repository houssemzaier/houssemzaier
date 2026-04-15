# PocketMCP — Claude Code Prompt

Copy-paste everything below into a fresh Claude Code session.

---

## PROMPT START

I want you to create a new private GitHub repository called `pocket-mcp` under my account `houssemzaier` and scaffold a complete Kotlin Multiplatform project.

### What is PocketMCP

A KMP mobile app (Android + iOS) where the user chats with Claude, and Claude can control the phone through tool use (function calling). The phone acts as an MCP server — Claude sends tool calls, the phone executes them natively, and returns the results.

There is also a Ktor backend proxy that sits between the app and Claude API, so the API key never lives on the device.

### Create the GitHub repo

Create a private repo `houssemzaier/pocket-mcp` with this description:
"KMP mobile agent — chat with Claude, your phone executes the tools. MCP for your pocket."

### Project structure to scaffold

Use the KMP Compose Multiplatform template structure. The project has 3 modules:

```
pocket-mcp/
├── shared/                          # KMP shared module
│   ├── src/
│   │   ├── commonMain/kotlin/com/houssemzaier/pocketmcp/
│   │   │   ├── domain/
│   │   │   │   ├── model/
│   │   │   │   │   ├── Message.kt
│   │   │   │   │   ├── ToolCall.kt
│   │   │   │   │   ├── ToolResult.kt
│   │   │   │   │   └── ToolDefinition.kt
│   │   │   │   └── usecase/
│   │   │   │       ├── SendMessageUseCase.kt
│   │   │   │       └── ExecuteToolUseCase.kt
│   │   │   ├── data/
│   │   │   │   ├── repository/
│   │   │   │   │   └── ChatRepository.kt
│   │   │   │   ├── remote/
│   │   │   │   │   └── ProxyApiClient.kt
│   │   │   │   └── tools/
│   │   │   │       ├── DeviceTool.kt
│   │   │   │       ├── ToolRegistry.kt
│   │   │   │       ├── ToolExecutor.kt
│   │   │   │       └── tools/
│   │   │   │           ├── FlashlightTool.kt
│   │   │   │           ├── BatteryTool.kt
│   │   │   │           ├── VibrateTool.kt
│   │   │   │           ├── ClipboardTool.kt
│   │   │   │           ├── CameraTool.kt
│   │   │   │           ├── LocationTool.kt
│   │   │   │           ├── DeviceInfoTool.kt
│   │   │   │           └── NotificationTool.kt
│   │   │   └── presentation/
│   │   │       ├── ChatViewModel.kt
│   │   │       └── ChatUiState.kt
│   │   ├── androidMain/kotlin/com/houssemzaier/pocketmcp/tools/
│   │   │   ├── FlashlightTool.android.kt
│   │   │   ├── BatteryTool.android.kt
│   │   │   ├── VibrateTool.android.kt
│   │   │   ├── ClipboardTool.android.kt
│   │   │   ├── CameraTool.android.kt
│   │   │   ├── LocationTool.android.kt
│   │   │   ├── DeviceInfoTool.android.kt
│   │   │   └── NotificationTool.android.kt
│   │   └── iosMain/kotlin/com/houssemzaier/pocketmcp/tools/
│   │       ├── FlashlightTool.ios.kt
│   │       ├── BatteryTool.ios.kt
│   │       ├── VibrateTool.ios.kt
│   │       ├── ClipboardTool.ios.kt
│   │       ├── CameraTool.ios.kt
│   │       ├── LocationTool.ios.kt
│   │       ├── DeviceInfoTool.ios.kt
│   │       └── NotificationTool.ios.kt
│   └── build.gradle.kts
│
├── composeApp/                      # Compose Multiplatform UI
│   ├── src/
│   │   ├── commonMain/kotlin/com/houssemzaier/pocketmcp/ui/
│   │   │   ├── App.kt
│   │   │   ├── ChatScreen.kt
│   │   │   ├── MessageBubble.kt
│   │   │   ├── ToolCallCard.kt
│   │   │   ├── PermissionDialog.kt
│   │   │   └── theme/
│   │   │       └── Theme.kt
│   │   ├── androidMain/
│   │   └── iosMain/
│   └── build.gradle.kts
│
├── server/                          # Ktor backend proxy
│   ├── src/main/kotlin/com/houssemzaier/pocketmcp/server/
│   │   ├── Application.kt
│   │   ├── routes/
│   │   │   └── ChatRoutes.kt
│   │   ├── service/
│   │   │   └── ClaudeService.kt
│   │   └── model/
│   │       ├── ChatRequest.kt
│   │       └── ChatResponse.kt
│   └── build.gradle.kts
│
├── build.gradle.kts
├── settings.gradle.kts
├── gradle.properties
├── .github/workflows/ci.yml
├── .gitignore
└── README.md
```

### Key interfaces and models to implement

#### DeviceTool interface (commonMain)

```kotlin
package com.houssemzaier.pocketmcp.data.tools

import com.houssemzaier.pocketmcp.domain.model.ToolResult
import kotlinx.serialization.json.JsonObject

interface DeviceTool {
    val name: String
    val description: String
    val parameters: JsonObject  // JSON Schema for Claude tool definition

    suspend fun execute(input: JsonObject): ToolResult
}
```

#### Domain models (commonMain)

```kotlin
// Message.kt
@Serializable
data class Message(
    val role: String,       // "user" or "assistant"
    val content: String
)

// ToolCall.kt
@Serializable
data class ToolCall(
    val id: String,
    val name: String,
    val input: JsonObject
)

// ToolResult.kt
@Serializable
data class ToolResult(
    val toolUseId: String,
    val success: Boolean,
    val content: String
)

// ToolDefinition.kt
@Serializable
data class ToolDefinition(
    val name: String,
    val description: String,
    val inputSchema: JsonObject
)
```

#### ToolRegistry (commonMain)

```kotlin
class ToolRegistry(private val tools: List<DeviceTool>) {
    fun getDefinitions(): List<ToolDefinition> =
        tools.map { ToolDefinition(it.name, it.description, it.parameters) }

    fun findTool(name: String): DeviceTool? =
        tools.firstOrNull { it.name == name }
}
```

### Tools to implement with expect/actual

Each tool file in commonMain should use `expect class`. Platform files provide `actual class`.

| Tool | Description for Claude | Android API | iOS API |
|------|----------------------|-------------|---------|
| `flashlight_toggle` | "Toggle the device flashlight on or off" | `CameraManager.setTorchMode()` | `AVCaptureDevice.torchMode` |
| `battery_status` | "Get current battery level and charging state" | `BatteryManager` | `UIDevice.current.batteryLevel` |
| `vibrate` | "Vibrate the device" | `VibratorManager` | `UIImpactFeedbackGenerator` |
| `clipboard_write` | "Copy text to clipboard" | `ClipboardManager` | `UIPasteboard.general` |
| `clipboard_read` | "Read text from clipboard" | `ClipboardManager` | `UIPasteboard.general` |
| `take_photo` | "Take a photo using device camera" | `CameraX` | `AVCaptureSession` |
| `get_location` | "Get current GPS coordinates" | `FusedLocationProviderClient` | `CLLocationManager` |
| `device_info` | "Get device model, OS version, system info" | `Build.*` | `UIDevice` |
| `set_reminder` | "Set a local notification reminder" | `NotificationManager` | `UNUserNotificationCenter` |

### Backend proxy (Ktor server)

The server has one endpoint:

```
POST /chat
Body: { "messages": [...], "tools": [...] }
Response: Claude's response (either text content or tool_use blocks)
```

The server holds the Claude API key (from environment variable `ANTHROPIC_API_KEY`) and forwards requests to `https://api.anthropic.com/v1/messages`.

### The tool use loop (this is how the app works)

```
1. User types a message
2. App sends messages + tool definitions to backend proxy
3. Backend forwards to Claude API
4. Claude responds — either plain text OR a tool_use block
5. If tool_use: app finds the tool in ToolRegistry, executes it on the device
6. App sends the tool_result back to the backend
7. Backend sends it to Claude so Claude can see the result
8. Claude responds with a natural language message
9. App displays it
```

### Gradle dependencies

**shared module:**
- `io.ktor:ktor-client-core:3.1.1`
- `io.ktor:ktor-client-content-negotiation:3.1.1`
- `io.ktor:ktor-serialization-kotlinx-json:3.1.1`
- `org.jetbrains.kotlinx:kotlinx-coroutines-core:1.10.1`
- `org.jetbrains.kotlinx:kotlinx-serialization-json:1.8.0`
- `org.jetbrains.androidx.lifecycle:lifecycle-viewmodel-compose:2.9.0`
- Android: `io.ktor:ktor-client-okhttp:3.1.1`, `com.google.android.gms:play-services-location:21.3.0`, `androidx.camera:camera-camera2:1.4.1`
- iOS: `io.ktor:ktor-client-darwin:3.1.1`

**server module:**
- `io.ktor:ktor-server-core:3.1.1`
- `io.ktor:ktor-server-netty:3.1.1`
- `io.ktor:ktor-server-content-negotiation:3.1.1`
- `io.ktor:ktor-serialization-kotlinx-json:3.1.1`
- `io.ktor:ktor-client-cio:3.1.1`
- `io.ktor:ktor-client-content-negotiation:3.1.1`

### GitHub Actions CI

Create `.github/workflows/ci.yml` that:
- Triggers on push to main and pull requests
- Runs on ubuntu-latest
- Sets up JDK 17
- Runs `./gradlew build` for the server module
- Runs `./gradlew :shared:testDebugUnitTest` for shared tests
- Runs `./gradlew :composeApp:assembleDebug` for Android build

### README.md for the repo

Write a professional README that includes:
- Project name and one-line description
- Architecture diagram (the ASCII one from above)
- How to run (backend + mobile)
- List of available tools
- Screenshots placeholder section
- Tech stack badges (KMP, CMP, Ktor, Claude)
- License: MIT

### What to scaffold vs what to leave as TODO

**Scaffold fully (compilable code):**
- All Gradle build files with correct dependencies and plugin configuration
- All domain models with serialization
- DeviceTool interface, ToolRegistry, ToolExecutor
- ProxyApiClient (Ktor HTTP client calling your backend)
- ChatRepository with the tool use loop logic
- ChatViewModel and ChatUiState
- All UI composables (ChatScreen, MessageBubble, ToolCallCard, PermissionDialog)
- Backend Application.kt, ChatRoutes.kt, ClaudeService.kt
- GitHub Actions CI
- README.md

**Scaffold as stubs (compiles but has TODO):**
- Each tool's expect/actual implementations — the common interface should be complete, platform implementations should have `TODO("Implement flashlight for Android")` etc.
- CameraTool and LocationTool permission handling

### Important

- The project MUST compile and build successfully after scaffolding
- Use Kotlin 2.1.0 and Compose Multiplatform 1.7.3 (or latest stable)
- Package name: `com.houssemzaier.pocketmcp`
- Min SDK Android: 26
- iOS deployment target: 16.0
- Create the repo as PRIVATE on GitHub
- Push all code to main branch
- Do NOT add any API keys anywhere in the code — use environment variables only

## PROMPT END
