# Preventing Remote Desktop Fraud on Mobile

<!-- AUTHOR joshis_tweets 2024-01-22T00:00:00Z -->
<!-- SIDEBAR _auto -->
<!-- TEMPLATE tutorial -->
<!-- COVER_IMAGE cover.webp -->

Remote desktop apps are useful tools when one needs to provide convenient customer support. However, they are often misused by fraudsters to steal money from banking accounts. In this tutorial, we want to provide the problem overview and help you design and implement appropriate remedies in your mobile apps, such as detecting screen-sharing apps on Android or blocking screen-sharing on iOS.

## Understanding the Fraud Scenario

Protection from remote desktop fraud requires first understanding the attack mechanics and then designing active mitigation measures to countermeasure the techniques used by the fraudsters.

In remote desktop fraud, the fraudster convinces the customer to install some of the remote desktop apps that are available on Google Play or App Stores, and convinces the user to share the screen, or even grants active access to the device to the fraudster. In principle, any remote desktop app can be used to carry out the attack, for example, AnyDesk, TeamViewer, Vysor, or similar apps.

The fraudster's argument for the installation can vary from "We will help you invest your money in crypto" to "We found a security incident on your account and want to help you resolve it." Fraudsters can be quite innovative when crafting their scripts. Very often, the user is also manipulated into installing additional addon applications (such as an official AnyDesk Plugin AD1 on Android) that enable full remote control of the device. During the process, the user is instructed to ignore any warnings the mobile OS or AnyDesk app itself is showing as the installation progresses.

After the customer installs the remote desktop app, the fraudster initializes the remote session, and the fraud progresses with several possible follow-up techniques to cause financial damage:

- Intercepting credentials and SMS OTP codes from displayed system notifications
- Recording and replaying the PIN code of the mobile banking
- Intercepting sensitive data, such as activation or recovery codes
- Intercepting other personal data from apps on the device to conduct identity theft

## Recommended Mitigation Techniques

To effectively mitigate the impact of remote desktop on your application, we recommend two main categories of improvements in your app:

- Improve Your Authentication
- Fortify Your Apps with In-App Protection

### Improve Your Authentication

As a general defense mechanism against remote desktop takeovers, you should ensure your authentication is not affected by remote desktop apps and that the credentials cannot be stolen via remote desktop.

#### Decommission SMS OTP or Code-Based Authentication

If you still provide a 2FA login via SMS OTP or OTP generated in the mobile app, it is time to change your authentication.

Both of these codes can be, in principle, captured by remote desktop apps. With SMS OTP, the situation is the worst as the victim does not even have to use your mobile banking app and can still be vulnerable to remote desktop fraud. As SMS codes are delivered to the system SMS app and there is no component present on the user's device that is under your control, you would learn about the impacted user once the fraud happens - in other words, too late.

You can learn more about replacing SMS OTP or app-based OTP codes with more advanced mobile-first authentication in our dedicated tutorial.

#### Consider Server-Side Facial Biometrics as Authentication Step-Up

To minimize damage in many fraudulent scenarios, including remote desktop fraud, we recommend designing techniques for authentication step up. One suitable technique can be server-side facial biometrics. Unlike Face ID, it captures the face of the person in front of the computer and verifies it against the biometric template on the server side. You can apply the authentication step up to server-side biometry in several situations, for example:

- For any payment approval, in case the user logs in via PIN code (to enforce independent factors for login and payment approvals in the same session when the PIN code could have been compromised during login).
- For a high-value transaction, or for every ~10th transaction (to prevent the fraud being carried out via several smaller payments).
- For a transaction to any new account number.

Of course, you can also deploy a more complex decision-making based on the responses from your fraud detection system.

### Fortify Your Apps with In-App Protection

#### Detect and React on Screen Sharing

On Android, apps like AnyDesk (with the additional plugin) allow fraudsters not only to record whatever happens on the mobile screen, but also send active gestures back to the app. As a result, the attacker can easily record the PIN code or password of the user and replay the PIN code back to the mobile app. As the PIN code is typically the default mechanism for authentication and payment approval, the attacker now has full control over your customer's account. While iOS only allows reading the screen content, the PIN code leakage is still highly problematic.

Note that the attackers could also record the PIN code for the device screen lock, which could be the same as the PIN code for your banking app - the users are not good at choosing different PIN codes for multiple apps. As a result, the fraudster who controls your customer's device via remote desktop can cause damage even if your customer does not enter your mobile banking to enter the PIN code. Also, some especially sinister fraudsters are toying with the users by closing the biometric dialog, forcing them to enter the PIN code instead of using biometric authentication.

Fraudsters also do not necessarily have to focus on the PIN code. Instead, they may look for various recovery information, such as recovery codes or device transfer codes displayed in the app or stored in various locations on the device.

Once you detect that screen sharing takes place when your customers use your app, we recommend hiding the screen content and preventing remote interactions. You can use the following feature of our in-app protection to make this change easily:

<!-- begin tabs -->
<!-- tab Kotlin -->
```kotlin
val raspConfig = RaspConfig.Builder()
    .screenSharing(DetectionConfig.Notify)
    .build()


val raspObserver = object: RaspObserver {
    override fun onScreenSharingDetected(screenSharingDetected: Boolean) {
        // handle screen sharing detection, hide the screen content and disable content interaction
    }
}

raspManager.registerRaspObserver(raspObserver)
```
<!-- tab Swift -->
```swift
let raspConfig = AppProtectionRaspConfig(
    screenCapture: .hide(), // will hide the app contents when the screen is captured (for example shared via airplay),
)

/// ...

// Set the delegate to the existing `AppProtectionService` instance to obtain RASP callbacks
appProtection.rasp.addDelegate(self)

/// ...

func screenCapturedChanged(isCaptured: Bool) {
    // react to screen capturing by blocking the content interaction
}
```
<!-- end -->

#### Block Untrusted Screen Readers

On the Android platform, some types of remote access tools may not use the traditional screen sharing but rather the accessibility services (aka "screen readers"). Accessibility services were originally designed to provide aid to visually or otherwise impaired users but became often misused for other use cases, such as scripted interaction with the user interface. For financial applications, only a list of allowed and legitimate accessibility apps should be able to interact with the app content this way.

<!-- begin box info -->
Unfortunately, the Android platform does not allow selectively blocking individual screen readers. This is because of how accessibility features work. In essence, your app emits accessibility events to the Android OS, which emits the events to all screen readers. To prevent abuse of the accessibility features, you need to stop emitting accessibility events from your app once you detect a possibly problematic screen reader, making your app invisible to all screen readers in the system.
<!-- end -->

You can use the following feature in our in-app protection on Android:

<!-- begin tabs -->
<!-- tab Kotlin -->
```kotlin
val allowList = listOf(
        // specify an allowed app by its package name and signature hash (recommended approach)
        RaspConfig.ApkAllowlistItem("com.google.android.marvin.talkback", "9b424c2d27ad51a42a337e0bb6991c76eca44461"),
        // specify an allowed app only by its package name (less secure)
        RaspConfig.ApkAllowlistItem("com.samsung.accessibility")
    )

val raspConfig = RaspConfig.Builder()
    .screenReader(
        ScreenReaderBlockConfig.Builder()
          .action(BlockConfig.Block)
          .allowedScreenReaders(allowList)
          .build()
    )
    .build()
```
<!-- end -->

This code will block screen readers that are not on the provided list of allowed apps. We provide a default recommended value of the list as `ScreenReaderBlockConfig.DEFAULT_ALLOWED_SCREEN_READERS`.

#### Block Screenshots

We found several clever Android applications (i.e., Vysor) that creatively use the ADB bridge to perform remote screencast using the same technique that would conventionally be used to make screenshots. Therefore, we recommend disabling the ability to make screenshots on Android. While this may cause minor complications when supporting users, the security aspect can not be ignored, and your users can find a workaround to their support claims (taking a photo with the second device or visiting a branch).

You can easily block screenshots in your Android app with our in-app protection SDK:

<!-- begin tabs -->
<!-- tab Kotlin -->
```kotlin
val raspConfig = RaspConfig.Builder()
    .screenshot(BlockConfig.Block)
    .build()
```
<!-- end -->

On iOS, the issue is less severe. However, an iOS device connected to a debugger or a trusted computer can leak screenshots via a technique called "trustjacking". Unfortunately, on iOS, it is not possible to block screenshot capture across the app effectively and generically. However, you can use our in-app protection SDK to learn that the screenshot took place, and if this happens too frequently in a short period of time, you can terminate the app.

<!-- begin tabs -->
<!-- tab Swift -->
```swift
// Set the delegate to the existing `AppProtectionService` instance to obtain RASP callbacks
appProtection.rasp.addDelegate(self)

/// ...

func userScreenshotDetected() {
    // react to user screenshot
}
```
<!-- end -->

To hide sensitive content in specific views, you can also use our class extension to hide content from selected `UIView` instances:

<!-- begin tabs -->
<!-- tab Swift -->
```swift
view.preventScreenshots()
```
<!-- end -->

<!-- begin box warning -->
Note: Due to techniques we use to prevent screenshots on specific views, the correct layout of your app can be impacted in case the feature is used carelessly. Cautiously select the right view to apply the protection to, ideally, a selected fixed view in your screen layout.
<!-- end -->

#### Detect Remote Desktop Apps Presence

Finally, a way to prevent damage caused by remote desktop apps is to directly detect such app presence and notify the user, explaining the possible risks. To detect screen reader apps from your application, you can use the following functionality of our app protection SDK:

<!-- begin tabs -->
<!-- tab Kotlin -->
```kotlin
val raspConfig = RaspConfig.Builder()
    .appPresence(
        AppPresenceDetectionConfig.Builder()
            .action(DetectionConfig.Notify)
            .remoteDesktopApps(AppPresenceDetectionConfig.DEFAULT_REMOTE_DESKTOP_APPS)
            .build()
    )
    .build()

val appPresenceDetection = raspManager.getAppPresenceDetection()

val raspObserver = object: RaspObserver {
    override fun onAppPresenceChanged(appPresenceDetection: AppPresenceDetection) {
        // handle app presence detection
    }
    // handle detection of other RASP features
}

raspManager.registerRaspObserver(raspObserver)
```
<!-- tab Swift -->
```swift
// Prepare the RASP feature configuration
let raspConfig = AppProtectionRaspConfig(
    appPresence: .notify([
        .KnownApps.anyDesk, 
        .KnownApps.teamViewer,
        .KnownApps.logMeIn,
        .KnownApps.msRemoteDekstop,
        .KnownApps.jumpDesktop,
        .KnownApps.parallelsAccess,
        .KnownApps.chromeRemoteDesktop,
    ])
)

let detectedApps = appProtection.rasp.installedApps

// Set the delegate to the existing `AppProtectionService` instance to obtain RASP callbacks
appProtection.rasp.addDelegate(self)

/// ...
func installedAppsChanged(installedApps: [DetectableApp]) {
}
```
<!-- end -->

#### Detect Other Signs of Remote Manipulation

As indicated in some of the items above, detecting a device which is remotely controlled could be tricky. And it can get even trickier. For example, what if the user has a Bluetooth keyboard connected to a smartphone, and that keyboard is hacked?

As a result, we recommend considering additional, less definitive signs of possible remote control, such as:

- Too many screenshots are taken in a short time period - you can use our SDK method for it
- Biometric prompt suddenly deactivated, even multiple times - you can measure it in your authentication flow
- The active call is in progress - you can use our SDK method for it

Of course, the list of techniques could grow over time, and the selection is always about balancing security with user experience.

#### Ensure Protection Measures Cannot Be Deactivated

Finally, we also greatly recommend fortifying the mobile apps' security by performing app hardening steps and obfuscation. This ensures impeding comprehension and complicates highly sophisticated and targeted attacks, which could result in deactivating any specific security measures implemented in the mobile app.

## Resources

- [Tutorial: Mobile-First Authentication in Banking (SCA)](https://developers.wultra.com/tutorials/posts/Mobile-First-Authentication/)
- [Mobile In-App Protection for iOS and Android](https://developers.wultra.com/tutorials/posts/In-App-Protection/)
- [In-App Protection for Android Apps](https://developers.wultra.com/components/malwarelytics-android/develop/documentation/)
- [In-App Protection for iOS Apps](https://developers.wultra.com/components/malwarelytics-apple/develop/documentation/)
- [In-App Protection for React Native Apps](https://developers.wultra.com/components/react-native-malwarelytics/1.0.x/documentation/)
- [In-App Protection for Cordova Apps](https://developers.wultra.com/components/malwarelytics-cordova-plugin/develop/documentation/)
