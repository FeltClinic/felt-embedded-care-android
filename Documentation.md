# Felt Embedded Care Android SDK
Felt offers a seamless telehealth experience inside health apps. Our platform handles the full user lifecycle, including onboarding, payments, and doctor visits, ensuring a cohesive healthcare experience.

## Initialization
Primary interaction with the SDK is mediated through the [EmbeddedCare.shared](https://www.javadoc.io/doc/clinic.felt/embeddedcare/latest/clinic/felt/embeddedcare/EmbeddedCare.Companion.html#getShared()) singleton.

At app launch, initialize the SDK with your SDK key using [`EmbeddedCare.shared.initialize`](https://www.javadoc.io/doc/clinic.felt/embeddedcare/latest/clinic/felt/embeddedcare/EmbeddedCare.html#initialize(AppCompatActivity,String)).

```kotlin
import clinic.felt.embeddedcare.EmbeddedCare

val feltSDKKey = "..."

class MainActivity: AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // other initialization logic

        val activity = this
        EmbeddedCare.shared.scope.launch {
            // Error will be thrown for invalid key.
            // Ensure that this is only called once per app launch.
            EmbeddedCare.shared.initalize(activity, sdkKey = /* your Felt SDK Key */)
        }
    }
}
```

## User Account and Patient Details
### Account
Once your app's current user has been authenticated, provide their account details to the SDK using [`EmbeddedCare.shared.account`](https://www.javadoc.io/doc/clinic.felt/embeddedcare/latest/clinic/felt/embeddedcare/EmbeddedCare.html#setAccount(Account)) .
```kotlin
val user = ...
EmbeddedCare.shared.account = Account(id = user.id, email = user.email)
```

When the user logs out, inform the SDK by setting `EmbeddedCare.shared.account` to `null`.
```kotlin
EmbeddedCare.shared.account = null
```

### Selected Patient
The SDK considers the logged in person currently using your app to be a `patient`.
Your user's account may be the same person or a distinct person from the actual patient using your app. 
For example, the account holder may be a parent and the patient may be their child.
Even if the account holder and patient represent the same person, take care to distinguish their ids 
if your app's data model keeps them distinct.

Set the [selectedPatient](https://www.javadoc.io/doc/clinic.felt/embeddedcare/latest/clinic/felt/embeddedcare/Account.html#selectedPatient) property to provide patient details to the SDK for the currently logged in patient.
```kotlin
val currentFamilyMember = ...
EmbeddedCare.shared.account = EmbeddedCare.shared.account?.copy(selectedPatient = Patient(
    id = currentFamilyMember.id,
    relationshipToAccountHolder: Patient.RelationshipToAccountHolder.FamilyMember,
    email = currentFamilyMember.email,
    // Any custom information you want to provide
    custom = mapOf("notes" to currentFamilyMember.notes)
))
```

If any information changes about the account or the selected patient, just update `EmbeddedCare.shared.account` with the latest information.

## Component Experiences
In order to integrate Felt embedded care content alongside your app's native content, use [ComponentFragment](https://www.javadoc.io/doc/clinic.felt/embeddedcare/latest/clinic/felt/embeddedcare/ComponentFragment.html), a `Fragment` subclass, or [Component](https://www.javadoc.io/doc/clinic.felt/embeddedcare/latest/clinic/felt/embeddedcare/ComponentFragmentKt.html#Component(ComponentExperience,Modifier,Map,Function0)), a Composable function.

See [ComponentExperience](https://www.javadoc.io/doc/clinic.felt/embeddedcare/latest/clinic/felt/embeddedcare/ComponentExperience.Companion.html) for available options.

### Fragment
```kotlin
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
Component contents may be loaded asynchronously, which means a component's layout can change over time. A component determines its intrinsic height based on its current width.

## Full Screen Experiences
Felt provides several full screen experiences you can explicitly present. See [FullScreenExperience](https://www.javadoc.io/doc/clinic.felt/embeddedcare/latest/clinic/felt/embeddedcare/FullScreenExperience.Companion.html) for available options.
```kotlin
EmbeddedCare.shared.presentExperience(FullScreenExperience.ValuePropositionScreen)
```
User interaction with components can automatically present full screen experiences.

Alternately, you can explicitly present full screen experiences using [FullScreenFragment](https://www.javadoc.io/doc/clinic.felt/embeddedcare/latest/clinic/felt/embeddedcare/FullScreenFragment.html).

### Fragment
```kotlin
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

The SDK will automatically modally present full screen experiences in response to user actions such as tapping a component. 

## Feature Gating
You can check if a feature is enabled for the current user by calling [EmbeddedCare.shared.checkFeatureGate](https://www.javadoc.io/doc/clinic.felt/embeddedcare/latest/clinic/felt/embeddedcare/BusinessOptimizationKt.html#checkFeatureGate(EmbeddedCare,FeatureGate)). See [FeatureGate](https://www.javadoc.io/doc/clinic.felt/embeddedcare/latest/clinic/felt/embeddedcare/FeatureGate.html) for available options.

For example, before displaying a component, you should call `EmbeddedCare.shared.checkFeatureGate` with the relevant feature gate to make sure the component should be displayed. 

```kotlin
if (EmbeddedCare.shared.checkFeatureGate(.globalRollout)) {
    // View logic to display component
}
```

## Deep Linking
The SDK can handle certain deep-links to embedded care content through either urls or push notifications. 
In your open-url and push notification handling code, call [EmbeddedCare.shared.handleURL](https://www.javadoc.io/doc/clinic.felt/embeddedcare/latest/clinic/felt/embeddedcare/EmbeddedCare.html#handleURL(String)) 
or [EmbeddedCare.shared.handleNotification](https://www.javadoc.io/doc/clinic.felt/embeddedcare/latest/clinic/felt/embeddedcare/EmbeddedCare.html#handleNotification(Bundle)) respectively to give the SDK an opportunity to respond to a deep-link event.

## Compatiblity
The SDK has minimum version requirements of Android SDK 24.
