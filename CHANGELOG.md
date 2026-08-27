# Changelog

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
