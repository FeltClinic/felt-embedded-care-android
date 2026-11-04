# Felt Embedded Care Android SDK
Felt offers a seamless telehealth experience inside health apps. Our platform handles the full user lifecycle, including onboarding, payments, and doctor visits, ensuring a cohesive healthcare experience.

These docs describe **1.1.0**. Upgrading from `1.0.0-beta.6`? See
[Upgrading](README.md#upgrading-from-100-beta6) first — initialization changed and several classes
moved packages.

For every public class and method, see the
[API reference](https://feltclinic.github.io/felt-embedded-care-android/).

## Initialization
Primary interaction with the SDK is mediated through the `EmbeddedCare.shared` singleton. Creating
it is the companion's job.

```kotlin
import clinic.felt.embeddedcare.EmbeddedCare

class MainActivity: AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // other initialization logic

        // Registers the runtime push-permission launcher. registerForActivityResult throws
        // once the activity has reached STARTED, so this belongs in onCreate.
        EmbeddedCare.prepareActivity(this)

        lifecycleScope.launch {
            // Error will be thrown for an invalid key.
            // Calling initialize again replaces the instance.
            EmbeddedCare.initialize(this@MainActivity, sdkKey = /* your Felt SDK Key */)
        }
    }
}
```

`EmbeddedCare.shared` throws `IllegalStateException` until the first `initialize` completes:

```
EmbeddedCare.initialize(activity, sdkKey) must be called before accessing .shared
```

So `EmbeddedCare.shared.initialize(...)` cannot work — evaluating the receiver throws before the
call is made. Use the companion, and guard early access with `EmbeddedCare.isInstanceInitialized`
rather than a `try`/`catch`:

```kotlin
if (EmbeddedCare.isInstanceInitialized && EmbeddedCare.shared.isInitialized) {
    // safe to use
}
```

Initialization is asynchronous, so a host that navigates immediately can still reach `.shared`
while managers are settling.

Call the public API from the main thread. Fragment presentation, tab bar visibility and badge
application all touch views.

## User Account and Patient Details
### Account
Once your app's current user has been authenticated, provide their account details to the SDK using
`EmbeddedCare.shared.auth.account`.

```kotlin
import clinic.felt.embeddedcare.models.Account

val user = ...
EmbeddedCare.shared.auth.account = Account(id = user.id, email = user.email)
```

When the user logs out, inform the SDK by setting the account to `null`.
```kotlin
EmbeddedCare.shared.auth.account = null
```

### Selected Patient
The SDK considers the logged in person currently using your app to be a `patient`.
Your user's account may be the same person or a distinct person from the actual patient using your app.
For example, the account holder may be a parent and the patient may be their child.
Even if the account holder and patient represent the same person, take care to distinguish their ids
if your app's data model keeps them distinct.

Set the `selectedPatient` property to provide patient details for the currently logged in patient.
```kotlin
import clinic.felt.embeddedcare.models.Patient

val currentFamilyMember = ...
EmbeddedCare.shared.auth.account = EmbeddedCare.shared.auth.account?.copy(
    selectedPatient = Patient(
        id = currentFamilyMember.id,
        relationshipToAccountHolder = Patient.RelationshipToAccountHolder.FamilyMember,
        email = currentFamilyMember.email,
        // Any custom information you want to provide
        custom = mapOf("notes" to currentFamilyMember.notes),
    )
)
```

If any information changes about the account or the selected patient, just update
`EmbeddedCare.shared.auth.account` with the latest information.

## Component Experiences
In order to integrate Felt embedded care content alongside your app's native content, use
`ComponentFragment`, a `Fragment` subclass, or `Component`, a Composable function. Both live in
`clinic.felt.embeddedcare.features.component`.

See `ComponentExperience` for available options.

### Fragment
```kotlin
import clinic.felt.embeddedcare.ComponentExperience
import clinic.felt.embeddedcare.features.component.ComponentFragment

val fragmentManager = activity?.supportFragmentManager ?: return
val fragment = ComponentFragment.create(ComponentExperience.DismissableCallToAction)
fragmentManager
    .beginTransaction()
    .add(containerViewId, fragment)
    .commit()
...
```

### Compose
```kotlin
import clinic.felt.embeddedcare.features.component.Component

@Composable
fun ContentView() {
    Column {
        ...
        Component(ComponentExperience.DismissableCallToAction)
        ...
    }
}
```

### Layout
Component contents may be loaded asynchronously, which means a component's layout can change over
time. A component determines its intrinsic height based on its current width, so give its container
a height that can grow — a fixed or minimum height will clip content or leave a gap.

## Full Screen Experiences
Felt provides several full screen experiences you can explicitly present. See `FullScreenExperience`
for available options.

```kotlin
EmbeddedCare.shared.presentExperience(FullScreenExperience.ValuePropositionScreen)
```

User interaction with components can automatically present full screen experiences.

Alternately, you can explicitly present them using `FullScreenFragment`, in
`clinic.felt.embeddedcare.features.fullscreen`.

### Fragment
```kotlin
import clinic.felt.embeddedcare.FullScreenExperience
import clinic.felt.embeddedcare.features.fullscreen.FullScreenFragment

val fragmentManager = activity?.supportFragmentManager ?: return
val fragment = FullScreenFragment.create(FullScreenExperience.PricingScreen, includeDismissButton = true)
fragmentManager
    .beginTransaction()
    .setTransition(FragmentTransaction.TRANSIT_FRAGMENT_OPEN)
    .add(android.R.id.content, fragment)
    .addToBackStack(null)
    .commit()
...
```

### Taking over presentation
By default the SDK presents full screen experiences itself, in response to user actions such as
tapping a component. Set a handler to route them into your own UI instead — a tab, say, rather than
a modal over it.

```kotlin
EmbeddedCare.shared.fullScreenExperiencePresentationHandler = { context ->
    when (context.suggestedPresentationMethod) {
        // Return false to let the SDK present this one itself.
        FullScreenPresentationContext.PresentationMethod.Modal -> false
        else -> {
            showCareTab(context.experience)
            true   // this app has taken responsibility for presenting it
        }
    }
}
```

Returning `false`, or setting no handler, leaves the SDK's own presentation in place.

## Events
`EmbeddedCare.shared.events` reports what is happening inside embedded content.

```kotlin
// Fires each time a Felt web view finishes loading. Useful for holding UI back
// until embedded content is ready — enabling a tab, dismissing a spinner.
EmbeddedCare.shared.events.onWebViewDidFinishLoading {
    careTab.isEnabled = true
}

EmbeddedCare.shared.events.onFeltSubscriptionCompleted { event ->
    // The user completed a subscription.
}

EmbeddedCare.shared.events.onTermsAcceptance { event ->
    // The user needs to accept new terms.
}
```

Each handler replaces any previous one — the SDK keeps a single handler per event — so register in
one place and fan out from there rather than registering twice.

## Tab Bar Visibility
If your app has a tab bar, the web app can ask for it to be hidden on screens where it does not
belong. `EmbeddedCare.shared.tabBarManager.isTabBarVisible` is a Compose `State<Boolean>` carrying
the current answer.

```kotlin
val isVisible by EmbeddedCare.shared.tabBarManager.isTabBarVisible
```

Reading it outside a composition needs the snapshot system to be pumped — in a View-based host with
no composition running, nothing calls `Snapshot.sendApplyNotifications()`, and `snapshotFlow` will
emit once and then stay silent. Either observe it from a composition, or register a global write
observer that posts `sendApplyNotifications()` on the main thread.

## Feature Gating
You can check if a feature is enabled for the current user by calling
`EmbeddedCare.shared.checkFeatureGate`. See `FeatureGate` for available options.

For example, before displaying a component, call it with the relevant gate to make sure the
component should be displayed.

```kotlin
import clinic.felt.embeddedcare.FeatureGate
import clinic.felt.embeddedcare.checkFeatureGate

if (EmbeddedCare.shared.checkFeatureGate(FeatureGate.GlobalRollout)) {
    // View logic to display component
}
```

Gates that do not exist read as `false`.

## Deep Linking
The SDK can handle certain deep-links to embedded care content through either urls or push
notifications. In your open-url and push notification handling code, call `EmbeddedCare.handleURL`
or `EmbeddedCare.handleNotification` respectively to give the SDK an opportunity to respond.

```kotlin
override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    EmbeddedCare.handleNotification(intent.extras ?: return)
}
```

Links that arrive before an account is set are queued rather than dropped. Replaying them is the
host's job, once your UI is stable:

```kotlin
override fun onResume() {
    super.onResume()
    EmbeddedCare.shared.processStoredDeeplinks()
}
```

Calling it too early is the failure mode to avoid — an experience presented into a half-built UI
gets dismissed by your own navigation, and the link is consumed. Call it once the UI has settled,
and again after any asynchronous bootstrap finishes. Repeated calls are safe; an empty queue is a
no-op.

## Error Handling
The SDK throws a sealed `Error` hierarchy, so a host can respond per case.

| Type | Thrown when |
|---|---|
| `Error.NotInitialized` | an API needing a live SDK is called before `initialize` |
| `Error.InvalidConfiguration` | the SDK key fails to decode, or configuration is malformed |
| `Error.AccountNotSet` | an account-dependent API is called with no account assigned |
| `Error.AccountStatusDisabled` | `isAccountActive()` is called while the account status gate is off |
| `Error.NetworkError` | an account-status request times out |

## Compatibility

| | |
|---|---|
| `minSdk` | 24 (Android 7.0) |
| Host activity | must be an `AppCompatActivity` |
| Jetpack Compose | required; the SDK ships Compose UI |
| Repositories | Maven Central **and** JitPack — Statsig is not on Central |

R8/ProGuard needs no rules of your own. The SDK's reflective surfaces are the Capacitor bridge and
kotlinx.serialization, and both ship their own consumer rules.
