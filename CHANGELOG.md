# Changelog

## 1.2.0

Adds public API and changes no existing API. **Check two things before you upgrade**, both under "Behavior changes to review" below: the Sentry entry, because it can stop an app at launch, and the notification permission entry.

```kotlin
implementation("clinic.felt:embeddedcare:1.2.0")
```

### Added

- **`TabBarVisibilityManager.visibility`**, a `StateFlow<Boolean>` for apps built with Views. Collect it to show or hide your own tab bar. It always holds the same value as the Compose `isTabBarVisible` state.
- **`BadgeManager.badgeCount`, `badgeType` and `isBadgeVisible`**, matching the names in the iOS SDK. They return the same values as `count`, `type` and `isVisible`, which remain.
- **Sharing files.** When the embedded care experience shares a file, Android's share sheet opens. Before, the experience reported that the device could not share files.

### Fixed

- **Feature gates after setting the account early.** If your app set `auth.account` while the SDK was still starting, `checkFeatureGate` returned `false` for every gate for the rest of the session, so a Care tab shown behind `FeatureGate.GlobalRollout` never appeared. The account set by your app is now kept.
- **Badges.** Badge counts now appear when badging is enabled for your app.
- **Links to other apps.** `tel:`, `mailto:` and `sms:` links in the experience now open the phone's dialer, email app and messaging app. Before, the experience showed a "Webpage not available" page. A link opens only when the user taps it.
- **Sentry.** The SDK no longer interferes with your app's own Sentry:
  - `EmbeddedCare.initialize` no longer fails in apps that use Sentry 8. The SDK works with Sentry 7.x and 8.x.
  - Before, starting the SDK could shut down Sentry in an app that had already started it.
  - The SDK reports its own errors separately from your app's.
- **FileProvider.** Apps that declare their own `androidx.core.content.FileProvider` no longer fail to build with a manifest merge error. You can remove any manifest workaround you added for it.
- **Rotation and other activity recreation:**
  - Requesting the notification permission no longer crashes the app after the activity that set up the SDK has been destroyed.
  - Permission state keeps refreshing after a rotation.
  - `takeOverBadging` badges the new tab bar after the activity is recreated.
  - The SDK no longer keeps a destroyed activity in memory.
- **Account changes.** When you change `auth.account`, experiences already on screen switch to the new account at once, including on their next page load and while they are in the background.
- **Reminders.** When the experience cancels a reminder that has already been shown, it is also removed from the notification shade, and tapping it no longer opens the experience.
- **Downloaded PDFs** keep the file name the server gives them, and are always saved with a `.pdf` extension.
- **Account status.** Checking account status several times at once no longer hangs.

### Behavior changes to review

- **Sentry: can stop your app at launch.** The SDK now depends on `io.sentry:sentry` instead of `io.sentry:sentry-android`, and no longer changes your app's Sentry settings, matching the iOS SDK. Earlier versions turned off `sentry-android`'s automatic start in every app. **If your app includes `sentry-android`, directly or through another library, and its manifest has no `io.sentry.dsn`, Sentry now tries to start at launch with no DSN and the app crashes.** Add `<meta-data android:name="io.sentry.auto-init" android:value="false" />` inside `<application>`, as Sentry's manual setup guide requires. See [Upgrading from 1.1.2](README.md#upgrading-from-112).
- **Notification permission comes from the activity on screen.** The SDK now asks for the permission from the activity that is currently showing, and only if that activity called `EmbeddedCare.prepareActivity(this)`. Otherwise it logs an error, returns the current permission state, and shows no prompt. Before, it used the activity that first called `prepareActivity`, which crashed the app once that activity was destroyed. Call `prepareActivity` in `onCreate` of every activity that can show the prompt.
- **Overlays open as a sheet.** `presentOverlay` now shows the experience in a card over your screen, which stays visible and dimmed behind it. Before, it opened full screen.
- **Presenting while an experience is open.** A presentation request in a different style (full screen, sheet or transparent) now closes the open experience and opens the new one in the requested style. Before, the new content kept the open experience's style. A request in the same style still navigates within the open experience.
- **SDK experiences are no longer on the fragment back stack.** System back still closes them. If your app counted back stack entries to detect an SDK experience, it will no longer see one.
- **Other non-web links.** Besides `tel:`, `mailto:` and `sms:`, any other non-web link the user taps in the experience (for example `geo:` or an app's own scheme) is handed to the app that handles it. When no app can open a link, a short message appears. Its text comes from these string resources, which you can override to translate: `felt_no_phone_app`, `felt_no_email_app`, `felt_no_messaging_app` and `felt_no_app_for_link`.
- **Badge polling pauses in the background.** While your app is in the background, badge updates stop and `BadgeManager.isPolling` is `false`. They resume when your app returns, so don't restart polling yourself.
- **Badge handlers always register.** `addStateChangeHandler` now always registers your handler. Before, it returned an id without registering anything when badging was off for your app. A handler you add before badging turns on is now called once it does.
- **`kotlinx-coroutines-android`** is now an `api` dependency, so `StateFlow` resolves without your app declaring it.
- **R8.** The SDK ships consumer keep rules that keep the names (not the code) of the SDK's classes and of Capacitor, so the SDK's crash reports stay readable in minified apps. Unused classes are still removed.

## 1.1.2

A drop-in update from `1.1.1`. There are no public API changes.

```kotlin
implementation("clinic.felt:embeddedcare:1.1.2")
```

### Fixed

- **16 KB memory page sizes.** Apps that include the SDK now pass Google Play's 16 KB page size requirement. The SDK's Sentry dependency (`io.sentry:sentry-android`) moves from `7.3.0` to `7.22.6`, whose native libraries are aligned for 16 KB pages. Your app also needs Android Gradle Plugin 8.5.1 or later so that its build packages native libraries at 16 KB alignment.

## 1.1.1

A drop-in update from `1.1.0`. There are no public API changes.

```kotlin
implementation("clinic.felt:embeddedcare:1.1.1")
```

### Added

- **On-device reminders.** The embedded care experience can now schedule and cancel reminders that are delivered as local notifications. Nothing new is needed from your app:
  - Delivery uses no push service and no network call.
  - Reminders require the notification permission, which is the same permission push notifications use.
  - Tapping a reminder opens your app.
  - Pending reminders are not restored after the device restarts.

### Fixed

- **Keyboard.** The keyboard no longer covers text fields in full-screen experiences. This includes experiences hosted with the `FullScreenView` composable. If your app already makes room for the keyboard around the container, the SDK detects it and does not resize the content a second time.
- **System back** on a full-screen experience presented by the SDK now closes the experience in one press:
  - Before, an app using Compose navigation could pop its own screen underneath while the experience stayed visible.
  - It also works after your app returns from the background and after the device rotates.
  - Back first steps through pages within the experience, then closes it.
- **Closing an experience hosted in your own screen** (for example, closing a plan screen shown inside a tab) now returns that screen to its home content instead of leaving it empty.
- **Links that open a new window** no longer replace the experience with a page the user cannot get back from:
  - Links to other sites open in the device's browser.
  - `mailto:` and `tel:` links open the matching app.
- **Tab bar.** Presenting a full-screen experience no longer hides your tab bar. Before, it stayed hidden after the experience closed, and `TabBarVisibilityManager` could not show it again. The presented experience covers the tab bar while it is open.
- **External pages.** The loading indicator now clears for pages outside the embedded care web app, and those pages keep their close button.
- **`presentTransparentFullScreen`** now presents the experience over the current screen, with a dimmed background through which the screen beneath shows wherever the content is transparent. Before, it was delivered to your presentation handler as a `Home` presentation.

### Behavior changes to review

- **Back in experiences inside your own screens.** In an experience hosted in your own screen (`FullScreenFragment` or `FullScreenView`), system back now steps back through the experience's pages while it is on screen. Your app's own back handling runs once there are no pages left.
- **Presentation handler.** `fullScreenExperiencePresentationHandler` now receives `PresentationMethod.TransparentFullScreen` for `presentTransparentFullScreen` calls. Before, it received `PresentationMethod.Home`. If you return `false`, or set no handler, the SDK presents it.
- **Tab bar visibility.** `TabBarVisibilityManager.isTabBarVisible` no longer changes when the SDK presents a full-screen experience. It still follows visibility requests from the embedded care experience and from your own `setTabBarVisibility` calls.

## 1.1.0

First release since `1.0.0-beta.6` (29 Jan 2025), landing roughly eighteen months of work.
**Contains breaking changes** — see [Upgrading from 1.0.0-beta.6](README.md#upgrading-from-100-beta6).

### Breaking

- **Initialization moved to the companion.** `EmbeddedCare.shared` throws until the first
  `initialize` completes, so `EmbeddedCare.shared.initialize(...)` no longer works. Use
  `EmbeddedCare.initialize(activity, sdkKey)`, and call `EmbeddedCare.prepareActivity(activity)`
  from `onCreate` first — it registers the runtime push-permission launcher, which must happen
  before the activity reaches `STARTED`.
- **The account moved behind `auth`** — `EmbeddedCare.shared.auth.account`.
- **Classes moved packages.** `Account`, `Patient` → `clinic.felt.embeddedcare.models`;
  `ComponentFragment` → `clinic.felt.embeddedcare.features.component`;
  `FullScreenFragment` → `clinic.felt.embeddedcare.features.fullscreen`.
- **A second repository is required.** Add JitPack alongside Maven Central; see the
  [README](README.md#getting-started) for the scoped form.

### Added

- **Deep linking** — URL and push-notification handling, queueing for links that arrive before an
  account is set, and `processStoredDeeplinks()` to replay them once your UI is ready. Anonymous
  routes no longer require an account.
- **Tab bar coordination** — embedded content can signal when a host tab bar should be hidden, via
  `tabBarManager.isTabBarVisible`.
- **Automatic tab badging.**
- **Push notifications** — permission manager with an explainer sheet, and device-token
  registration.
- **Events** — `onWebViewDidFinishLoading`, `onFeltSubscriptionCompleted`, `onTermsAcceptance`.
- **Presentation control** — `fullScreenExperiencePresentationHandler`, which can decline
  individual routes so interception is selective. Plus `presentOverlay` and a splash-screen
  experience.
- **Downloads and PDFs** from embedded content, and haptic feedback.
- **`isAccountActive()`**, `getExperiment()`, `Patient.profilePicture`, `Account.sessionToken`,
  SDK version constants, and a `Uri` overload for `handleURL`.
- **Screenshot and screen-recording protection** for sensitive content.
- **Error reporting and structured logging**, with PII obfuscation.

### Fixed

- Faster startup — `EmbeddedCare.shared` is available sooner, as initialization no longer waits on
  remote configuration.
- `ComponentFragment` no longer crashes the host after detach.
- Coroutine cancellation is no longer reported as a network failure.
- Badge polling recovers from failures instead of stopping permanently.
- FCM device tokens are now accepted.
- Configuration state is safe to read across threads.
- RTL locale support in embedded layouts.
- Safe-area insets account for the host navigation bar; inline components no longer receive one.
- Web context is injected at document start and refreshed when the account changes.
- Network retries apply to page loads only, not subresources.

## 1.0.0-beta.6

29 Jan 2025.
