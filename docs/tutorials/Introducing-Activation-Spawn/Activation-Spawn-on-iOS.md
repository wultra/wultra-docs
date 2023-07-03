# Activation Spawn on iOS SDK
<!-- AUTHOR joshis_tweets 2023-06-26T00:00:00Z -->
<!-- SIDEBAR _Sidebar.md sticky -->
<!-- TEMPLATE tutorial -->

This tutorial contains information on how to implement both parts of activation spawn (main and secondary, read [the activation spawn overview](./Readme.md) for more information) on iOS.

## Installation

<!-- begin box info -->
Note that `WultraActivationSpawn` and `WultraDeviceFingerprint` frameworks are not publicly available.
<!-- end -->

### Dependencies

- [PowerAuth SDK for Mobile Apps](https://github.com/wultra/powerauth-mobile-sdk)
- [PowerAuth Networking for Apple platforms](https://github.com/wultra/networking-apple)
- Wultra Device Fingerprint for Apple (closed source, available via Wultra repository)

### Swift Package Manager

1. Create (or append to if already exists) `~/.netrc` file in your home directory with the following credentials you were provided alongside this document: 
   ```
   machine wultra.jfrog.io
         login [name@yourcompany.com]
         password [password]
   ```

2. Add the following repositories as a dependency into your project:
   ```
   https://github.com/wultra/networking-apple
   https://github.com/wultra/activation-spawn-apple-release
   https://github.com/wultra/device-fingerprint-apple-release
   ```
   You can use Xcode's dedicated user interface to do this or add the dependency manually, for example:
   
   ```swift
   // swift-tools-version:5.7

   import PackageDescription

   let package = Package(
       name: "YourLibrary",
       products: [
           .library(
               name: "YourLibrary",
               targets: ["YourLibrary"]),
       ],
       dependencies: [
           .package(name: "WultraPowerAuthNetworking", url: "https://github.com/wultra/networking-apple.git", .upToNextMajor(from: "1.1.0")),
           .package(name: "WultraActivationSpawn", url: "https://github.com/wultra/activation-spawn-apple-release.git", .upToNextMajor(from: "1.3.0")),
           .package(name: "WultraDeviceFingerprint", url: "https://github.com/wultra/device-fingerprint-apple-release.git", .upToNextMajor(from: "1.3.0"))
       ],
       targets: [
           .target(
               name: "YourLibrary",
               dependencies: ["WultraActivationSpawn", "WultraPowerAuthNetworking", "WultraDeviceFingerprint" ]
            )
       ]
   )
   ```

### CocoaPods

The library is distributed through a public git repository, which contains a podspec and scripts to download the framework from a private artifactory. If you're not using cocoapods in your project, visit [usage guide](https://guides.cocoapods.org/using/using-cocoapods.html).

1. Add pod to your `Podfile`:
   ```rb
   target 'MyProject' do
       use_frameworks!
       pod 'WultraActivationSpawn', :git => 'https://github.com/wultra/activation-spawn-apple-release.git', :tag => '1.3.0'
       pod 'WultraDeviceFingerprint', :git => 'https://github.com/wultra/device-fingerprint-apple-release.git', :tag => '1.3.2'
   end
   ```
   You can check the latest versions of libraries above at release pages:
   - [WultraActivationSpawn releases page](https://github.com/wultra/device-fingerprint-apple-release/releases)
   - [WultraDeviceFingerprint releases page](https://github.com/wultra/activation-spawn-apple-release/releases)

2. Run `pod install` in your project dictionary to make the `WultraActivationSpawn` and `WultraDeviceFingerprint` frameworks available in your project.

## Main Application

### Secondary App Definition

Define the secondary apps that are available for activation.

You can create the app representation by instantiating the `WASApplication` class. It provides a deep link scheme, an App Store ID and backend ID.

```swift
let app = WASApplication(
    deeplinkScheme: "apptoactivate",
    appStoreIdentifier: 389801252,
    backendIdentifier: "MyApplicationToActivate"
)
```

### Checking for Application Installation

At any point in time, you can check if the secondary app is installed and if not, request the installation:

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

### Defining Deep Link Scheme

The app needs to declare which secondary apps it can activate in `Info.plist`:

```xml
<key>LSApplicationQueriesSchemes</key>
<array>
    <string>apptoactivate</string>
</array>
```

### Obtaining `WASActivator`

You can create `WASActivator` instance in the following manner:

```swift
import WultraActivationSpawn
import PowerAuth2

do {
    // powerauth is a PowerAuth SDK instance
    // app is WASApplication instance
    let activator = try WASActivator(powerAuth: powerauth, config: .init(sslValidation: .default))

} catch {
    // activator failed to be created
}
```

### Obtaining Activation Data

In case you are using your own authentication scheme, you can fetch the data using your authenticated service.

When using PowerAuth for authentication, you can retrieve activation data for the user in the following manner:

```swift
let auth = PowerAuthAuthentication()
// prepare authentication object for 2FA
// ..
// ..

activator.retrieveActivationData(for: app, with: auth) { result in
    // process the data or error
}
```

### Forward Activation Data via Deep Link

Activate the app by forwarding the activation data via the URL scheme.

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

## Secondary App

### Defining Deep Link Scheme

The secondary app needs to declare own deep link URL scheme which will be used for transportation in `Info.plist`:

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

### Performing Device Activation

When the secondary app is opened from the activation deep link, it needs to parse it using our helper methods. If the data is correctly retrieved, the app can proceed with a standard PowerAuth activation (under the hood, it is a standard activation via activation code and activation OTP):

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
- [Activation Spawn on Android](Activation-Spawn-on-Android.md#)

## Reference

- [Activation Spawn API Reference](Activation-Spawn-API-Reference.md)
