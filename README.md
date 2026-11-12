# Felt Embedded Care Android SDK
Felt offers a seamless telehealth experience inside health apps. Our platform handles the full user lifecycle, including onboarding, payments, and doctor visits - with just a few lines of code.

## Getting Started
The SDK is published on [Maven Central](https://central.sonatype.com/artifact/clinic.felt/embeddedcare).

> These docs describe **1.2.0**. Upgrading from `1.1.2`? See [Upgrading from 1.1.2](#upgrading-from-112).
> If you are upgrading from `1.0.0-beta.6`, initialization has
> changed and several classes have moved packages: see [Upgrading from 1.0.0-beta.6](#upgrading-from-100-beta6).
> Pin an explicit version rather than tracking the latest.

**Start with the [integration guide](Documentation.md).** It covers install, initialization, signing
the user in (including passing your own auth tokens), showing care screens, deep links and push,
lifecycle, and how Android differs from iOS.

In your build script:
```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        // One of the SDK's dependencies publishes to JitPack rather than Maven Central.
        maven("https://jitpack.io")
    }
}

// app/build.gradle.kts
dependencies {
    implementation("clinic.felt:embeddedcare:1.2.0")
}
```

Then, in outline:

1. In your activity's `onCreate`, every time it is created, call `EmbeddedCare.prepareActivity(this)`.
2. Call `EmbeddedCare.initialize(activity, sdkKey)` **once per process**, from a scope that outlives
   the activity. Use the companion, never `EmbeddedCare.shared`, which throws until `initialize` has
   finished. See [Initialize](Documentation.md#3-initialize) for a pattern that survives rotation.
3. Set the signed-in user with `EmbeddedCare.shared.auth.account = Account(...)`.
   See [Sign the user in](Documentation.md#4-sign-the-user-in).
4. Show care experiences, once the SDK has started:

```kotlin
// a modal care experience
EmbeddedCare.shared.presentExperience(FullScreenExperience.ValuePropositionScreen)

// a care tab, in Compose
FullScreenView(experience = FullScreenExperience.HomeScreen, modifier = Modifier.fillMaxSize())

// a small component alongside your own content
Component(experience = ComponentExperience.DismissableCallToAction)
```

5. Hand deep links and notification taps to the SDK from `onResume`.
   See [Deep links and push notifications](Documentation.md#6-deep-links-and-push-notifications).

🎉 By working with Felt, we will populate these views with a customized, white-labeled experience
that will delight your users.

## Upgrading from 1.1.2

`1.2.0` changes no existing API. Check these two cases; if neither applies to your app, there is
nothing to change. [CHANGELOG.md](CHANGELOG.md) lists everything else.

**If your app includes `sentry-android`** (directly, or through another library) **and your manifest
has no `io.sentry.dsn`**, add this inside `<application>`, as
[Sentry's manual setup guide](https://docs.sentry.io/platforms/android/manual-setup/) requires:

```xml
<meta-data android:name="io.sentry.auto-init" android:value="false" />
```

`1.1.2` turned off `sentry-android`'s automatic start in every app, so an app could work without this
line. `1.2.0` no longer changes your Sentry settings, which matches the iOS SDK. Without the line,
Sentry tries to start at launch with no DSN and the app crashes on every launch. To check whether
your app includes `sentry-android`, run `./gradlew :app:dependencies` and search for it.

**If your app shows the notification permission prompt from more than one activity**, call
`EmbeddedCare.prepareActivity(this)` in `onCreate` of each of them. The SDK now asks from the activity
currently on screen, and shows no prompt if that activity didn't call it.

Optionally, remove any manifest workaround you added for the SDK's `FileProvider`. The SDK now
declares its own provider class, so it no longer conflicts with yours.

## Upgrading from 1.0.0-beta.6

`1.0.0-beta.6` was published in January 2025 and this release lands everything since. Three changes
require edits to an existing integration.

**Initialization moved to the companion.**

```diff
- EmbeddedCare.shared.scope.launch {
-     EmbeddedCare.shared.initialize(activity, sdkKey)
- }
+ EmbeddedCare.prepareActivity(activity)   // in onCreate, every time
+ appScope.launch {                          // once per process
+     EmbeddedCare.initialize(activity, sdkKey)
+ }
```

[Initialize](Documentation.md#3-initialize) shows the full pattern.

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
| `compileSdk` | 34 or higher |
| Host activity | must be an `AppCompatActivity`; `initialize` and `prepareActivity` require one |
| Android Gradle Plugin | 8.5.1 or newer |
| Kotlin | 2.0 or newer |
| Repositories | Maven Central **and** JitPack |

See [Requirements](Documentation.md#requirements) for the reasons and for permissions.

## Documentation
You can find the integration guide [here](Documentation.md) and the API reference [here](https://feltclinic.github.io/felt-embedded-care-android/).

## Support

### Slack
We should have a Slack Connect setup you should be able to find us individually below or at `#<company name>-felt`

Reach out to sid (at) feltclinic.com or josh (at) feltclinic.com if you are not finding us on Slack.


### Email

* sid (at) feltclinic.com
* daniel (at) feltclinic.com
