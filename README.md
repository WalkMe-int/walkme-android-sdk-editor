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
    implementation "com.github.WalkMe-int:walkme-android-sdk-editor:0.1.7-beta11"
}
```

**Kotlin DSL**

```kotlin
dependencies {
    implementation("com.github.WalkMe-int:walkme-android-sdk-editor:0.1.7-beta11")
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
| `setUserId(userId)`            | Set or clear (`null`) the end-user id for segmentation, analytics, and support.                                                                                                                                                                                                        |
| `setLanguage(language)`        | Set UI language where your WalkMe configuration supports it (requires the relevant admin option when applicable).                                                                                                                                                                      |
| `setVariable(key, value)`      | Set a custom variable used by WalkMe rules and segments; pass `null` for `value` to clear.                                                                                                                                                                                             |
| `setEventUserVars(values)`     | Set keys for WalkMe **event** payloads (`userVars`). Pass a `Map<WalkMeEventUserVarsKey, String>`. Each call **merges** into the stored map (same key overwrites). Use `com.walkme.api.WalkMeEventUserVarsKey` (`NAME`, `ROLE`, `TYPE`, `STATUS`, `INFO`).                             |
| `startItemByID(itemId, deepLink?)` | Start a specific **promotion** by WalkMe `itemId`. If another promotion is already playing, it is stopped first. Optional `deepLink` is a URI string; when non-null and your app can resolve `ACTION_VIEW` for that URI (same package), the SDK opens it before playing the promotion. |
| `sendEvent(name, attributes)`  | Sends a custom tracked event: name identifies the event, attributes is an optional map of key/value data.                                                                                                                                                                              |

**Startup options**

- `com.walkme.api.WalkMeStartOptions` — same as the core SDK: `systemGuid` (required), `environment`, `dataCenter` ([WalkmeDataCenter]).
- `com.walkme.api.WalkmeDataCenter` — `prod`, `eu`, `us01`, `eu01`, or `Custom("…")` for any wire string.

**Example (Kotlin)**

```kotlin
import com.walkme.api.WalkMeStartOptions
import com.walkme.api.WalkmeDataCenter
import com.walkme.pm.WalkmeSdkPowerMode

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        WalkmeSdkPowerMode.start(
            this,
            WalkMeStartOptions(
                systemGuid = "<YOUR_SYSTEM_GUID>",
                environment = "Production",
                dataCenter = WalkmeDataCenter.eu,
            ),
        )
    }

    override fun onDestroy() {
        WalkmeSdkPowerMode.stop()
        super.onDestroy()
    }
}
```

Adjust `environment` and `dataCenter` to match your WalkMe environment.

## 5. Integration checklist

1. Add **JitPack** to repositories.
2. Add **`walkme-android-sdk-editor`** with your release version.
3. If the app is **not** Compose-based, add the **Compose** dependencies in §3.
4. Obtain **`systemGuid`**, **`environment`**, and **`dataCenter`** from your WalkMe project / onboarding.
5. Call **`start`** once per process when ready; call **`stop`** before starting again or when tearing down.
6. Wire **`setUserId`** / **`setVariable`** after login and clear on logout if your policy requires it.

---

**Related:** For the core SDK only (no Power Mode), see [Walkme-Android-Sdk](https://github.com/WalkMe-int/walkme-android-sdk). Do not add both artifacts at the same time.
