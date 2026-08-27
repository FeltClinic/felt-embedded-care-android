# Felt Embedded Care Android SDK
Felt offers a seamless telehealth experience inside health apps. Our platform handles the full user lifecycle, including onboarding, payments, and doctor visits - with just a few lines of code.

## Getting Started
The SDK is published on [Maven Central](https://central.sonatype.com/artifact/clinic.felt/embeddedcare).

> These docs describe **1.1.0**. If you are upgrading from `1.0.0-beta.6`, initialization has
> changed and several classes have moved packages — see [Upgrading from 1.0.0-beta.6](#upgrading-from-100-beta6).
> Pin an explicit version rather than tracking the latest.

In your build script:
```kotlin
dependencyResolutionManagement {
    repositories {
        mavenCentral()
        // Statsig, a transitive dependency, publishes to JitPack rather than Maven Central.
        maven("https://jitpack.io")
    }
}

dependencies {
    // Add embedded care as a dependency
    implementation("clinic.felt:embeddedcare:${embeddedCareVersion}")
}
```

At app launch, prepare the activity and initialize the SDK with your SDK key.

```diff
+ import clinic.felt.embeddedcare.EmbeddedCare

class MainActivity: AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // other initialization logic

+        // Registers the runtime push-permission launcher. Must run in onCreate:
+        // registerForActivityResult throws once the activity reaches STARTED.
+        EmbeddedCare.prepareActivity(this)
+
+        lifecycleScope.launch {
+            // Error will be thrown for an invalid key.
+            EmbeddedCare.initialize(this@MainActivity, sdkKey = /* your Felt SDK Key */)
+        }
    }
}
```

Two things there are easy to get wrong:

* **Call `initialize` on the companion, not on `.shared`.** `EmbeddedCare.shared` throws
  `IllegalStateException` until the first `initialize` completes, so
  `EmbeddedCare.shared.initialize(...)` cannot work — the receiver throws before the call happens.
  Guard any early access with `EmbeddedCare.isInstanceInitialized`.
* **`prepareActivity` must be called from `onCreate`.** It registers an `ActivityResultLauncher`,
  and `registerForActivityResult` throws if the activity has already reached `STARTED`.

Now you can embed care experiences, below are a few examples.
See [full documentation](Documentation.md#Component-Experiences) for a complete list of approaches.

```kotlin
import clinic.felt.embeddedcare.ComponentExperience
import clinic.felt.embeddedcare.FullScreenExperience
import clinic.felt.embeddedcare.features.component.ComponentFragment
import clinic.felt.embeddedcare.features.fullscreen.FullScreenFragment

// for a modal care experience
EmbeddedCare.shared.presentExperience(FullScreenExperience.ValuePropositionScreen)

// alternatively, FullScreenFragment and FullScreenView offer flexibility to present the experience
val fragmentManager = activity?.supportFragmentManager ?: return
val fragment = FullScreenFragment.create(FullScreenExperience.PricingScreen, includeDismissButton = true)
fragmentManager
    .beginTransaction()
    .setTransition(FragmentTransaction.TRANSIT_FRAGMENT_OPEN)
    .add(android.R.id.content, fragment)
    .addToBackStack(null)
    .commit()

// ComponentFragment and Component can be shown alongside your existing views
val fragmentManager = activity?.supportFragmentManager ?: return
val fragment = ComponentFragment.create(ComponentExperience.DismissableCallToAction)
fragmentManager
    .beginTransaction()
    .add(containerViewId, fragment)
    .commit()
```

Finally, you will need to share context with the SDK to have a complete experience. This can be achieved by

* setting account and patient information with `EmbeddedCare.shared.auth.account`
* setting component custom data with `ComponentFragment.create(..., custom = ...)`

🎉 That's it! By working with Felt, we will populate these views with a customized, white-labeled experience that will delight your users.

## Upgrading from 1.0.0-beta.6

`1.0.0-beta.6` was published in January 2025 and this release lands everything since. Three changes
require edits to an existing integration.

**Initialization moved to the companion.**

```diff
- EmbeddedCare.shared.scope.launch {
-     EmbeddedCare.shared.initialize(activity, sdkKey)
- }
+ EmbeddedCare.prepareActivity(activity)
+ lifecycleScope.launch {
+     EmbeddedCare.initialize(activity, sdkKey)
+ }
```

**The account moved behind `auth`.**

```diff
- EmbeddedCare.shared.account = Account(id = user.id, email = user.email)
+ EmbeddedCare.shared.auth.account = Account(id = user.id, email = user.email)
```

**Several classes moved packages.**

| Was | Now |
|---|---|
| `clinic.felt.embeddedcare.Account` | `clinic.felt.embeddedcare.models.Account` |
| `clinic.felt.embeddedcare.Patient` | `clinic.felt.embeddedcare.models.Patient` |
| `clinic.felt.embeddedcare.ComponentFragment` | `clinic.felt.embeddedcare.features.component.ComponentFragment` |
| `clinic.felt.embeddedcare.FullScreenFragment` | `clinic.felt.embeddedcare.features.fullscreen.FullScreenFragment` |

`ComponentExperience`, `FullScreenExperience`, `FeatureGate` and `EmbeddedCare` itself are unchanged.

## Requirements

| | |
|---|---|
| `minSdk` | 24 (Android 7.0) |
| Host activity | must be an `AppCompatActivity` — `initialize` and `prepareActivity` require one |
| Jetpack Compose | required; the SDK ships Compose UI |
| Repositories | Maven Central **and** JitPack (for Statsig) |

## Documentation
You can find further documentation [here](Documentation.md) and api reference documentation [here](https://www.javadoc.io/doc/clinic.felt/embeddedcare/latest/clinic/felt/embeddedcare/package-summary.html).

## Support

### Slack
We should have a Slack Connect setup you should be able to find us individually below or at `#<company name>-felt`

Reach out to sid (at) feltclinic.com or josh (at) feltclinic.com if you are not finding us on Slack.


### Email

* sid (at) feltclinic.com
* daniel (at) feltclinic.com
