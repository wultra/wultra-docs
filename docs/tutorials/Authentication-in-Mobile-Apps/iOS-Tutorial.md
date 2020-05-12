# Implementing Authentication in Mobile Banking Apps (SCA) on iOS Platform

<!-- AUTHOR joshis_tweets 2020-05-04T00:00:00Z -->

In this tutorial, we will show you how to implement authentication into your mobile banking or fintech app on iOS. This tutorial has four parts:

- [Mobile Authentication Overview](Readme.md)
- [Tutorial for Server Side Developers](Server-Side-Tutorial.md)
- **Tutorial for iOS Developers**
- [Tutorial for Android Developers](Android-Tutorial.md)

## Prerequisites

This tutorial assumes, that you have:

- Read and understood the [Mobile Authentication Overview](Readme.md)
- [Required back-end infrastructure up and running](Server-Side-Tutorial.md).
- Xcode 11.4+ with the Developer Tools installed

## Introduction

**/// TBD:**

## Getting the SDK

The easiest way to install the PowerAuth SDK into your project is using [Cocoapods](https://cocoapods.org/), by editing your Podfile:

```rb
platform :ios, '8.0'
target '<Your Target App>' do
  pod 'PowerAuth2'
end

# Disable bitcode for iOS targets (also see chapter Disabling bitcode)
post_install do |installer|
  installer.pods_project.targets.each do |target|
    if target.platform_name == :ios
      puts "Disabling bitcode for target  #{target.name}"
      target.build_configurations.each do |config|
        config.build_settings['ENABLE_BITCODE'] = 'NO'
      end
    end
  end
end
```

_Note: You need to disable bitcode for PowerAuth SDK to work._

After you change the Podfile, run the install command:

```sh
$ pod install
```

Finally, you can import PowerAuth SDK in any file where you need it:

```swift
import PowerAuth2
```

## Configuration

To configure your `PowerAuthSDK` instance, you need the following values from the PowerAuth Server:

- `APP_KEY` - Application key that binds an activation with a specific application.
- `APP_SECRET` - Application secret that binds an activation with a specific application.
- `KEY_MASTER_SERVER_PUBLIC` - Master Server Public Key used for non-personalized encryption and server signature verification.

Finally, you need to know the location of your [PowerAuth Standard RESTful API](https://github.com/wultra/powerauth-crypto/blob/develop/docs/Standard-RESTful-API.md) endpoints. That path should contain everything that goes before the `/pa/**` prefix of the API endpoints.

This is how the example `PowerAuthSDK` configuration looks like:

```swift
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {

    // Prepare the configuration
    let configuration = PowerAuthConfiguration()
    configuration.instanceId = Bundle.main.bundleIdentifier ?? ""
    configuration.appKey = "sbG8gd...MTIzNA=="
    configuration.appSecret = "aGVsbG...MTIzNA=="
    configuration.masterServerPublicKey = "MTIzNDU2Nz...jc4OTAxMg=="
    configuration.baseEndpointUrl = "https://localhost:8080/demo-server"

    // Configure default PowerAuthSDK instance
    PowerAuthSDK.initSharedInstance(configuration)

    return true
}
```

## Checking the Activation Status

After a user launches an application, you need to determine which user interface to show. Should you display a login UI? Or a new activation flow? Or an information that the device is blocked or removed?

Luckily, we have a simple method calls to obtain a detailed activation status:

```swift
// Check if there is some activation data on the device
if PowerAuthSDK.sharedInstance().hasValidActivation() {
    // If there is an activation on the device, check the status with server
    PowerAuthSDK.sharedInstance().fetchActivationStatus() { (status, customObject, error) in
        // If no error occurred, process the status
        if error == nil {
            // Show the UI relevant to the activaton status.
            self.presentUi(with: status)
        } else {
            // Network error occurred, report it to the user.
        }
    }
} else {
    // No activation is present on device.
    // Show the UI for a new activation.
}
```

##### Mockups

Here is an example mockup of the screens that need to be implemented:

![ Activation Status Check on iOS ](./03.png)

## New Activation

In the case no activation is present on the iOS device, you can guide the user through the steps to create it. Each activation has two major flows on the mobile device:

- **Creating the Activation** - Exchanging the user's identity proof with the server for the cryptographic activation data.
- **Committing the Activation** - Storing the cryptographic activation data on the device using the user's local credentials.

### Creating the Activation

The first step of a new activation process is exchanging the identity proof for the cryptographic activation data.

#### Using Activation Code

The easiest way to create the activation is using the PowerAuth activation code:

```swift
// Create the activation with an activation code
let deviceName = UIDevice.current.name
let activationCode = "VVVVV-VVVVV-VVVVV-VTFVA"

// Create activation object with given activation code.
guard let activation = PowerAuthActivation(activationCode: activationCode, name: deviceName) else {
    // Activation code is invalid
}

// Create a new activation with just created activation object
PowerAuthSDK.sharedInstance().createActivation(activation) { (result, error) in
    if error == nil {
        // No error occurred, proceed to credentials entry (PIN prompt, Enable Touch ID switch, ...) and commit
    } else {
        // An error occurred, report it to the user
    }
}
```

_Note: You can let the user scan the activation code from a QR code._

##### Mockups

Here is an example mockup of the screens that need to be implemented:

![ Activation Status Check on iOS ](./04.png)

#### Using Custom Credentials

In case you would like to use some other credentials your server supports, you can do so easily:

```swift
// Create a new activation with given device name and custom login credentials
let deviceName = UIDevice.current.name
let credentials = [
    "username": "john.doe@example.com",
    "password": "YBzBEM",
    "otp": "072471"
]

// Create activation object with given credentials.
guard let activation = PowerAuthActivation(identityAttributes: credentials, name: deviceName) else {
    // Activation credentials are empty
}

// Create a new activation with just created activation object
PowerAuthSDK.sharedInstance().createActivation(activation) { (result, error) in
    if error == nil {
        // No error occurred, proceed to credentials entry (PIN prompt, Enable Touch ID switch, ...) and commit
    } else {
        // Error occurred, report it to the user
    }
}
```

In the example, we used a combination of the username, password and an OTP generated elsewhere (for example, via a HW token, or delivered via SMS). But any credentials that you determined are suitable for activation will work.

##### Mockups

Here is an example mockup of the screens that need to be implemented:

![ Activation Status Check on iOS ](./05.png)

### Committing the Activation

After you successfully perform the steps for creating the activation, you can prompt the user to enter the new PIN code / password, allow an opt-in for the biometric authentication. After that, you can easily commit the newly created activation using the requested authentication factors:

```swift
do {
    let auth = PowerAuthAuthentication()
    auth.usePossession = true
    auth.usePassword   = "1234" // user PIN code
    auth.useBiometry   = true

    try PowerAuthSDK.sharedInstance().commitActivation(with: auth)
} catch _ {
    // happens only in case SDK was not configured or
    // when activation is not in the correct state to be committed
}
```

In most cases, the `usePossession` is set to `true` and `usePassword` to the value of the PIN or password user selected. The `useBiometry` value should be set to `true` in the case user decided to opt-in for biometric authentication, to `false` otherwise.

## Transaction Signing

In case you have successfully activated the device, you can use the new activation for future request signing.

### User Experience Perspective

To make the user experience consistent, we recommend making a solid UI abstraction on top of the transaction signing logic. This usually means implementing the transaction signing logic inside a unified PIN keyboard. Such keybord would than handle the typical use-cases people expect to see when working with PIN keyboards, such as:

- Entering a PIN code for the purpose of transaction signing.
- Alowing to use biometry as a faster alternative to PIN code.
- Checking the number of remaining attempts.
- Showing the transaction signing progress.
- Error reporting, activation status handling, etc.

To cover these use-cases efficiently, the PIN keyboard should be configurable with at least the following attributes:

- **The URI** - Basically the address that will be used for sending the signed request to.
- **The URI ID** - ***Be very careful here!*** In the examples below, we use a value of `uriId` for this value. While it is is remarkably similar to the end of an actual URI (we use `uri` in the example), this value is in fact an arbitrarily chosen constant that the client and server must agree on beforehand for a particular server-side operation represented. You need to ask your server developer for the value.
- **The HTTP request body** - This is basically the data that will be signed.
- **The HTTP medhod** - (Optional) The HTTP method to be used for the call. For the most cases, the calls should be made via the `POST` value and hence the "POST" value should be the default.
- **The HTTP headers** - (Optional) Value of all other HTTP headers you need to use when calling your service.

### Checking the Activation Status

We have covered this use-case earlier to frame the new activation flow. However, you should also check for the activation status before every attempt to use the trasnaction signing, since the activation might have been blocked or removed on the server side.

In case you check the activation status and the result is anything else than `.active`, you should cancel the transaction signing flow and redirect the user to the appropriate alternate flow, such as new activation wizard, unblocking tutorial, etc.

For the `.active` status, you should check if the number of failed attempts is greater than zero and show the UI for the number of remaining attempts in such case. The outline of the logic is the following:

{% codetabs %}
{% codetab Swift %}
```swift
// Check if there is some activation data on the device
if PowerAuthSDK.sharedInstance().hasValidActivation() {
    // If there is an activation on the device, check the status with server
    PowerAuthSDK.sharedInstance().fetchActivationStatus() { (status, customObject, error) in
        // If no error occurred, process the status
        if error == nil {
            if status.state == .active {
                if status.status.failCount > 0 {
                    self.remainingLabel.isHidden = false
                    self.remainingLabel.text = "Remaining attempts: " + status.remainingAttempts
                } else {
                    self.remainingLabel.isHidden = true
                }
                // ... see determining the biometry status
            } else {
                // Show the UI relevant to the activaton status.
                self.presentUi(with: status)
            }
        } else {
            // Network error occurred, report it to the user.
            self.presentNetworkError()
        }
    }
} else {
    // No activation is present on device.
    // Show the UI for a new activation.
    self.presentNewActivationUi()
}
```
{% endcodetab %}
{% codetab Objective-C %}
```objc
// Check if there is some activation data on the device
if ([[PowerAuthSDK sharedInstance] hasValidActivation]) {
    // If there is an activation on the device, check the status with server
    [[PowerAuthSDK sharedInstance] fetchActivationStatus:^ (status, customObject, error) {
    }
}
```
{% endcodetab %}
{% endcodetabs %}

### Determining the Biometry Status

In case the biometry is present and allowed by the user, you should trigger transaction signing using the biometry right away. To check the status of the biometry, you can use the following logic:

```swift
if PA2Keychain.canUseBiometricAuthentication && status.remainingAttempts > 2 && self.autoTriggerBiometry {
    self.biometryButton.enabled = true
    self.signWithBiometry()
} else {
    // Let user enter PIN code
    self.biometryButton.enabled = false
}
self.autoTriggerBiometry = false
```

Note that we not only decided to check for the mere usability of the biometry, but we also made sure that the user is not blocked by biometry in the case of the wet fingers. Also, you can notice the property `autoTriggerBiometry` that we use to automatically launch the biometry the first time unified PIN keyboard is opened.

### Request Signing

To sign the request data, you first need to prepare the `PowerAuthAuthentication` instance that specifies the authentication factors that you want to use. After that, you can compute the HTTP header with the signature and send the request to the server.

```swift
// Transaction signing with a biometry
func signWithBiometry() -> URLSessionDataTask? {
    let auth = PowerAuthAuthentication()
    auth.usePossession = true
    auth.useBiometry   = true
    return signWith(authentication: auth)
}

// Transaction signing with a password or a PIN code
func signWith(password: String) -> URLSessionDataTask? {
    let auth = PowerAuthAuthentication()
    auth.usePossession = true
    auth.usePassword   = password
    return signWith(authentication: auth)
}

// Transaction signing with an authentication object
func signWith(authentication: PowerAuthAuthentication) -> URLSessionDataTask? {
    do {
        // Get the request attributes
        let uri    = self.uri    // "https://my.server.example.com/payment"
        let uriId  = self.uriId  // "/payment"
        let method = self.method // "POST"
        let body   = self.body   // the serialized bytes of HTTP request body

        // Compute the signature header
        let signature = try PowerAuthSDK.sharedInstance().requestSignature(with: auth, method: method, uriId: uriId, body: body)
        let header = [ signature.key: signature.value ]

        // Send HTTP request with the HTTP header computed above
        // Note that we are sending the POST call to the actual URI, with
        // a computed HTTP header with signature and the request body bytes
        return self.httpClient.post(uri, header, body)
    } catch _ {
        // In case of invalid configuration, invalid activation
        // state or corrupted state data
        return nil
    }
}
```

You can hook the `signWithBiometry` method to the button for the biometric authentication and the `signWithPinCode` method to the PIN keyboard (for example, to be triggered when sufficiently long PIN code is entered by the user).

Note that the method returns an `URLSessionDataTask` instance, or a `nil` value in case there is an invalid state. In case the `URLSessionDataTask` is launched, you should wait for it to complete and check the HTTP status.

In case the HTTP status is `401` or `403`, it means that the transaction signing failed and in such case, you can simply restart the loop of checking the activation status, displaying the failure count, etc.

In case the HTTP status is `200`, it means that the transaction signing was successful. You can retrieve any data that you need from the response and close the PIN keyboard (ideally, with some nice victory animation:)).

## Resources

You can find more details about the iOS SDK in our reference documentation:

- [Mobile SDK for iOS and Android Documentation](https://github.com/wultra/powerauth-mobile-sdk)

## Continue Reading

Proceed with one of the following chapters:

- [Mobile Authentication Overview](Readme.md)
- [Tutorial for Server Side Developers](Server-Side-Tutorial.md)
- [Tutorial for Android Developers](Android-Tutorial.md)
