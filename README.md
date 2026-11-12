```
    ███████╗███████╗██╗  ████████╗
    ██╔════╝██╔════╝██║  ╚══██╔══╝
    █████╗  █████╗  ██║     ██║
    ██╔══╝  ██╔══╝  ██║     ██║
    ██║     ███████╗███████╗██║
    ╚═╝     ╚══════╝╚══════╝╚═╝

    E M B E D D E D   C A R E   ·   A N D R O I D
```

# Felt Embedded Care — Android SDK

Felt puts a complete telehealth experience inside your health app: onboarding, payments and doctor
visits, with a few lines of code.

**Current version: `1.2.0`** · [Maven Central](https://central.sonatype.com/artifact/clinic.felt/embeddedcare) · [Changelog](CHANGELOG.md)

## Documentation

The docs live in **two places**:

| Where | What's there | Read it when |
|---|---|---|
| 📘 **Here, on GitHub:** [Documentation.md](Documentation.md) | The full integration guide: install, initialize, sign the user in, show care screens, deep links and push, lifecycle, and how Android differs from iOS | You are adding the SDK to your app. Start here |
| 📚 **The docs site:** [feltclinic.github.io/felt-embedded-care-android](https://feltclinic.github.io/felt-embedded-care-android/) | A short setup guide, then the API reference: every public class, function and property, with its documentation | You need the details of one specific API |

## Getting started

### 1. Add the dependency

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven("https://jitpack.io") // one of the SDK's dependencies is published only on JitPack
    }
}

// app/build.gradle.kts
dependencies {
    implementation("clinic.felt:embeddedcare:1.2.0")
}
```

Pin an exact version rather than tracking the latest.

### 2. Set up the SDK

1. In your activity's `onCreate`, **every time** it is created, call `EmbeddedCare.prepareActivity(this)`.
2. **Once per process**, call `EmbeddedCare.initialize(activity, sdkKey)` from a scope that outlives the
   activity. Use the companion (`EmbeddedCare.initialize`), not `EmbeddedCare.shared`, which throws until
   `initialize` has finished. → [Initialize](Documentation.md#3-initialize)
3. Set the signed-in user with `EmbeddedCare.shared.auth.account = Account(...)`.
   → [Sign the user in](Documentation.md#4-sign-the-user-in)
4. Hand deep links and notification taps to the SDK from `onResume`.
   → [Deep links and push notifications](Documentation.md#6-deep-links-and-push-notifications)

### 3. Show care

```kotlin
// a modal care experience
EmbeddedCare.shared.presentExperience(FullScreenExperience.ValuePropositionScreen)

// a care tab, in Compose
FullScreenView(experience = FullScreenExperience.HomeScreen, modifier = Modifier.fillMaxSize())

// a small component alongside your own content
Component(experience = ComponentExperience.DismissableCallToAction)
```

🎉 That's it. Felt fills these views with a customized, white-labeled experience for your users.

## Requirements

| | |
|---|---|
| `minSdk` | 24 (Android 7.0) |
| `compileSdk` | 34 or higher |
| Host activity | An `AppCompatActivity` (`initialize` and `prepareActivity` require one) |
| Android Gradle Plugin | 8.5.1 or newer |
| Kotlin | 2.0 or newer |
| Repositories | Maven Central **and** JitPack |

The guide's [Requirements](Documentation.md#requirements) section explains why, and lists permissions.

## Upgrading

### Upgrading from 1.1.2

`1.2.0` changes no existing API. Check the two cases below. If neither applies to your app, there is
nothing to change. The [changelog](CHANGELOG.md) lists everything else.

**1. Your app includes `sentry-android` and its manifest has no `io.sentry.dsn`.** This includes
`sentry-android` pulled in by another library. Add this inside `<application>`, as
[Sentry's manual setup guide](https://docs.sentry.io/platforms/android/manual-setup/) requires:

```xml
<meta-data android:name="io.sentry.auto-init" android:value="false" />
```

Without it, your app crashes on every launch. `1.1.2` turned off Sentry's automatic start for every app,
which hid the missing line. `1.2.0` no longer changes your Sentry settings, which matches the iOS SDK.
To check whether your app includes `sentry-android`, run `./gradlew :app:dependencies` and search the
output.

**2. Your app shows the notification permission prompt from more than one activity.** Call
`EmbeddedCare.prepareActivity(this)` in `onCreate` of each of them. The SDK now asks from the activity on
screen, and shows no prompt if that activity didn't call it.

*Optional:* remove any manifest workaround you added for the SDK's `FileProvider`. The SDK now declares
its own provider class, so it no longer conflicts with yours.

### Upgrading from 1.0.0-beta.6

`1.0.0-beta.6` was published in January 2025. Three changes need edits to an existing integration.

**1. Initialization moved to the companion.**

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

**2. The account moved behind `auth`.**

```diff
- EmbeddedCare.shared.account = Account(id = user.id, email = user.email)
+ EmbeddedCare.shared.auth.account = Account(id = user.id, email = user.email)
```

**3. Several classes moved packages.**

| Was | Now |
|---|---|
| `clinic.felt.embeddedcare.Account` | `clinic.felt.embeddedcare.models.Account` |
| `clinic.felt.embeddedcare.Patient` | `clinic.felt.embeddedcare.models.Patient` |
| `clinic.felt.embeddedcare.ComponentFragment` | `clinic.felt.embeddedcare.features.component.ComponentFragment` |
| `clinic.felt.embeddedcare.FullScreenFragment` | `clinic.felt.embeddedcare.features.fullscreen.FullScreenFragment` |

`ComponentExperience`, `FullScreenExperience`, `FeatureGate` and `EmbeddedCare` itself are unchanged.

## Support

- **Slack:** we share a Slack Connect channel with you, usually `#<company name>-felt`.
- **Email:** sid (at) feltclinic.com, josh (at) feltclinic.com or daniel (at) feltclinic.com.
