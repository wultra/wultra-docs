# In-App Protection&#58; Implementing on iOS

<!-- AUTHOR joshis_tweets 2023-12-29T00:00:00Z -->
<!-- SIDEBAR _auto -->
<!-- TEMPLATE tutorial -->

This tutorial summarizes how to integrate Wultra's in-app protection into your iOS app.

<!-- begin box info -->
In-app protection for Apple supports iOS 12+ and requires Xcode 14.3+.
<!-- end -->

<!-- begin box warning -->
**Personalized Configuration Required.**<br/>
In order to use in-app protection for Apple, you need a custom configuration and access credentials. Contact our sales representatives or technical consultant in order to obtain the required prerequisites.
<!-- end -->

## Get the In-App Protection SDK

You can obtain the in-app protection SDK for Apple from our Artifactory.

You need to create a credentials file to access our private repository (needed for both SPM and Cocoapods integration). Create an `.netrc` file in your home directory `(~`) with the credentials to our Artifactory.

```
machine wultra.jfrog.io
      login [name@yourcompany.com]
      password [password]
```

Then, you can add our SPM repository to your Xcode project:

```
https://github.com/wultra/malwarelytics-apple-release
```

## Implement Configuration in Your App

The main class for in-app protection SDK for Apple is `AppProtectionService`. You are responsible for configuring the instance as soon as possible after the application launch. You should also implement the `AppProtectionRaspDelegate` to handle RASP callbacks. The easiest way to initialize the in-app protection SDK in your `AppDelegate`.

The following `AppSecurity` Swift class is a sample wrapper implementation over the `AppProtectionService`.

```swift
import Foundation
import AppProtection

class AppSecurity: AppProtectionRaspDelegate {
    
    private let appProtection: AppProtectionService

    /// Creates AppSecurity instance
    init() {
    
        // Prepare the RASP feature configuration
        let raspConfig = AppProtectionRaspConfig(
            jailbreak: .exit("https://myurl.com/jalibreak-explained"),
            debugger: .block,
            reverseEngineeringTools: .notify,
            httpProxy: .notify,
            repackage:.exit([AppProtectionTrustedCert(withBase64EncodedString: "BASE_64_ENCODED_CERT")!], "https://myurl.com/repackage-explained"),
            screenCapture: .hide(),
            vpnDetection: .notify,
            callDetection: .notify,
            appPresence: .notify([.KnownApps.anyDesk])
        )
        
        // Prepare a configuration for service
        let config = AppProtectionConfig(
            raspConfig: raspConfig
        )

        // Create the Service
        self.appProtection = AppProtectionService(config: config)

        // Register self as delegate
        appProtection.rasp.addDelegate(self)
    }
    
    deinit {
        // When our AppSecurity class is being "destroyed", we want
        // to stop all features of the AppProtectionService.
        self.appProtection.release()
    }

    // MARK: - AppProtectionRaspDelegate

    func debuggerDetected() {
        // react to debugger
    }

    func jailbreakDetected() {
        // react to jailbreak
    }

    func repackageDetected() {
        // react to repackage
    }

    func httpProxyEnabled() {
        // react to http proxy enabled
    }

    func userScreenshotDetected() {
        // react to user screenshot
    }

    func reverseEngineeringToolsDetected() {
        // react to reverse engineering tools
    }

    func systemPasscodeConfigurationChanged(enabled: Bool) {
        // react to system passcode change
    }

    func systemBiometryConfigurationChanged(enabled: Bool) {
        // react to biometry configuration changed
    }
    
    func screenCapturedChanged(isCaptured: Bool) {
        // react to screen capturing (casting to different device)
    }
    
    func vpnChanged(active: Bool) {
		// react to VPN state changes
	}
	
	func onCallChanged(isOnCall: Bool) {
	   // on call status has changed
	}
	
	func installedAppsChanged(installedApps: [DetectableApp]) {
		// installed apps list has changed
	}
}
```

### Configuring Repackaging Detection

Repackaging detection is a security feature that detects if the application was modified and resigned with a different signing certificate.

To properly configure the repackage detection, you need to get the Base64 encoded string of your signing certificate:

- Open the "Keychain Access" application.
- Find a certificate that will be used to sign your application, for example, “Apple Development: Jan Tester (c)”.
- Right-click on the item and click “Export…”.
- Export the certificate in the .cer format.
- Open up the terminal and cd into the folder with your exported certificate.
- Encode the certificate in Base64 with `cat your_exported.cer | base64`.
- Copy the output of the command and use it as a parameter for the repackage detection configuration.

```swift
// Prepare the RASP feature configuration
let raspConfig = AppProtectionRaspConfig(
    // ...
    repackage:.exit([AppProtectionTrustedCert(withBase64EncodedString: "BASE_64_ENCODED_CERT")!], "https://myurl.com/repackage-explained")
    // ...
)
```

### Obtaining the Detection Results

You can observe for the detection results by registering the `AppProtectionRaspDelegate` class, as illustrated in the example above.

Besides responding to callbacks in the delegate, you can also access any RASP detections just-in-time, like so:

```swift
// root detection
let isJailbroken = appProtection.rasp.isJailbroken

// debugger
let isDebuggerConnected = appProtection.rasp.isDebuggerConnected

// repackaging
let isRepackaged = appProtection.rasp.isRepackaged

// screen sharing
let isScreenCaptured = appProtection.rasp.isScreenCaptured

// system passcode
let isSystemPasscodeEnabled = appProtection.rasp.isSystemPasscodeEnabled

// system biometry
let isSystemBiometryEnabled = appProtection.rasp.isSystemBiometryEnabled

// simulator build
let isEmulator = appProtection.rasp.isEmulator

// reverse engineering
let isReverseEngineeringToolsPresent = appProtection.rasp.isReverseEngineeringToolsPresent

// http proxy present
let isHttpProxyEnabled = appProtection.rasp.isHttpProxyEnabled

// VPN active
let isVpnActive = appProtection.rasp.isVpnActive

// on call
let isOnCall = appProtection.rasp.isOnCall

// detected apps
let detectedApps = appProtection.rasp.installedApps
```

## Connect to Our Threat Intelligence Cloud

_(optional)_ In case you have access to our online console, you can configure credentials in our SDK so that the mobile app sends signals whenever a security policy violation is detected.

The credential consist of the following values:

- `API_USERNAME` - Username associated with your Android application ID.
- `API_PASSWORD` - API password value.
- `API_SIGNATURE_PUBLIC_KEY` - Public key that the SDK uses to verify authenticity of data received from the server.

You need to adjust your configuration in code the following way:

```swift
// Prepare the configuration for events
let eventConfig = AppProtectionEventConfig(
    enableEventCollection: true,
    enableScreenshotTakenCollection: true
)

// Prepare user identification
let idConfig = AppProtectionIdentificationConfig(
    userId: userId,
    deviceId: deviceId
)

// Prepare the Online configuration (optional)
let onlineConfig = .inAppProtectionOnlineConfig(
    username: "$USERNAME",
    password: "$PASSWORD",
    signaturePublicKey: "$PUBKEY",
    clientIdentification: idConfig,
    eventsConfig: eventConfig,
    customerGroupingConfig: nil,
    environment: .production
)

// Prepare a configuration for service
let config = AppProtectionConfig(
    raspConfig: raspConfig,
    onlineConfig: onlineConfig
)
```

After you compile and launch the iOS application with the online console access configured, the `AppProtection` class will scan the device and automatically send the impulse from an active device to the online service.

## App Hardening With App Shielding

_(optional)_ As an additional code obfuscation and app hardening step, you can include the additional compile step to your project. This process is referred to as "app shielding". The integration with Xcode is based on the command-line script - to add the step in your project, you need to:

1. Copy the provided tooling into your Xcode project folder (the `$PROJECT_DIR` variable in Xcode).
1. Set `FRAMEWORK_SEARCH_PATHS=$(inherited) $(PROJECT_DIR)/shielder/`
1. Prepare environment variables to be able to control the process properly.
1. Add a **New Run Script Phase** to your target build phases, with the following script as a content:

```sh
#set -x

if [[ "${APP_SHIELDING}" == "YES" ]]; then

    if [[ "${APP_SHIELDING_TRUST_THIS_BUILD}" == "YES" ]]; then
        /bin/bash "${SHIELD_SCRIPT}" "-config" "${SHIELD_CONFIG}" "-shielder" "${SHIELD_UTILITY}" "-framework" "${SHIELD_FRAMEWORK}" "--trustsigner"
    else
        /bin/bash "${SHIELD_SCRIPT}" "-config" "${SHIELD_CONFIG}" "-shielder" "${SHIELD_UTILITY}" "-framework" "${SHIELD_FRAMEWORK}"
    fi
    RESULT=$?
    if [[ $RESULT != 0 ]]; then
        if [[ "${GCC_PREPROCESSOR_DEFINITIONS}" == *"DEBUG=1"* ]]; then
            echo "warning: App shielding failed!"
        else
            echo "error: App shielding failed!"
            exit $RESULT
        fi
    fi
else
    echo "App shielding is turned off"
fi
```

## Summary

We just integrated the in-app protection into your iOS app and - if you configured the online access - sent the first data sample to the online service. After you launch your app to the public, the console will start to fill up with useful data about active insecure devices.

## Reference

- [In-app protection for Apple](/components/malwarelytics-apple/)
