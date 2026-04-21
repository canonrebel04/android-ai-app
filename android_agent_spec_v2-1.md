# Android Local Agent — Full Specification v2
### OpenClaw-Level Intelligence + Native Phone Control

> A self-contained Android agent that surpasses OpenClaw: every model OpenRouter offers, a skills plugin system, messaging-app UI channels, voice mode, persistent memory — plus what OpenClaw can never do: direct AccessibilityService control of any app on your phone.

---

## 1. What This Is vs. OpenClaw

| Capability | OpenClaw | This App |
|---|---|---|
| 300+ models via OpenRouter | ✅ | ✅ |
| Claude, GPT, Gemini, Mistral, Llama etc. | ✅ | ✅ |
| Tiered model routing (cheap → expensive) | ✅ | ✅ |
| Model fallback chains | ✅ | ✅ |
| Skills / plugin system | ✅ | ✅ |
| Telegram / WhatsApp as UI | ✅ | ✅ |
| Voice mode (TTS + STT) | ✅ | ✅ |
| Persistent memory across sessions | ✅ | ✅ |
| Runs fully on-device, no PC gateway needed | ❌ | ✅ |
| Screen reading (AccessibilityService) | ❌ | ✅ |
| Tapping / swiping any app | ❌ | ✅ |
| Launching and controlling apps | ❌ | ✅ |
| Native Android UI (Jetpack Compose) | ❌ | ✅ |
| Rust core (memory-safe, fast) | ❌ (Node.js) | ✅ |

**The differentiator:** OpenClaw is a messaging bot that can run shell commands. This app IS the phone. It can open your banking app, navigate to transfers, fill in an amount, and hit Send — all autonomously.

---

## 2. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                        Android App Process                        │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    Jetpack Compose UI                        │ │
│  │   Home · History · Skills · Models · Settings · Channels    │ │
│  └────────────────────────┬────────────────────────────────────┘ │
│                            │ StateFlow                            │
│  ┌─────────────────────────▼──────────────────────────────────┐  │
│  │                  AgentViewModel (Kotlin)                     │  │
│  │         Coroutines · Channel bridge · Log relay             │  │
│  └────────────────────────┬───────────────────────────────────┘  │
│                            │ JNI                                   │
│  ┌─────────────────────────▼───────────────────────────────────┐  │
│  │                    Rust Core (libagent.so)                   │  │
│  │                                                              │  │
│  │  ┌──────────────┐  ┌─────────────┐  ┌────────────────────┐ │  │
│  │  │  Agent Loop  │  │ Model Router│  │  Skills Engine     │ │  │
│  │  │  State Mach. │  │ Tier Select │  │  Plugin Loader     │ │  │
│  │  └──────┬───────┘  └──────┬──────┘  └──────────┬─────────┘ │  │
│  │         │                 │                     │           │  │
│  │  ┌──────▼─────────────────▼─────────────────────▼─────────┐│  │
│  │  │              HTTP Client (reqwest)                      ││  │
│  │  │   OpenRouter · Mistral · Anthropic · Google · Local     ││  │
│  │  └─────────────────────────────────────────────────────────┘│  │
│  │                                                              │  │
│  │  ┌───────────────┐  ┌──────────────┐  ┌──────────────────┐ │  │
│  │  │Context Manager│  │Memory Manager│  │ Log Ring Buffer  │ │  │
│  │  │Token Trimmer  │  │MEMORY.md I/O │  │ Trace Output     │ │  │
│  │  └───────────────┘  └──────────────┘  └──────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────┐   ┌──────────────────────────────────┐ │
│  │ AccessibilityService │   │      Channel Services            │ │
│  │ Screen Reader        │   │  TelegramBotService              │ │
│  │ Node Resolver        │   │  WhatsApp (via Accessibility)    │ │
│  │ Gesture Executor     │   │  VoiceService (STT/TTS)          │ │
│  └──────────────────────┘   └──────────────────────────────────┘ │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │           Gateway WebSocket Server (optional)                 │ │
│  │      Expose agent to local network / Pi / desktop            │ │
│  └──────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
        │ HTTPS (all providers)          │ Intents / Gestures
        ▼                                ▼
  OpenRouter (300+ models)        Android OS / Apps
  Mistral Direct
  Anthropic Direct
  Google Gemini
  Local llama.cpp
```

---

## 3. Model Layer — Full OpenRouter Parity

### 3.1 Supported Providers

| Provider | Base URL | Auth |
|----------|----------|------|
| OpenRouter | `https://openrouter.ai/api/v1` | Bearer API key |
| Mistral Direct | `https://api.mistral.ai/v1` | Bearer API key |
| Anthropic Direct | `https://api.anthropic.com/v1` | `x-api-key` header |
| Google Gemini | `https://generativelanguage.googleapis.com/v1beta/openai` | Bearer API key |
| OpenAI | `https://api.openai.com/v1` | Bearer API key |
| Local (llama.cpp) | `http://127.0.0.1:8080/v1` | None |
| Custom endpoint | User-defined | User-defined |

OpenRouter is the recommended default. A single key unlocks 300+ models including every Claude, GPT, Gemini, Llama, Mistral, Qwen, DeepSeek, Kimi, MiniMax, and more.

### 3.2 Tiered Model Routing

Replicate OpenClaw's most powerful feature: automatically route tasks to the cheapest model that can handle them. Don't burn Opus tokens on "set a timer."

```rust
pub enum TaskComplexity {
    Trivial,    // Heartbeats, status checks, simple reminders
    Standard,   // Everyday tasks, drafting, searching
    Complex,    // Multi-step planning, code, analysis
    Critical,   // High-stakes actions: payments, sends, deletes
}

pub struct ModelTier {
    pub complexity: TaskComplexity,
    pub primary: String,          // e.g. "openrouter/google/gemini-flash-2.5"
    pub fallbacks: Vec<String>,   // tried in order if primary fails/times out
    pub max_tokens: u32,
    pub temperature: f32,
}
```

**Default tier configuration (user-editable):**

| Tier | Default Model | Est. Cost/1M tokens in |
|------|--------------|------------------------|
| Trivial | `openrouter/google/gemini-flash-2.5` | ~$0.075 |
| Standard | `openrouter/mistralai/mistral-small-3.2` | ~$0.10 |
| Complex | `openrouter/anthropic/claude-sonnet-4-6` | ~$3.00 |
| Critical | `openrouter/anthropic/claude-opus-4-6` | ~$15.00 |

**Complexity classifier** (runs before every LLM call, zero-cost — rule-based in Rust):

```rust
fn classify_task(prompt: &str, context: &AgentContext) -> TaskComplexity {
    let word_count = prompt.split_whitespace().count();
    let has_destructive_keywords = ["send", "delete", "pay", "transfer", "buy", "post"]
        .iter().any(|kw| prompt.to_lowercase().contains(kw));
    let has_code_keywords = ["code", "script", "write", "debug", "function"]
        .iter().any(|kw| prompt.to_lowercase().contains(kw));
    let multi_step_indicator = ["then", "after", "next", "finally", "step"]
        .iter().any(|kw| prompt.to_lowercase().contains(kw));

    if has_destructive_keywords { return TaskComplexity::Critical; }
    if has_code_keywords || multi_step_indicator { return TaskComplexity::Complex; }
    if word_count > 15 { return TaskComplexity::Standard; }
    TaskComplexity::Trivial
}
```

### 3.3 Model Fallback Chains

```rust
pub async fn call_with_fallback(request: &LlmRequest, tiers: &[ModelTier]) -> Result<LlmResponse> {
    let tier = select_tier(request.complexity, tiers);
    let all_models = std::iter::once(&tier.primary)
        .chain(tier.fallbacks.iter());

    for model in all_models {
        match http_client::call(model, request).await {
            Ok(response) => return Ok(response),
            Err(LlmError::RateLimit(retry_after)) => {
                tokio::time::sleep(Duration::from_secs(retry_after)).await;
                continue;
            }
            Err(LlmError::ModelUnavailable) => continue,
            Err(e) => return Err(e),
        }
    }
    Err(LlmError::AllFallbacksExhausted)
}
```

### 3.4 Prompt Caching

For providers that support it (Anthropic, OpenRouter on select models), send cache-control markers on the system prompt and memory file. This cuts input costs 50–90% on cached tokens for an always-on agent.

```rust
fn build_messages_with_cache(system: &str, memory: &str, history: &[Message], user_turn: &str) -> Value {
    json!({
        "system": [
            { "type": "text", "text": system,
              "cache_control": { "type": "ephemeral" } },   // cache system prompt
            { "type": "text", "text": memory,
              "cache_control": { "type": "ephemeral" } }    // cache memory file
        ],
        "messages": build_history(history, user_turn)
    })
}
```

---

## 4. Skills / Plugin System

OpenClaw's most extensible feature — replicated in Rust with a clean trait-based plugin API.

### 4.1 Skill Definition

Each skill is a `.toml` config file + an optional Rust `.so` plugin or a pure-prompt skill:

```toml
# ~/.agent/skills/web_search.toml
[skill]
name = "web_search"
description = "Search the web for current information"
trigger_keywords = ["search", "look up", "find", "what is", "news"]
complexity = "Standard"

[tool]
name = "web_search"
parameters = { query = "string", max_results = "integer" }

[implementation]
type = "http"
url = "https://api.search.brave.com/res/v1/web/search"
auth_env = "BRAVE_API_KEY"
result_path = "web.results[*].{title,url,description}"
```

```toml
# ~/.agent/skills/phone_call.toml
[skill]
name = "phone_call"
description = "Make a phone call to a contact or number"
trigger_keywords = ["call", "phone", "dial", "ring"]
complexity = "Critical"
requires_confirmation = true

[implementation]
type = "android_intent"
action = "android.intent.action.CALL"
data_template = "tel:{number}"
```

```toml
# ~/.agent/skills/reminder.toml
[skill]
name = "set_reminder"
description = "Set a reminder or alarm at a specific time"
complexity = "Trivial"

[implementation]
type = "android_intent"
action = "android.intent.action.INSERT"
extras = { title = "{title}", beginTime = "{time_ms}" }
```

### 4.2 Built-in Skill Library

| Skill | Description | Implementation |
|-------|-------------|----------------|
| `screen_control` | Tap, swipe, type on any app | AccessibilityService |
| `open_app` | Launch app by name or package | Android Intent |
| `web_search` | Search via Brave/SearXNG | HTTP |
| `web_browse` | Open URL, read page content | Chrome + Accessibility |
| `send_message` | Send SMS, Telegram, WhatsApp | Accessibility / Intent |
| `calendar` | Read/create calendar events | ContentProvider |
| `contacts` | Look up contact info | ContentProvider |
| `phone_call` | Make calls | Intent (with confirmation) |
| `set_alarm` | Set alarms/timers | AlarmManager Intent |
| `file_read` | Read a file from storage | File API |
| `file_write` | Write/create a file | File API |
| `clipboard` | Read or write clipboard | ClipboardManager |
| `screenshot` | Capture current screen | MediaProjection |
| `notifications` | Read notification list | NotificationListenerService |
| `shell_cmd` | Run shell command (Termux IPC) | Intent to Termux |
| `camera` | Take a photo | CameraX |
| `tts_speak` | Speak text aloud | Android TTS |
| `memory_read` | Read from MEMORY.md | File API |
| `memory_write` | Update MEMORY.md | File API |

### 4.3 Skill Loader (Rust)

```rust
pub trait Skill: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn to_tool_schema(&self) -> serde_json::Value;
    fn execute(&self, params: &serde_json::Value) -> Pin<Box<dyn Future<Output = SkillResult>>>;
}

pub struct SkillRegistry {
    skills: HashMap<String, Box<dyn Skill>>,
}

impl SkillRegistry {
    pub fn load_from_dir(path: &Path) -> Self {
        let mut registry = Self::default();
        // Load .toml skill configs
        for entry in fs::read_dir(path).unwrap() {
            if let Ok(skill) = TomlSkill::load(&entry.path()) {
                registry.register(Box::new(skill));
            }
        }
        // Register built-in skills
        registry.register(Box::new(ScreenControlSkill::new()));
        registry.register(Box::new(WebSearchSkill::new()));
        // ...
        registry
    }

    pub fn tools_for_prompt(&self) -> Vec<serde_json::Value> {
        self.skills.values().map(|s| s.to_tool_schema()).collect()
    }
}
```

---

## 5. Messaging Channels — Alternative UIs

Just like OpenClaw: let users control the agent via Telegram, WhatsApp, or any messaging app they already use. The Jetpack Compose UI remains primary, but channels give you remote access.

### 5.1 Telegram Bot

```kotlin
class TelegramBotService : Service() {
    private val client = OkHttpClient()
    private var offset = 0L
    private lateinit var botToken: String

    fun startPolling() {
        CoroutineScope(Dispatchers.IO).launch {
            while (isRunning) {
                val updates = pollUpdates(offset)
                updates.forEach { update ->
                    val text = update.message?.text ?: return@forEach
                    val chatId = update.message.chat.id
                    handleCommand(text, chatId)
                    offset = update.updateId + 1
                }
                delay(1000)
            }
        }
    }

    private suspend fun handleCommand(text: String, chatId: Long) {
        when {
            text.startsWith("/status") -> sendMessage(chatId, agentViewModel.statusSummary())
            text.startsWith("/stop")   -> { agentViewModel.stopTask(); sendMessage(chatId, "Agent stopped.") }
            text.startsWith("/logs")   -> sendMessage(chatId, agentViewModel.recentLogs())
            text.startsWith("/model ") -> { changeModel(text.removePrefix("/model ")); sendMessage(chatId, "Model updated.") }
            else -> {
                // Treat as a task
                sendMessage(chatId, "Starting task…")
                agentViewModel.startTask(text) { update ->
                    sendMessage(chatId, update)  // stream progress back to Telegram
                }
            }
        }
    }
}
```

**Telegram commands supported:**

| Command | Action |
|---------|--------|
| `/status` | Current agent state + last action |
| `/stop` | Halt running task |
| `/logs [n]` | Last n log lines |
| `/model <name>` | Switch active model |
| `/skills` | List installed skills |
| `/memory` | Show current MEMORY.md |
| `/tier <trivial\|standard\|complex\|critical> <model>` | Update routing tier |
| Any other text | Execute as a new task |

### 5.2 WhatsApp Channel

WhatsApp doesn't have a bot API. Instead, use the AccessibilityService to monitor WhatsApp notifications from a designated trigger contact/group, parse the message, and reply via the WhatsApp compose field. Opt-in only, clearly disclosed.

### 5.3 Voice Channel

```kotlin
class VoiceService : Service() {
    private val speechRecognizer = SpeechRecognizer.createSpeechRecognizer(this)
    private val tts = TextToSpeech(this) { /* init */ }

    fun startListening(wakeWord: String = "hey agent") {
        // Use on-device wake word detection (Porcupine or Rhino via ONNX)
        // Then start full STT on wake
        val intent = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH).apply {
            putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
            putExtra(RecognizerIntent.EXTRA_PARTIAL_RESULTS, true)
        }
        speechRecognizer.startListening(intent)
    }

    fun speak(text: String) {
        tts.speak(text, TextToSpeech.QUEUE_FLUSH, null, null)
    }
}
```

---

## 6. Persistent Memory System

OpenClaw uses a `MEMORY.md` file on disk. Replicate this — the agent reads and writes a structured memory file that persists across sessions and is injected into every system prompt.

### 6.1 Memory File Structure

```markdown
# Agent Memory

## User Profile
- Name: [extracted from conversations]
- Preferred model: claude-sonnet-4-6
- Communication style: concise, technical

## Persistent Facts
- Work email: ...
- Home WiFi name: ...
- Usual morning routine: ...

## Recent Context
- Last task: "Sent email to Alex about the meeting" (2026-04-21)
- Pending: "Follow up with dentist office"

## Learned Preferences
- Prefers Telegram for status updates
- Dislikes verbose responses
- Always confirms before sending messages

## Skills Used Most
1. send_message (47 times)
2. web_search (31 times)
3. screen_control (28 times)
```

### 6.2 Memory Update Loop

After every completed task, the agent calls the model with:

```
System: You are a memory manager. The user just completed a task. 
Update the memory file to reflect any new facts, preferences, or context learned.
Return ONLY the updated memory file. Keep it under 800 tokens.

Current memory:
{MEMORY_MD}

Completed task: {TASK}
Task summary: {SUMMARY}
```

The Rust core writes the result back to `~/.agent/MEMORY.md`. This model call uses the Trivial tier (cheap).

---

## 7. Gateway WebSocket Server (Optional)

Expose the agent over your local network or Tailscale so your Pi, laptop, or desktop can send it tasks.

```rust
pub struct GatewayServer {
    port: u16,
    auth_token: String,
}

impl GatewayServer {
    pub async fn start(&self) {
        let server = TcpListener::bind(format!("127.0.0.1:{}", self.port)).await.unwrap();
        while let Ok((stream, _)) = server.accept().await {
            let token = self.auth_token.clone();
            tokio::spawn(async move {
                let ws = accept_async(stream).await.unwrap();
                handle_connection(ws, token).await;
            });
        }
    }
}

// WebSocket message protocol:
// Client → Agent: { "type": "task", "prompt": "...", "model": "..." }
// Agent → Client: { "type": "log", "text": "..." }
// Agent → Client: { "type": "action", "action": {...} }
// Agent → Client: { "type": "done", "summary": "..." }
// Agent → Client: { "type": "confirm_required", "action": {...} }
// Client → Agent: { "type": "confirm", "approved": true }
```

**In settings:** "Gateway" toggle. Shows local IP and auth token (QR code for easy pairing with Pi).

---

## 8. Rust Core — Updated Module List

```
rust/src/
├── lib.rs
├── jni_exports.rs          # All JNI entry points
├── agent_loop.rs           # Main perception→reason→act cycle
├── state_machine.rs        # Task lifecycle states
├── model_router.rs         # Tier selection + fallback chains
├── http_client.rs          # Multi-provider HTTP with retries
├── provider/
│   ├── openrouter.rs       # OpenRouter request shaping
│   ├── anthropic.rs        # Anthropic direct (cache markers)
│   ├── google.rs           # Gemini via OpenAI-compat
│   ├── mistral.rs          # Mistral direct
│   └── local.rs            # llama.cpp / Ollama
├── tool_parser.rs          # Parse LLM function calls → AgentAction
├── skill_registry.rs       # Trait + loader for skills
├── skills/
│   ├── screen_control.rs   # Bridges to AccessibilityService via JNI callback
│   ├── web_search.rs       # HTTP search skill
│   └── shell_cmd.rs        # Termux IPC
├── context_manager.rs      # Rolling history with token budget
├── memory_manager.rs       # MEMORY.md read/write/update
├── prompt_cache.rs         # Anthropic/OpenRouter cache_control markers
├── complexity_classifier.rs
├── log_ring_buffer.rs
└── gateway_server.rs       # Optional WebSocket gateway
```

---

## 9. Android Project Structure (Updated)

```
app/
├── src/main/kotlin/com/yourdomain/agent/
│   ├── ui/screens/
│   │   ├── HomeScreen.kt           # Task input + live log
│   │   ├── ModelsScreen.kt         # Browse/configure all model tiers
│   │   ├── SkillsScreen.kt         # Enable/disable/install skills
│   │   ├── ChannelsScreen.kt       # Telegram, WhatsApp, Voice setup
│   │   ├── MemoryScreen.kt         # View and edit MEMORY.md
│   │   ├── HistoryScreen.kt        # Past tasks with token/cost data
│   │   ├── GatewayScreen.kt        # Local gateway config + QR code
│   │   └── PermissionsScreen.kt    # Onboarding
│   ├── service/
│   │   ├── AgentAccessibilityService.kt
│   │   ├── TelegramBotService.kt
│   │   ├── VoiceService.kt
│   │   ├── NotificationListenerService.kt
│   │   └── GatewayWebSocketService.kt
│   ├── bridge/
│   │   └── RustBridge.kt
│   └── data/
│       ├── KeystoreManager.kt
│       ├── SkillRepository.kt
│       └── db/  (Room: tasks, skills, memory_snapshots)
└── rust/  (as before)
```

---

## 10. Settings Screen — Full Option Set

### Models
- Provider picker (OpenRouter / Mistral / Anthropic / Gemini / OpenAI / Local / Custom)
- API key per provider (Keystore-backed)
- Tier configuration: Trivial / Standard / Complex / Critical model per tier
- Fallback chain editor (add/reorder models)
- Enable prompt caching: toggle (for supported providers)
- Auto model: use `openrouter/auto` for intelligent routing

### Skills
- Installed skills list with enable/disable toggles
- Skill detail: description, trigger keywords, complexity tier, last used, usage count
- Install from URL (`.toml` file) or built-in gallery
- Per-skill confirmation toggle (require user approval before execution)

### Channels
- Telegram: bot token input, test connection, notification settings
- WhatsApp: enable monitoring (explicit consent UI), trigger contact selector
- Voice: wake word on/off, wake word phrase, TTS engine/voice selection
- Gateway: enable toggle, port, auth token, QR code, Tailscale mode

### Agent Behavior
- Max steps per task (10–100, default 50)
- Action delay between steps (0–3000ms, default 300ms)
- Stall timeout (2–10s)
- Loop guard (max identical screens before abort)
- Require confirmation for: Send / Delete / Payment / Call (individual toggles)
- Vision Mode: off / on (adds screenshot to every LLM call, higher cost)

### Memory
- Enable persistent memory: toggle
- View/edit MEMORY.md directly
- Memory update frequency: after every task / daily summary / manual only
- Clear memory

### Usage & Cost
- Token usage by task (today / week / month)
- Estimated cost by tier
- Budget alert threshold (notify when monthly spend exceeds $X)
- Export usage log as CSV

---

## 11. UI — Home Screen (Updated)

```
┌─────────────────────────────────────┐
│ ◉ AGENT  [claude-sonnet]  [Running] │  ← Status + active model
├─────────────────────────────────────┤
│ "Open Gmail, find the email from    │
│  Sarah about the contract, and      │  ← Current task
│  forward it to Mike"                │
├─────────────────────────────────────┤
│ LIVE LOG                    [⏸][■] │
│ ─────────────────────────────────── │
│ → Classifier: Complex → Sonnet      │  ← Tier routing shown
│ → Skill: screen_control             │
│ → Opening Gmail…                    │
│ → Found: "Sarah - Contract draft"   │
│ → Tapped email                      │
│ ⟳ Reasoning (claude-sonnet-4-6)…   │
│                                     │
├─────────────────────────────────────┤
│ ┌────────────────────────────────┐  │
│ │  Enter task or use voice…   🎙 │  │
│ └────────────────────────────────┘  │
│  [▶ Run]  [Skills ▾]  [Model ▾]    │
└─────────────────────────────────────┘
```

**Quick access from home:**
- Model switcher dropdown (tap to override tier for this task only)
- Skills pill — shows active skills for this session
- Voice button — tap to use STT for task input
- Long-press log line → copy text or report issue

---

## 12. Permissions Manifest

```xml
<!-- Screen control -->
<uses-permission android:name="android.permission.BIND_ACCESSIBILITY_SERVICE"/>

<!-- Background persistence -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
<uses-permission android:name="android.permission.WAKE_LOCK"/>
<uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS"/>
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>

<!-- Networking -->
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>

<!-- Channels + skills -->
<uses-permission android:name="android.permission.RECORD_AUDIO"/>
<uses-permission android:name="android.permission.CAMERA"/>
<uses-permission android:name="android.permission.BIND_NOTIFICATION_LISTENER_SERVICE"/>
<uses-permission android:name="android.permission.READ_CONTACTS"/>
<uses-permission android:name="android.permission.READ_CALL_LOG"/>
<uses-permission android:name="android.permission.CALL_PHONE"/>
<uses-permission android:name="android.permission.SEND_SMS"/>
<uses-permission android:name="android.permission.READ_CALENDAR"/>
<uses-permission android:name="android.permission.WRITE_CALENDAR"/>

<!-- Storage (for memory + skill files) -->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>

<!-- Vision mode -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION"/>
```

**Onboarding permission flow:** Each permission group is explained on a dedicated card before the system dialog fires. Card text: what it enables, why it's needed, what it will never be used for.

---

## 13. Development Roadmap (Updated)

### Phase 1 — Core Engine (Weeks 1–2)
- [ ] Rust: `http_client` module with all 5 provider implementations
- [ ] Rust: `tool_parser` + `model_router` with tier selection
- [ ] Rust: `fallback_chain` retry logic
- [ ] Test via Termux: fire OpenRouter requests to 5+ different models
- [ ] Rust: `skill_registry` trait + TOML loader
- [ ] Rust: `context_manager` with token trimming
- [ ] Rust: `complexity_classifier` (rule-based, no LLM needed)

### Phase 2 — AccessibilityService (Week 3)
- [ ] Screen tree serializer → compact JSON
- [ ] Gesture executor (tap, swipe, type, scroll, key)
- [ ] Action confirmation dialog
- [ ] `screen_control` skill wrapping the service
- [ ] End-to-end: "Open calculator and type 42"

### Phase 3 — Agent Loop Integration (Week 4)
- [ ] JNI bridge + ViewModel wiring
- [ ] Full perception → model call → action → verify cycle
- [ ] Stall detection + loop guard
- [ ] Memory manager: read/write MEMORY.md, post-task update call

### Phase 4 — Skills & Channels (Week 5)
- [ ] Built-in skill library: web_search, open_app, calendar, contacts, clipboard, camera
- [ ] Telegram bot service + full command set
- [ ] Voice: STT task input + TTS response readback
- [ ] Notification listener skill
- [ ] Termux shell_cmd skill (IPC via Intent)

### Phase 5 — UI & Settings (Week 6)
- [ ] Full Jetpack Compose UI: Home, Models, Skills, Channels, Memory, History
- [ ] Tier configuration UI (drag-to-reorder fallbacks)
- [ ] Token usage + cost estimation tracking
- [ ] Onboarding permission flow
- [ ] Foreground service + persistent notification

### Phase 6 — Hardening (Week 7)
- [ ] Prompt caching for Anthropic + OpenRouter
- [ ] Budget alert system
- [ ] Gateway WebSocket server
- [ ] Vision Mode (MediaProjection)
- [ ] 10-scenario manual test matrix
- [ ] APK signing + sideload

---

## 14. Security Notes

- **Never expose the gateway to 0.0.0.0** without reverse proxy auth. Bind to `127.0.0.1` or use Tailscale.
- All API keys stored in Android Keystore (hardware-backed AES/GCM). Never logged, never in plain text config files.
- Skills from external `.toml` files can only use the declared implementation types (http, android_intent, shell_cmd). They cannot execute arbitrary Rust code unless compiled and sideloaded as a plugin `.so` — treat those with the same caution as APK installs.
- The `shell_cmd` skill (Termux IPC) is gated behind an explicit "Developer Mode" toggle with a red warning UI. Power users only.
- Prompt injection is a real threat: any text the agent reads from the screen (emails, web pages, notifications) could contain adversarial instructions. Apply a sanitization pass in `context_manager.rs` that strips common injection patterns before they reach the model.

---

## 15. What Makes This Better Than OpenClaw

1. **Phone control.** No other agent framework does this. OpenClaw can send a Telegram message. This app can open Telegram, navigate to a contact, compose a message, and send it — without any API.

2. **Zero dependency on a gateway PC.** OpenClaw's Android app is dead without a Mac/Linux gateway running somewhere. This app is fully standalone.

3. **Rust core.** OpenClaw is Node.js. On a phone, this matters: lower memory, faster startup, no GC pauses during action loops.

4. **Unified interface.** The same app controls the phone AND chats via Telegram AND responds to voice. OpenClaw needs multiple separate pieces to do all three.

5. **On-device model path.** Route trivial tasks to a local llama.cpp model. Zero cost, zero latency, zero network for the simple stuff.
