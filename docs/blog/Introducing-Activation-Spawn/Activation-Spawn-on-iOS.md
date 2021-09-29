# Activation Spawn on iOS SDK
<!-- AUTHOR joshis_tweets 2021-09-17T00:00:00Z -->
<!-- SIDEBAR _Sidebar.md sticky -->
<!-- TEMPLATE post -->

This tutorial contains information on how to implement both parts of activation spawn (main and secondary, read [the activation spawn overview](./Readme.md) for more information) on iOS.

## Main Application

1. Your app needs to declare that it can activate such an app in `Info.plist`:

    ```xml
    <key>LSApplicationQueriesSchemes</key>
    <array>
        <string>apptoactivate</string>
    </array>
    ```

2. You need to add `WultraDeviceFingerprint`, `WultraActivationSpawn`, and `PowerAuth2` dependency to your project via Cocoapods.

    ```rb
    pod 'WultraDeviceFingerprint'
    pod 'PowerAuth2'
    pod 'WultraActivationSpawn'
    ```

    <!-- begin box info -->
    Note that `WultraActivationSpawn` and `WultraDeviceFingerprint` frameworks are not publicly available. [Follow this guide](Configuring-Private-Cocoapods-Repository.md) to configure your project to receive private Wultra libraries.
    <!-- end -->

3. Define the secondary apps that are available for activation.

    You can create the app representation by instantiating the `WASApplication` class. It provides a deep link scheme, an App Store ID and backend ID.

    ```swift
    let app = WASApplication(
        deeplinkScheme: "apptoactivate",
        appStoreIdentifier: 389801252,
        backendIdentifier: "MyApplicationToActivate"
    )
    ```

4. Check if the secondary app is installed:

    ```swift
    import WultraActivationSpawn

    // app is WASApplication instance
    do {
        if try app.isInstalled() {
            // app is installed
        } else {
            // when app is not installed, open it's store page sheet
            // self is a view controller, when using SwiftUI, you can replace
            // it with UIApplication.shared.windows.first?.rootViewController?
            self.openAppStoreProductPage(application: app) {
                print("Done!")
            }
        }
    } catch {
        // handle isInstalled error
    }
    ```

5. Retrieve activation data for the user:

    ```swift
    import WultraActivationSpawn
    import PowerAuth2

    let auth = PowerAuthAuthentication()
    // prepare authentication object for 2FA
    // ..
    // ..

    do {
        // powerauth is configured and activated PowerAuth SDK instance
        // app is WASApplication instance
        let activator = try WASActivator(powerAuth: powerauth, config: .init(sslValidation: .default))
        activator.retrieveActivationData(for: app, with: auth) { result in
            // process the data or error
        }
    } catch {
        // activator failed to be created
    }
    ```

6. Activate the app by forwarding the activation data via URL scheme.

    ```swift
    import WultraActivationSpawn
    import WultraDeviceFingerprint

    // Create transporter with fingerprint generator. This needs to be the same as in
    // the app that will receive the deep link. mainAppSecret is the pre-shared
    // value specific for the main app.
    let transporter: WASTransporter
    do {
        let generator = try DeviceFingerprintGenerator.stable(forVendor: false, withAdditionalData: mainAppSecret, validFor: 10)
        transporter = WASTransporter(generator: generator)
    } catch {
        // activator failed to create fingerprint generator
        return
    }

    // app is WASApplication
    // data is activation data retrieved from `retrieveActivationData` call
    // secondaryAppSecret is a pre-shared value specific for the secondary app.
    transporter.transport(data: data, to: app, with: secondaryAppSecret) { result in
        switch result {
        case .success:
            // activation data transported to the other app
        case .failure(let error):
            // activation data failed to transport
        }
    }
    ```

### Secondary App

1. The secondary app needs to declare a deep link which will be used for transportation in `Info.plist`:

    ```xml
    <key>CFBundleURLTypes</key>
    <array>
        <dict>
            <key>CFBundleTypeRole</key>
            <string>Editor</string>
            <key>CFBundleURLName</key>
            <string>deeplink</string>
            <key>CFBundleURLSchemes</key>
            <array>
                <string>apptoactivate</string>
            </array>
        </dict>
    </array>
    ```

2. When the secondary app is opened from the activation deep link, it needs to parse it using our helper methods. If the data is correctly retrieved, the app can proceed with a standard PowerAuth activation (under the hood, it is a standard activation via activation code and activation OTP):

    ```swift
    import WultraActivationSpawn
    import WultraDeviceFingerprint

    // Create transporter with fingerprint generator. This needs to be the same as in the app that creates the deep link.
    let transporter: WASTransporter
    do {
        let generator = try DeviceFingerprintGenerator.stable(forVendor: false, withAdditionalData: mainAppSecret, validFor: 10)
        transporter = WASTransporter(generator: generator)
    } catch {
        // failed to create fingerprint generator
        return
    }

    // url is the URL object retrieved from the deep link API provided by the UIKit
    // or SwiftUI, activation name is the name of the user device / model for
    // display purposes
    do {
        // sharedInfo is demo baked-in data
        let data = try transporter.process(deeplink: url, with: sharedInfo)
        // powerauth is configured but not activated `PowerAuthSDK` instance
        powerauth.createActivation(data: data, name: activationName) { result in
        	// process the activation result
        }
    } catch let e {
        // process the error
    }
    ```

## Continue Reading

- [Overview](Readme.md#)
- [Activaton Spawn on Android](Activation-Spawn-on-Android.md#)

## Reference

- [Activation Spawn API Reference](Activation-Spawn-API-Reference.md)
