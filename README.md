# Felt Embedded Care Android SDK
Felt offers a seamless telehealth experience inside health apps. Our platform handles the full user lifecycle, including onboarding, payments, and doctor visits - with just a few lines of code.

## Getting Started
The SDK is published on [Maven Central](https://central.sonatype.com/artifact/clinic.felt/embeddedcare).

> These docs describe **1.0.0-beta.6**, the current release on Maven Central. Pin an explicit
> version rather than tracking the latest: the next release changes how the SDK is initialized and
> moves several classes into new packages, and this guide will be updated alongside it.

In your build script:
```kotlin
dependencyResolutionManagement {
    repositories {
        // Make sure Maven Central is added as a repository
        mavenCentral()
    }
}

dependencies {
    // Add embedded care as a dependency
    implementation("clinic.felt:embeddedcare:${embeddedCareVersion}")
}
```

At app launch, initialize the SDK with your SDK key using [`EmbeddedCare.shared.initialize`](https://www.javadoc.io/doc/clinic.felt/embeddedcare/latest/clinic/felt/embeddedcare/EmbeddedCare.html#initialize(AppCompatActivity,String)). For example, the implementation could look like:

```diff
+ import clinic.felt.embeddedcare.EmbeddedCare

class MainActivity: AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // other initialization logic

+        val activity = this
+        EmbeddedCare.shared.scope.launch {
+            // Error will be thrown for invalid key.
+            // Ensure that this is only called once per app launch.
+            EmbeddedCare.shared.initialize(activity, sdkKey = /* your Felt SDK Key */)
+        }
    }
}
```

Now you can embed care experiences, below are a few examples. 
See [full documentation](Documentation.md#Component-Experiences) for a complete list of approaches.

```kotlin

// for a modal care experiences
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

* setting account and patient information with [`EmbeddedCare.shared.account`](https://www.javadoc.io/doc/clinic.felt/embeddedcare/latest/clinic/felt/embeddedcare/EmbeddedCare.html#setAccount(Account)) 
* setting component custom data with [`ComponentFragment.create(..., custom = ...)`](https://www.javadoc.io/doc/clinic.felt/embeddedcare/latest/clinic/felt/embeddedcare/ComponentFragment.Companion.html#create(ComponentExperience,Map))

🎉 That's it! By working with Felt, we will populate these views with a customized, white-labeled experience that will delight your users.

## Documentation
You can find further documentation [here](Documentation.md) and api reference documentation [here](https://www.javadoc.io/doc/clinic.felt/embeddedcare/latest/clinic/felt/embeddedcare/package-summary.html).

## Support

### Slack
We should have a Slack Connect setup you should be able to find us individually below or at `#<company name>-felt`

Reach out to sid (at) feltclinic.com or josh (at) feltclinic.com if you are not finding us on Slack.


### Email

* sid (at) feltclinic.com
* daniel (at) feltclinic.com
