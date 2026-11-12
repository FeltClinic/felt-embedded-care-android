# Felt Embedded Care — Android integration guide

This guide takes an Android app from no SDK to a working care experience: install, initialize, sign
the user in, show care screens, and handle deep links and push. It ends with what differs from the
iOS integration, for teams shipping both.

It describes **1.2.0**, and every API named here is public in that version. For every public class
and method, see the [API reference](https://feltclinic.github.io/felt-embedded-care-android/).
Upgrading from `1.1.2`? Read [Upgrading from 1.1.2](README.md#upgrading-from-112) first: one Sentry setup
can crash at launch. Upgrading from `1.0.0-beta.6`? Read [Upgrading](README.md#upgrading-from-100-beta6):
initialization changed and several classes moved packages.

## Requirements

| | |
|---|---|
| `minSdk` | 24 (Android 7.0) |
| `compileSdk` | 34 or higher |
| Host activity | An `AppCompatActivity`. `initialize` and `prepareActivity` do not accept a plain `Activity` or `ComponentActivity`. |
| Android Gradle Plugin | 8.5.1 or newer, so native libraries are packaged for 16 KB memory pages, which Google Play requires. |
| Kotlin | 2.0 or newer. The SDK is built with Kotlin 2.1, and older compilers can't read its metadata. |
| UI toolkit | Jetpack Compose or Views. Both are supported; the Compose entry points need Compose in your app. |

**Permissions.** The SDK declares none of its own. Its networking dependencies bring
`android.permission.INTERNET` and `android.permission.ACCESS_NETWORK_STATE` into your merged
manifest. The SDK needs no storage, camera or vibration permission. For notifications on Android 13
and later, you declare `POST_NOTIFICATIONS` yourself (section 6).

## 1. Install

The SDK is on Maven Central. One of its dependencies resolves from JitPack, so add both repositories:

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven("https://jitpack.io")
    }
}
```

```kotlin
// app/build.gradle.kts
dependencies {
    implementation("clinic.felt:embeddedcare:1.2.0")
}
```

The current version is **1.2.0**. At runtime, `EmbeddedCare.VERSION_NAME` reports the version you
are running.

**R8 / ProGuard:** no rules are needed from you. The parts of the SDK that are reached by reflection
are covered by the consumer rules its dependencies ship, which Gradle applies to your app
automatically. Felt checks this with a minified build on every change.

## 2. Get an SDK key

Felt gives you an SDK key for each environment (for example staging and production). The key tells
the SDK which Felt environment to load, so use the staging key in debug and test builds and the
production key in release builds. Ask your Felt contact for the keys to use in your Android app.

Treat the key like any other app configuration value, for example a `BuildConfig` field per build
type.

## 3. Initialize

```kotlin
import clinic.felt.embeddedcare.EmbeddedCare
import clinic.felt.embeddedcare.Error as FeltError

class App : Application() {
    val appScope = MainScope()
}

object FeltStartup {
    private var started = false
    val isReady = MutableStateFlow(false)

    fun start(activity: AppCompatActivity) {
        if (started) return
        started = true
        (activity.application as App).appScope.launch {
            try {
                EmbeddedCare.initialize(activity, sdkKey = BuildConfig.FELT_SDK_KEY)
                isReady.value = true
            } catch (e: FeltError.InvalidConfiguration) {
                started = false
                reportToYourCrashReporter(e)
                return@launch
            }
            // If the user is already signed in, set the account here (section 4),
            // then replay any link that arrived during startup (section 6).
        }
    }
}

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        EmbeddedCare.prepareActivity(this)
        FeltStartup.start(this)
        setContent { /* … */ }
    }
}
```

Six rules:

1. **Host the SDK from one activity, and call `EmbeddedCare.prepareActivity(this)` in its `onCreate`
   every time it is created**, including after rotation. It registers the system notification
   permission prompt, and Android only allows that before the activity reaches `STARTED`. The SDK
   keeps one registration, for the activity that called it last, so present care screens and the
   permission prompt from that same activity.
2. **Initialize once per process, from a scope that outlives the activity.** Calling `initialize`
   again replaces the SDK instance, and the signed-in account goes with it. Starting takes a moment,
   and `EmbeddedCare.isInstanceInitialized` only becomes `true` once it has finished, so it can't stop
   a second start on its own: a rotation in that window would start again. Keep your own flag, as
   `FeltStartup` does, and launch from an application-level scope so a rotation doesn't cancel it.
3. **Call `initialize` on the companion** (`EmbeddedCare.initialize(…)`), never through
   `EmbeddedCare.shared`. `EmbeddedCare.shared` throws `IllegalStateException` until `initialize` has
   finished, so check `EmbeddedCare.isInstanceInitialized` before touching it anywhere that might run
   early.
4. **Catch `Error.InvalidConfiguration` from `initialize`.** It means the SDK key can't be read. The
   SDK's `Error` shares its simple name with Kotlin's `Error`, hence the import alias above.
5. **Show SDK screens only once the SDK has started.** `FullScreenView`, `FullScreenFragment`,
   `Component` and `ComponentFragment` read `EmbeddedCare.shared` as soon as they are created or
   shown, so showing one earlier crashes. Wait for `FeltStartup.isReady` (section 5).
6. **Call the SDK from the main thread.** `MainScope()` runs on the main thread. Callbacks that
   arrive on other threads, such as Firebase's, need to post to it (section 6).

## 4. Sign the user in

The SDK learns who the user is from an `Account` you set on `EmbeddedCare.shared.auth`:

```kotlin
import clinic.felt.embeddedcare.models.Account

EmbeddedCare.shared.auth.account = Account(
    id = user.id,
    firstName = user.firstName,
    lastName = user.lastName,
    email = user.email,
    custom = mapOf(
        "yourTokenKey" to session.token,
    ),
)
```

| Field | Type | Notes |
|---|---|---|
| `id` | `String` | **Required.** Your stable identifier for this user. |
| `firstName`, `lastName` | `String?` | |
| `email` | `String?` | |
| `birthDate` | `kotlinx.datetime.LocalDate?` | A date, with no time or zone. The SDK brings `kotlinx-datetime` onto your compile classpath for this. |
| `selectedPatient` | `Patient?` | The person care is currently for, when that is not the account holder. |
| `coveredPersons` | `List<Patient>?` | Everyone covered by the account. |
| `sessionToken` | `String?` | Leave unset unless Felt asks you to use it. |
| `custom` | `Map<String, String>?` | Partner-specific values, including your auth tokens. |

`Patient` (`clinic.felt.embeddedcare.models.Patient`) takes an `id` plus optional name, contact,
`birthDate`, `gender`, `genderIdentity`, `preferredLanguage`, `profilePicture`,
`relationshipToAccountHolder` and `custom` fields.

### The selected patient

The SDK treats the person currently using your app as the patient. That may be the account holder,
or someone else on the account, such as a parent's child. Set `selectedPatient` to say who it is:

```kotlin
import clinic.felt.embeddedcare.models.Patient

EmbeddedCare.shared.auth.account = EmbeddedCare.shared.auth.account?.copy(
    selectedPatient = Patient(
        id = familyMember.id,
        relationshipToAccountHolder = Patient.RelationshipToAccountHolder.FamilyMember,
        email = familyMember.email,
    ),
)
```

If your data model gives the account holder and the patient different ids, keep them different here
too, even when they are the same person.

### Passing your auth tokens

If your integration authenticates the care experience with tokens from your own backend, Felt
agrees the token key names with you during onboarding. `yourTokenKey` above is a placeholder; use
the names Felt gives you. Put the tokens in `custom` under those names. The care
experience reads them from there and exchanges them with Felt's backend. Your app never calls a Felt
endpoint itself.

**When your tokens change, assign a new `Account`.** Screens that are already open receive the new
account without reloading. Setting an account equal to the current one does nothing, so building a
new `Account` with the same values on every resume is harmless.

```kotlin
EmbeddedCare.shared.auth.account = currentAccount.copy(
    custom = currentAccount.custom.orEmpty() + ("yourTokenKey" to refreshedToken),
)
```

**On sign-out, clear it:** `EmbeddedCare.shared.auth.account = null`.

If you need to react to account changes yourself, collect `EmbeddedCare.shared.auth.accountChanges`.
It emits each new value, not the current one, so read `EmbeddedCare.shared.auth.account` first.

## 5. Show care experiences

An experience is a route in Felt's care app. The built-in ones are `FullScreenExperience.HomeScreen`,
`FullScreenExperience.PricingScreen`, `FullScreenExperience.SplashScreen` and
`FullScreenExperience.ValuePropositionScreen`. Any other route is `FullScreenExperience("/route")`.

### As a modal, presented by the SDK

```kotlin
EmbeddedCare.shared.presentExperience(FullScreenExperience.PricingScreen)
EmbeddedCare.shared.presentOverlay(FullScreenExperience("/chat"))
```

The SDK adds the screen on top of your current activity and removes it when the user, or the care
experience, closes it. System back steps through the experience's own history first, then closes it.

Both calls need an activity in the foreground. A call made while your app is in the background does
nothing, so present in response to something the user did.

### Inline, in Compose

`FullScreenView` hosts an experience inside your own layout, typically as a tab:

```kotlin
import clinic.felt.embeddedcare.features.fullscreen.FullScreenView

@Composable
fun CareTab(bottomBarHeight: Dp) {
    val isReady by FeltStartup.isReady.collectAsState()
    if (!isReady) {
        CareTabPlaceholder()
        return
    }
    FullScreenView(
        experience = FullScreenExperience.HomeScreen,
        modifier = Modifier.fillMaxSize(),
        extraBottomSafeArea = bottomBarHeight,
    )
}
```

- **Compose it only once the SDK has started.** Before that, `FullScreenView` throws. The same goes
  for `FullScreenFragment`, `Component` and `ComponentFragment`.
- **Pass `Modifier.fillMaxSize()`.** With an unconstrained width the view lays out but taps do not
  reach it.
- **`extraBottomSafeArea`** tells the experience how much of its bottom edge your own navigation bar
  covers, so it can keep its content clear of it. Leave it at `0.dp` if nothing overlaps.
- The view resizes itself for the keyboard. Don't add your own IME padding around it.
- When the care experience closes something inside an inline view, the view goes back to the care
  home screen. It never removes itself from your layout.

### Inline, with Views

Use `FullScreenFragment` in your own container:

```kotlin
import clinic.felt.embeddedcare.features.fullscreen.FullScreenFragment

supportFragmentManager.beginTransaction()
    .replace(R.id.care_container, FullScreenFragment.create(FullScreenExperience.HomeScreen))
    .commit()
```

If you present a `FullScreenFragment` modally yourself, create it with
`FullScreenFragment.create(experience, includeDismissButton = true)`. That is what tells it it is a
modal: it shows a close button, and closes itself when the care experience is done. Without it, a
close from the care experience sends the user to the care home screen instead.

### Small embedded components

Components are compact care entry points that size themselves to their content:

```kotlin
import clinic.felt.embeddedcare.features.component.Component

Component(
    experience = ComponentExperience.TriagedCallToAction,
    modifier = Modifier.fillMaxWidth(),
    onDismiss = { hideCareCard() },
)
```

In a View hierarchy, use `ComponentFragment.create(ComponentExperience.DismissableCallToAction, mapOf())`.

A component's content loads asynchronously, so its height changes over time. Give its container a
height that can grow (`wrap_content`, or no fixed height in Compose). A fixed or minimum height clips
the content or leaves a gap.

### Feature gates

To check whether a feature is turned on for the current user, for example before showing a
component, call `checkFeatureGate`:

```kotlin
import clinic.felt.embeddedcare.FeatureGate
import clinic.felt.embeddedcare.checkFeatureGate

if (EmbeddedCare.shared.checkFeatureGate(FeatureGate.GlobalRollout)) {
    showCareCard()
}
```

A gate that doesn't exist reads as `false`.

### Deciding how experiences open

To control presentation yourself, for example to switch to your care tab instead of opening a modal,
set `fullScreenExperiencePresentationHandler`. Return `true` if you presented the experience, or
`false` to let the SDK present it.

```kotlin
EmbeddedCare.shared.fullScreenExperiencePresentationHandler = { context ->
    when (context.suggestedPresentationMethod) {
        PresentationMethod.Home -> {
            showCareTab(context.experience)
            true
        }
        else -> false
    }
}
```

`PresentationMethod` is `FullScreenPresentationContext.PresentationMethod`. It is one of `Home`,
`Modal`, `Overlay` or `TransparentFullScreen`. `context.experience.route` tells you which route is
being opened.

## 6. Deep links and push notifications

### Link format

```
https://<your-felt-domain>?felt-route=/chat&felt-method=modal
```

| Parameter | |
|---|---|
| `felt-route` | **Required.** The route to open. |
| `felt-method` | Optional: `modal` (default), `home` or `overlay`. |

Push notifications carry the same two keys in their data payload.

### Hand links to the SDK

Once the SDK is running, it opens a link as soon as you hand it over, and it can only open one into
a **resumed** activity. A link handed over earlier, from `onCreate` or `onNewIntent` while the app is
coming back from the background, is used up without being shown. So hold the intent and hand it
over in `onResume`:

```kotlin
class MainActivity : AppCompatActivity() {

    private var pendingFeltIntent: Intent? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        EmbeddedCare.prepareActivity(this)
        if (savedInstanceState == null) {
            pendingFeltIntent = intent
        }
        FeltStartup.start(this)
    }

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        pendingFeltIntent = intent
    }

    override fun onResume() {
        super.onResume()
        pendingFeltIntent?.let { handleFeltIntent(it) }
        pendingFeltIntent = null
        replayStoredFeltLinks()
    }

    private fun handleFeltIntent(intent: Intent) {
        intent.data?.let { EmbeddedCare.handleURL(it) }
        intent.extras?.let { EmbeddedCare.handleNotification(it) }
    }
}
```

Use the companion forms, `EmbeddedCare.handleURL` and `EmbeddedCare.handleNotification`. They are
safe before `initialize`: a link that arrives before the SDK is running is stored rather than
dropped. Both return `false` for anything that is not a Felt link, so you can call them on every
intent. The `savedInstanceState` check stops a recreated activity from handling its launch intent
twice.

### Cold start: replay stored links

A link that arrives before the SDK is running is stored until you ask for it. Call
`processStoredDeeplinks()` only when **the SDK is running, the account is set, and your activity is
resumed**:

```kotlin
fun replayStoredFeltLinks() {
    if (EmbeddedCare.isInstanceInitialized && EmbeddedCare.shared.auth.account != null) {
        EmbeddedCare.shared.processStoredDeeplinks()
    }
}
```

Call it from `onResume`, as above, and again right after you set the account, if that happens while
the activity is already resumed (for example at the end of `FeltStartup`, or when sign-in finishes).

Each condition matters, because a replay that fails one of them uses the link up:

- **No account:** most routes are blocked. The link is discarded and `deeplinkBlockedHandler` is
  called with its route. Set the handler to send the user to sign in:
  `EmbeddedCare.shared.deeplinkBlockedHandler = { route -> showSignIn() }`. With no handler set,
  nothing visible happens. The same applies to a link handed over while no account is set.
- **Activity not resumed:** the experience can't be shown, and the link is not kept.
- **UI still being built:** an experience presented into a screen that is still being set up can be
  closed again by your own navigation.

Calling `processStoredDeeplinks()` with nothing stored does nothing, so it is safe to call
repeatedly.

### Push notifications (FCM)

The SDK does not depend on Firebase. Your app keeps its own `FirebaseMessagingService` and passes
the SDK what it needs:

```kotlin
class AppMessagingService : FirebaseMessagingService() {

    override fun onNewToken(token: String) {
        Handler(Looper.getMainLooper()).post {
            if (EmbeddedCare.isInstanceInitialized) {
                EmbeddedCare.shared.notifications.registerDeviceToken(token)
            }
        }
    }
}
```

- Firebase calls `onNewToken` on a background thread, so post to the main thread before calling the
  SDK.
- `registerDeviceToken(token)` takes the FCM token as a `String`. It registers the token for the
  signed-in account; a token received before an account is set is registered when the account is
  set. Also pass the current token after initializing, for example from
  `FirebaseMessaging.getInstance().token`, because `onNewToken` only fires when the token changes.
- **Hand a notification to the SDK when the user taps it, not when it arrives.** A notification
  tapped while your app is in the background opens your launcher activity with the data payload as
  intent extras, which `handleFeltIntent` above already covers. For a message received while your
  app is in the foreground, post the notification yourself and copy `felt-route` and `felt-method`
  from `RemoteMessage.data` into the extras of its tap intent. If you hold the payload as a
  `Map<String, String>` instead, `EmbeddedCare.handleNotification(map)` accepts it directly.

### Notification permission

On Android 13 and later, notifications need the user's permission. Declare it in your manifest; the
SDK does not declare it for you:

```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

Without that line, Android refuses the request without showing a prompt.

To show Felt's explanation before the system prompt:

```kotlin
import clinic.felt.embeddedcare.features.pushnotifications.ui.presentPermissionRequest

EmbeddedCare.shared.pushNotifications.presentPermissionRequest(
    activity = this,
    onPermissionGranted = { /* … */ },
)
```

The SDK shows the prompt from the activity currently on screen, and only if that activity called
`EmbeddedCare.prepareActivity(this)` in `onCreate`. If it didn't, no prompt appears and the SDK logs an
error. Call `prepareActivity` in every activity that can show the prompt.

## 7. Lifecycle

| Situation | What the SDK does | What you do |
|---|---|---|
| Activity created | Nothing until you call it. | `EmbeddedCare.prepareActivity(this)` in `onCreate`, every time. Hold the intent for `onResume`. |
| Rotation and other configuration changes | Screens it presented, and `FullScreenView` / `FullScreenFragment` instances, are restored along with your activity's other fragments, as long as your app's process is still running. The SDK follows whichever activity is resumed. | Call `prepareActivity` again. Don't start the SDK again, and don't handle the launch intent again. |
| App goes to the background | Open screens stay as they are. New presentations are ignored until you come back. | Nothing. |
| App returns to the foreground | Open screens resume. | In `onResume`, hand over any new intent, then replay stored links if an account is set. |
| User signs out | Open screens receive the cleared account. | `EmbeddedCare.shared.auth.account = null`. |

**Back button:** while a care screen is showing, system back moves back through the experience's
history, then closes a modal. An inline `FullScreenView` hands back to your app when it has no more
history. Inline components never handle back.

## 8. Events from the care experience

```kotlin
EmbeddedCare.shared.events.onWebViewDidFinishLoading { event -> /* a care screen finished loading */ }
EmbeddedCare.shared.events.onFeltSubscriptionCompleted { event -> /* … */ }
EmbeddedCare.shared.events.onTermsAcceptance { event -> /* … */ }
```

Each event carries a `timestamp`. The SDK keeps one handler per event, so registering again replaces
the previous handler. Register each in one place and fan out from there.

### Hiding your tab bar

Some care screens ask for your tab bar to be hidden. In Compose, read
`EmbeddedCare.shared.tabBarManager.isTabBarVisible` (a `State<Boolean>`) and hide your navigation bar
when it is `false`.

In an app built with Views, collect `tabBarManager.visibility`, a `StateFlow<Boolean>` that always
holds the same value:

```kotlin
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        EmbeddedCare.shared.tabBarManager.visibility.collect { visible ->
            bottomNavigationView.isVisible = visible
        }
    }
}
```

### Tab badges

If Felt has enabled badges for your app, one call keeps a care tab's badge up to date:

```kotlin
import clinic.felt.embeddedcare.features.badge.ui.takeOverBadging

EmbeddedCare.shared.takeOverBadging(this)
```

It finds your bottom navigation bar and badges the care tab. Pass an
`AutomaticTabBadging.Configuration` as the second argument to change the badge style.

To draw the badge yourself, read `EmbeddedCare.shared.badges`: `isBadgeVisible`, `badgeCount` and
`badgeType`, the same names as in the iOS SDK. Register `addStateChangeHandler` to hear about changes.
Badge updates pause while your app is in the background and resume when it returns.

## 9. Errors

The SDK throws `clinic.felt.embeddedcare.Error`, a sealed type:

| Case | Thrown when |
|---|---|
| `Error.InvalidConfiguration` | `initialize` can't read the SDK key. `isAccountActive()` also throws it when the SDK has not finished starting. |
| `Error.NotInitialized` | An SDK method runs on an instance whose setup has not finished. |
| `Error.AccountNotSet` | You call an API that needs an account, with none set. |
| `Error.AccountStatusDisabled` | You call `isAccountActive()` and account status is not enabled for your app. |
| `Error.NetworkError` | An account status request times out. |

`EmbeddedCare.shared` itself throws Kotlin's `IllegalStateException`, not an SDK `Error`, when it is
read before `initialize` has finished. Check `EmbeddedCare.isInstanceInitialized` rather than
catching it.

## 10. Differences from the iOS integration

The two SDKs share their concepts and most names. These are the places where Android differs.

| | iOS | Android |
|---|---|---|
| Initialize | `EmbeddedCare.initialize(sdkKey:)`, `async throws` | `EmbeddedCare.initialize(activity, sdkKey)`, `suspend`, needs an `AppCompatActivity` |
| Per-activity setup | None | `EmbeddedCare.prepareActivity(activity)` in the hosting activity's `onCreate` |
| Account | `EmbeddedCare.shared.auth.account` | Same |
| `Account.birthDate` | `Date` | `kotlinx.datetime.LocalDate`, a date with no time |
| Push token | APNs device token (`Data`) | FCM token as a `String`: `EmbeddedCare.shared.notifications.registerDeviceToken(token)` |
| Notification payload | `handleNotification(_: UNNotification)` | `EmbeddedCare.handleNotification(bundle)` for intent extras, or `EmbeddedCare.handleNotification(map)` for FCM `RemoteMessage.data` |
| Links | `EmbeddedCare.handleURL(_: URL)` | `EmbeddedCare.handleURL(uri)`, or `EmbeddedCare.handleURL(string)` |
| Deep link timing | `processStoredDeeplinks()` | Hand them over in `onResume`, and call `processStoredDeeplinks()` yourself once the account is set (section 6) |
| Blocked link, no handler set | Shows a "Login Required" alert | Nothing visible. Set `deeplinkBlockedHandler`. |
| Rotation, process lifecycle | Not applicable | See section 7 |
| Inline full screen | SwiftUI `FullScreenView` | Compose `FullScreenView`, or `FullScreenFragment` for Views |
| SDK version | | `EmbeddedCare.VERSION_NAME` and `EmbeddedCare.VERSION_CODE` |

## Known issues

- **Restoring after Android ends your app's process:** if Android ends your app's process in the
  background while an SDK screen is open, reopening the app restores that screen before the SDK has
  started, and the app crashes. Rotation and ordinary backgrounding are not affected, because the
  process keeps running.

## Getting help

Contact your Felt integration team with your SDK version (`EmbeddedCare.VERSION_NAME`), device and
Android version, and steps to reproduce.
