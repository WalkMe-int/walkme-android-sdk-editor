# WalkMe Android SDK (Editor / Power Mode) — integration guide

WalkMe **Power Mode**: PM-specific features on top of the core SDK. Artifact: **`walkme-android-sdk-editor`**.

## Requirements

- **Minimum SDK:** API **24+**
- Use Android Gradle Plugin and Kotlin versions compatible with your chosen SDK release (follow release notes if provided).

## 1. Add the JitPack repository

In your **root** `settings.gradle` / `settings.gradle.kts` (Gradle 7+):

**Groovy (`settings.gradle`)**

```gradle
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url "https://jitpack.io" }
    }
}
```

**Kotlin (`settings.gradle.kts`)**

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven(url = "https://jitpack.io")
    }
}
```

If repositories are declared only in the project `build.gradle`, add the same `maven { url "https://jitpack.io" }` there.

## 2. Add the dependency

Replace the version with any tag or commit published on JitPack.

**Groovy**

```gradle
dependencies {
    implementation "com.github.WalkMe-int:walkme-android-sdk-editor:1.1.2"
}
```

**Kotlin DSL**

```kotlin
dependencies {
    implementation("com.github.WalkMe-int:walkme-android-sdk-editor:1.1.2")
}
```

## 3. Compose dependencies (host app does **not** use Jetpack Compose)

The Editor SDK includes UI built with Compose. If your app **already** uses Compose, align versions with your own Compose BOM and you usually **do not** need the block below. If your app **does not** use Compose, add **all** of the following:

**Groovy**

```gradle
dependencies {
    implementation platform("androidx.compose:compose-bom:2025.12.00")
    implementation "androidx.compose.ui:ui:1.10.6"
    implementation "androidx.compose.ui:ui-tooling-preview:1.10.6"
    implementation "androidx.compose.material3:material3:1.4.0"
    implementation "androidx.compose.material:material-icons-core:1.7.8"
}
```

**Kotlin DSL**

```kotlin
dependencies {
    implementation(platform("androidx.compose:compose-bom:2025.12.00"))
    implementation("androidx.compose.ui:ui:1.10.6")
    implementation("androidx.compose.ui:ui-tooling-preview:1.10.6")
    implementation("androidx.compose.material3:material3:1.4.0")
    implementation("androidx.compose.material:material-icons-core:1.7.8")
}
```

Keep Compose library versions consistent with the BOM and with future SDK release notes if versions change.

## 4. Public API — `WalkmeSdkPowerMode`

**Package:** `com.walkme.pm`

| API                            | Purpose                                                                                                                                                                                                                                                                                |
|--------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `start(activity, options)`     | Start WalkMe in Power Mode. **Intended once per process**; further calls are ignored until `stop()` has run.                                                                                                                                                                           |
| `start(application, options)`  | Start WalkMe in Power Mode. **Intended once per process**; further calls are ignored until `stop()` has run.                                                                                                                                                                           |
| `stop()`                       | Stop Power Mode and the underlying SDK; after this, `start()` may be called again.                                                                                                                                                                                                     |
| `restart()`                    | Re-initialize Power Mode with the same options and host as the last successful `start()`. No-op if the SDK is not running (`start()` was not called, or `stop()` already ran).                                                                                                         |
| `setUserId(userId)`            | Set or clear (`null`) the end-user id for segmentation, analytics, and support.                                                                                                                                                                                                        |
| `setLanguage(language)`        | Set UI language where your WalkMe configuration supports it (requires the relevant admin option when applicable).                                                                                                                                                                      |
| `setVariable(key, value)`      | Set a custom variable used by WalkMe rules and segments; pass `null` for `value` to clear.                                                                                                                                                                                             |
| `setEventUserVars(values)`     | Set keys for WalkMe **event** payloads (`userVars`). Pass a `Map<WalkMeEventUserVarsKey, String>`. Each call **merges** into the stored map (same key overwrites). Use `com.walkme.api.WalkMeEventUserVarsKey` (`NAME`, `ROLE`, `TYPE`, `STATUS`, `INFO`).                             |
| `setTenantId(tenantId)`        | Set or clear (`null`) the tenant ID for the current user (max **50 characters**; longer values are truncated). Attached to analytics events for tenant-based reporting. Call after sign-in when the tenant is known; pass `null` on sign-out. Persisted across app sessions.                                                             |
| `startItemByID(itemId, deepLink?)` | Start a specific **promotion** by WalkMe `itemId`. If another promotion is already playing, it is stopped first. Optional `deepLink` is a URI string; when non-null and your app can resolve `ACTION_VIEW` for that URI (same package), the SDK opens it before playing the promotion. |
| `dismissItem()` | Dismiss the **currently presented** WalkMe promotion (not launchers). Does not stop the SDK. No-op if no promotion is active or the SDK is not started. |
| `sendEvent(name, attributes)`  | Sends a custom tracked event: name identifies the event, attributes is an optional map of key/value data.                                                                                                                                                                              |
| `setItemInfoListener(listener)` | Register a listener for item lifecycle callbacks (`onItemPresented`, `onItemDismissed`, `onItemAction`). Pass `null` to clear. See **Item info callbacks** below.                                                                                                          |
| `setAnalyticsListener(listener)` | Register a listener for successfully posted analytics events (`onSendAnalyticsEvent`). Pass `null` to clear. See **Analytics callbacks** below. No callbacks when `analyticsEnabled` is `false`. |

**Startup options**

`com.walkme.api.WalkMeStartOptions` — same as the core SDK.

| Option | Type | Default | Purpose |
|--------|------|---------|---------|
| `systemGuid` | `String` | — | WalkMe system GUID (required). |
| `environment` | `String` | `"Production"` | Environment name (e.g. `"Production"`). May be overridden internally (e.g. preview mode). |
| `dataCenter` | `WalkmeDataCenter` | `prod` | Region — `prod`, `eu`, `us01`, `eu01`, or `Custom("…")` for any wire string. |
| `analyticsEnabled` | `Boolean` | `true` | When `false`, the SDK does not send analytics/events to WalkMe (including heartbeat). |
| `localLogsEnabled` | `Boolean` | `false` | When `true`, SDK debug logs are written to Logcat (`WMLogger`). Use for troubleshooting only. |

`analyticsEnabled` and `localLogsEnabled` are mutable properties (not constructor parameters). Set them on the options instance before calling `start`:

```kotlin
val options = WalkMeStartOptions(
    systemGuid = "<YOUR_SYSTEM_GUID>",
    environment = "Production",
    dataCenter = WalkmeDataCenter.eu,
)
options.analyticsEnabled = true   // default
options.localLogsEnabled = false  // default; set true for debug builds if needed
```

**Example (Kotlin)**

```kotlin
import com.walkme.api.WalkMeStartOptions
import com.walkme.api.WalkmeDataCenter
import com.walkme.pm.WalkmeSdkPowerMode

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val options = WalkMeStartOptions(
            systemGuid = "<YOUR_SYSTEM_GUID>",
            environment = "Production",
            dataCenter = WalkmeDataCenter.eu,
        )
        WalkmeSdkPowerMode.start(this, options)
    }

    override fun onDestroy() {
        WalkmeSdkPowerMode.stop()
        super.onDestroy()
    }
}
```

Adjust `environment` and `dataCenter` to match your WalkMe environment.

## 5. Item info callbacks

Register **`WMItemInfoListener`** (`com.walkme.api`) to receive item lifecycle events and forward them to your analytics, CRM, or app logic.

| Callback | When it fires |
|----------|----------------|
| `onItemPresented(itemInfo)` | Right before a deployable item is shown (including when `environment` is `"preview"`). |
| `onItemDismissed(itemInfo)` | After a deployable item is dismissed (close, submit, remind-me-later, etc.), including in `"preview"`. |
| `onItemAction(itemInfo, args)` | When the user performs an action on a deployable (e.g. button click), including in `"preview"`. `args` is an optional map of action parameters. |

**`WMItemInfo`** (`itemId`, `itemActionType`, `userData`) — context for the item and the action that triggered the callback (if any).

**`WMUserData`** — user/device snapshot at interaction time: `userAttributesMap`, `sessionDuration`, `deviceVersion`, `deviceId`, `deviceModel`, `deviceOrientation`, `appVersion`, `appName`, `locale`, `sdkVer`, `sessionId`, `isNewUser` (`"true"` / `"false"`), `timezone`, `network`, `systemName`, `timestamp`.

Callbacks are delivered on the **main thread** in all environments, including Power Mode preview (`environment == "preview"`). The listener is cleared when you call `stop()`.

**Power Mode note:** Like other runtime APIs, `setItemInfoListener` is **ignored** while a PM account session is active and the SDK is **not** in preview mode. In preview mode, register the listener after `start()` to receive item callbacks while testing deployables.

**Example (Kotlin)**

```kotlin
import com.walkme.api.WMItemInfo
import com.walkme.api.WMItemInfoListener
import com.walkme.pm.WalkmeSdkPowerMode

WalkmeSdkPowerMode.setItemInfoListener(object : WMItemInfoListener {
    override fun onItemPresented(itemInfo: WMItemInfo) {
        // Item about to show — itemInfo.itemId, itemInfo.userData, …
    }

    override fun onItemDismissed(itemInfo: WMItemInfo) {
        // Item dismissed — itemInfo.itemActionType describes the dismiss action
    }

    override fun onItemAction(itemInfo: WMItemInfo, args: Map<String, String>?) {
        // User interacted with a deployable action
    }
})

// On teardown:
WalkmeSdkPowerMode.setItemInfoListener(null)
```

## 6. Analytics callbacks

Register **`WMAnalyticsListener`** (`com.walkme.api.analytics`) to receive analytics payloads after the SDK successfully posts them to WalkMe (`event/postEvent`).

| Callback | When it fires |
|----------|----------------|
| `onSendAnalyticsEvent(eventName, params)` | After an analytics event POST succeeds. Not called on network failure or when events are not sent. |

- **`eventName`** — WalkMe event type string (same as the `"type"` field in the POST body, e.g. `"play"`, `"click"`, `"activity"`).
- **`params`** — Full JSON body posted to WalkMe (`time`, `type`, `data`, `env`, `version`, `wm`, `ctx`, `sId`). Treat as read-only.

Callbacks are delivered on the **main thread**. The listener is cleared when you call `stop()`. No callbacks are delivered when **`analyticsEnabled`** is `false` on your startup options (the SDK does not send events in that case).

**Power Mode note:** Like other runtime APIs, `setAnalyticsListener` is **ignored** while a PM account session is active and the SDK is **not** in preview mode. In preview mode, register the listener after `start()` to receive analytics callbacks while testing.

**Example (Kotlin)**

```kotlin
import com.walkme.api.analytics.WMAnalyticsListener
import com.walkme.pm.WalkmeSdkPowerMode
import org.json.JSONObject

WalkmeSdkPowerMode.setAnalyticsListener(object : WMAnalyticsListener {
    override fun onSendAnalyticsEvent(eventName: String, params: JSONObject) {
        // Forward to your analytics — eventName, params.optJSONObject("data"), …
    }
})

// On teardown:
WalkmeSdkPowerMode.setAnalyticsListener(null)
```

## 7. Permalinks

WalkMe permalinks let external links invoke SDK actions via a custom URL scheme. The SDK handles them automatically when a host `Activity` starts with a matching `VIEW` intent — no manual forwarding code is required.

**Prerequisites**

1. **`systemGuid` from WalkMe onboarding** — the same value must be used in:
   - `WalkMeStartOptions.systemGuid` passed to `start()`
   - The manifest intent-filter `android:scheme` (via `WalkMePermalinks.scheme(systemGuid)`)
2. **SDK must be started** — the host manifest registers `com.walkme.api.{systemGuid}`. After `start()`, the SDK validates the permalink scheme against `WalkMeStartOptions.systemGuid` and dispatches the action. Permalinks received before `start()` are saved and replayed when `start()` completes.
3. **Call `start()` in `onCreate`** — permalinks on the same launch are queued until `start()` finishes, then processed automatically:

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    WalkmeSdkPowerMode.start(this, options) // must run before permalink handling on this launch
    // ...
}
```

If your app can be cold-started via a permalink, consider calling `start()` from `Application.onCreate` instead.

**URL format (v1.0):**

```text
com.walkme.api.{systemGuid}://1.0/{action}?{query}
```

Use `com.walkme.api.permalink.WalkMePermalinks.scheme(systemGuid)` to build the manifest scheme string (replace `systemGuid` with your WalkMe project GUID).

| Action | Path | Required query | SDK API |
|--------|------|----------------|---------|
| Restart SDK | `restart_sdk` | — | `restart()` |
| Start item | `start_item` | `item_id` | `startItemByID(itemId)` |
| Start item + redirect | `start_item` | `item_id`, `redirect` | `startItemByID(itemId, redirect)` |
| Send tracked event | `send_event` | `name` | `sendEvent(name, attributes)` |
| Set variable | `set_variable` | `key` | `setVariable(key, value)` |
| Set end user ID | `set_user_id` | — | `setUserId(userId)` — `user_id` optional |

**Example:** `com.walkme.api.c22c935518874267b946f5ae49b21d20://1.0/restart_sdk`

**Manifest** — add an intent-filter to your deep-link `Activity`. The scheme must use the **same** `systemGuid` you pass to `start()`:

```xml
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data
        android:scheme="com.walkme.api.YOUR_SYSTEM_GUID"
        android:host="1.0" />
</intent-filter>
```

For hot delivery (`singleTop` / `singleTask`), call `setIntent(intent)` in `onNewIntent` so the latest permalink is visible to the SDK.

## 8. Integration checklist

1. Add **JitPack** to repositories.
2. Add **`walkme-android-sdk-editor`** with your release version.
3. If the app is **not** Compose-based, add the **Compose** dependencies in §3.
4. Obtain **`systemGuid`**, **`environment`**, and **`dataCenter`** from your WalkMe project / onboarding.
5. Call **`start`** once per process when ready; call **`stop`** before starting again or when tearing down.
6. Wire **`setUserId`** / **`setVariable`** / **`setTenantId`** after login and clear on logout if your policy requires it.
7. Optionally register **`setItemInfoListener`** after `start()` if you need item lifecycle hooks.
8. Optionally register **`setAnalyticsListener`** after `start()` if you need successfully posted analytics payloads.
9. Add a **permalink intent-filter** (§7) on your deep-link `Activity` using the same `systemGuid` as in `start()`.

---

**Related:** For the core SDK only (no Power Mode), see [Walkme-Android-Sdk](https://github.com/WalkMe-int/walkme-android-sdk). Do not add both artifacts at the same time.
