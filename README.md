# Consumer Clarity Research Notes

Technical record for the Consumer Clarity Talabat patch project. This repository preserves analysis and implementation knowledge independently of the patch source. It contains no APKs, credentials, private keys, device identifiers, or local workstation paths.

## Current deliverables

- Patch source: https://github.com/AltF4G0x/consumer-clarity
- Current release: https://github.com/AltF4G0x/consumer-clarity/releases/tag/v1.1.0
- Patch bundle: `patches-1.1.0.mpp`
- Target application: talabat (`com.talabat`) 14.4.0, version code 2368
- Supported architecture: ARM64 only for native sorting
- Release MPP SHA-256: `147c669744b7fb40af33c90ec6f542e6d7a5c8c4e9f6d372aa4e1082f0d8dad4`

The source publishes only Morphe patch bundles and metadata. No pre-patched APK is distributed.

## Analysis toolchain

REA 6.5.0 runs in the existing Docker environment and was configured as a project-local MCP server. Its Android package, class/method, resource, ADB, and native-analysis tools were used throughout. Jadart 1.3.0 supplemented REA for Dart AOT object recovery and pseudo-Dart lifting.

The application is Flutter/Dart rather than a conventional Kotlin UI. Restaurant and review logic therefore lives mainly in `libapp.so`, not in DEX methods that can be modified with ordinary Morphe bytecode fingerprints alone.

REA identified the base package and splits:

- Base APK SHA-256: `91161aeaf9eaaa4e4de72a691ef426b461d5fddf1da65944bf7c72b7237944f0`
- ARM64 split SHA-256: `f665df12f054bea9fde3f9fe8124f2774f137f3433a6eaceb112cbc6c7332b7d`
- Density split SHA-256: `55ff921485671500971b2a4097a8934d9f72cf4896c8cac409e16931e85211a3`
- `libapp.so` SHA-256: `ad40de2c7dbe56dd00a0cb0ef9a1c5fd7f831b5a9d1a39ac60eddda7742b12f6`
- Dart VM snapshot hash: `78da37fed6bf1489361a312568249f3f`
- Dart version: 3.11.5, compressed pointers, dwarf stack traces mode

Jadart passed all 11 Tier A and 3 Tier B acceptance gates, covering 685,535 byte-level checks. This established that subsequent AOT findings used a validated snapshot grammar.

## Existing patches

### Consistent restaurant and review sorting

A small ARM64 native library installs two instruction-guarded hooks. It checks the exact original instruction bytes before activation and refuses to patch a mismatched binary.

Review behavior:

- Hook VA: `0x00ffced0`
- `_kDartIsolateSnapshotInstructions` VA: `0x00be6a40`
- Reviews are sorted newest-first using valid ISO date strings.
- Review ID is the stable ascending tie-breaker.
- Review text and star values are not changed.
- Each response is processed as received, including later pages.

Review model fields confirmed in the AOT:

- ID: `0x08`
- Author: `0x0c`
- Text: `0x10`
- Rating: `0x14`
- Date: `0x18`

Restaurant behavior:

- Hook VA: `0x017ae4e4`
- Entries with a positive `sponsored_rank` are placed after organic entries.
- Remaining entries sort by descending rating and ascending ID.
- Organic CPC attribution metadata is not treated as proof of sponsorship.

Restaurant model fields confirmed in the AOT:

- ID: `0x08`
- Rating (`rat`): `0x34`
- `sponsored_rank`: `0x197`

The native code reorders compressed list-element pointers in place and performs no allocation or network activity. The trampolines preserve general and SIMD registers and replay the overwritten Dart prologue before returning.

### Reduce analytics tracking

Confirmed void or result-returning SDK boundaries are patched independently of restaurant, checkout, login, and order-tracking traffic:

- Firebase `AnalyticsConnectorImpl.logEvent`
- Adjust `trackEvent`, `trackAdRevenue`, `trackPlayStoreSubscription`
- Perseus `PerseusHitsRepositoryImpl.insertEvent`
- Braze `logCustomEvent`, `logPurchase`, `logBannerClick`, `logBannerImpression`, `logFeatureFlagImpression`, `setGoogleAdvertisingId`

A Dart routine initially considered as a central telemetry dispatcher was rejected after decompilation showed it only constructed event objects. It was not patched.

### Hide Braze in-app popups

`BrazeInAppMessageManager.registerInAppMessageManager` and `addInAppMessage` return early, preventing Braze promotional in-app messages from being registered or queued. Functional dialogs and push notifications are unchanged.

### Disable Play license gate

`com.pairip.licensecheck.LicenseClient.checkLicense` and `scheduleRepeatedLicenseCheck` return early. This suppresses the pairip startup check and its scheduler without globally disabling Google APIs.

The unmodified app displayed the same licensing activity in the emulator, proving that the gate was not introduced by these patches.

## Native packaging

The first test APK failed because Morphe compressed the added `.so` while Talabat declared `extractNativeLibs=false`. The sorting patch now uses a full resource patch to set `android:extractNativeLibs=true`, making Manager-produced APKs installable without manual archive edits.

The native loader correctly executes `System.loadLibrary("consumerclarity")` after the Dart library is loaded. Both hooks logged successful activation (`reviews=1 restaurants=1`) with no crash report.

## Repository and release format

The public patch repository uses semantic-release and the standard Morphe template conventions:

- `patches-bundle.json` resolves the latest stable MPP and metadata.
- `patches-list.json` describes all five patches and exact compatibility.
- MPP manifest includes name, description, version, source, author, license, and `Patcher-Version: 2.1.0`.
- GitHub Releases contain only MPP files, never APKs.

Morphe Desktop independently resolved the repository and downloaded all 712,858 bytes of the v1.0.0 release MPP. It then loaded every patch successfully. This verified the remote-source path independently of device-specific source caching. For v1.1.0, the public release asset and `patches-bundle.json` were downloaded successfully, and the release asset's SHA-256 exactly matches the locally built bundle.

## Build environment

The source build uses:

- Morphe patches Gradle plugin 2.0.0
- Morphe patcher 2.1.0
- JDK 17 for patches
- Java 21 for Morphe Desktop
- Android NDK 29.0.14206865

Official Morphe patcher and plugin artifacts can be published to Maven Local when GitHub Packages authentication is unavailable. Local release-signing is optional; it does not affect Morphe's own signing of patch sources or output APKs.

## Expanded feature set

A fifth patch now routes `openUrlInApp` to Android's default browser while preserving the original path when its setting is disabled. The URL-launcher analysis also confirmed that `launchUrl` already used an external browser, so only the in-app path was changed.

A native Consumer Clarity settings activity is registered through Talabat's manifest and linked from the About page. It provides:

- Sponsored modes: demote, hide, or original order.
- Review modes: newest, oldest, highest rated, or original order.
- Runtime toggles for analytics reduction, Braze popups, the Play license gate, and external links.
- Links to the original Facebook and FAQ pages, patch source, and this research record.
- Developer information explaining that no separate Talabat developer menu was found and that server experiments are not safe universal switches.

The activity uses Android shared preferences. Native sorting reads the same preference file directly, while Java/SDK patches check it before returning early. Disabling a setting therefore restores the original code path rather than just changing a label.

The extension manifest alone did not register the activity in Morphe's output, so the resource patch now declares it explicitly. The final manifest and About JSON were inspected to confirm the deep link and label are present.

Video autoplay was deliberately not changed: the available `VideoPlayerInstanceApi.play` path cannot distinguish autoplay from manual playback, and suppressing it would break user-initiated playback.

## Verification status and limitations

Confirmed:

- All five patches compile and apply with Morphe validation.
- Native sorting hooks activate in an ARM64 emulator session.
- The release MPP and both metadata files download correctly through Morphe's own resolver.
- Repository files contain no workstation usernames, host paths, private keys, or test credentials.

Not fully verified:

- Complete restaurant/review UI flows after all patches, because the emulator went offline before those checks.
- Server-provided aggregate rating truth or stability. A client patch cannot make changing server values independently truthful.
- Google-independent behavior of Maps, login, push, payments, or other Google-dependent features.
- Display-ad inventory and sponsored-deals popups outside Braze are not claimed as blocked.
- Later-page global ordering is not claimed; each response is sorted as it arrives.

## Development history and privacy

The initial private repository was superseded by a fresh public repository with a clean initial commit, ensuring old workstation paths could not survive in public history. The older private repository was left untouched rather than deleted without approval.

GitHub credentials are stored outside all repositories with owner-only permissions. No credential is included in source, release assets, or this research record.
