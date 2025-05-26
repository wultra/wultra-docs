# Implementing Authentication in Mobile Banking Apps (SCA) on the Android Platform

<!-- AUTHOR joshis_tweets 2020-05-04T00:00:00Z -->
<!-- SIDEBAR _Sidebar_Android.md sticky -->
<!-- TEMPLATE tutorial -->

In this tutorial, we will show you how to implement authentication into your mobile banking or fintech app on Android.

This tutorial has four parts:

- [Mobile Authentication Overview](Readme.md)
- [Tutorial for Server Side Developers](Server-Side-Tutorial.md)
- [Tutorial for iOS Developers](iOS-Tutorial.md)
- **Tutorial for Android Developers**

## Prerequisites

This tutorial assumes that you have:

- Read and understood the [Mobile Authentication Overview](Readme.md)
- [Required back-end infrastructure up and running](Server-Side-Tutorial.md).
- Android Studio and Android SDK (API level 21+).

## Introduction

When implementing the authentication flow on mobile, your task consists of building two major use-cases outlined in the [introductory overview documentation](Readme.md):

- Device activation
- Transaction signing

Of course, you will also need to implement several auxiliary use-cases, but these will become simple once you have device activation and transaction signing use-cases in place.

Our [Mobile Security Suite SDK](https://wultra.com/mobile-security-suite) will help you with the above-mentioned use cases. Since the underlying authentication protocol is called PowerAuth, the Mobile Security Suite SDK technical components inherit this naming. The "Mobile Security Suite SDK" is called the **PowerAuth SDK** on the technical level.

During the device activation flow, the SDK communicates with the public enrollment services. During the transaction signing, the SDK communicates with your server that publishes some protected resources (login, payment approval). The mobile app never communicates with the PowerAuth Server interface since this component is hidden deep in the secure infrastructure. See the component description in the [introductory overview documentation](Readme.md).

## Getting the SDK

The easiest way to install the PowerAuth SDK into your project is from JCenter. Simply edit your `gradle.build` file:

{% codetabs %}
{% codetab Gradle %}
```groovy
repositories {
    mavenCentral() // if not defined elsewhere...
}

dependencies {
    compile 'io.getlime.security.powerauth:powerauth-android-sdk:1.9.4'
}
```
{% endcodetab %}
{% endcodetabs %}

_Note: We use 1.9.4 in the example, replace the version number with the [latest release](https://github.com/wultra/powerauth-mobile-sdk/releases)._

## Configuration

To configure your `PowerAuthSDK` instance, you need the following values from the PowerAuth Server:

- `MOBILE_SDK_CONFIG ` - Base64 string with the cryptographic configuration.
- `BASE_ENDPOINT_URL` - The location of your [PowerAuth Standard RESTful API](https://github.com/wultra/powerauth-crypto/blob/develop/docs/Standard-RESTful-API.md) endpoints. The path should contain everything that goes before the `/pa/**` prefix of the API endpoints (usually _https://***/enrollment-server_)

All these values should be provided to you by your [server-side team](Server-Side-Tutorial.md), who configured the back-end infrastructure.

Use the provided values to configure the `PowerAuthSDK` instance:

{% codetabs %}
{% codetab Kotlin %}
```kotlin
val INSTANCE_ID = applicationContext.packageName
val MOBILE_SDK_CONFIG = "MTIzNDU2Nz...jc4OTAxMg=="
val API_SERVER = "https://<your-domain>/enrollment-server"

try {
    val configuration = PowerAuthConfiguration.Builder(
        INSTANCE_ID,
        API_SERVER,
        MOBILE_SDK_CONFIG)
        .build()
    val powerAuthSDK = PowerAuthSDK.Builder(configuration)
        .build(applicationContext)
} catch (exception: PowerAuthErrorException) {
    // Failed to construct `PowerAuthSDK` due to insufficient keychain protection.
    // (See next chapter for details)
}
```
{% endcodetab %}
{% endcodetabs %}

<!-- begin box warning -->
In case you use a development infrastructure with self-signed certificates, make sure to set the `PA2ClientSslNoValidationStrategy` instance to networking - see the [reference guide for more details](https://github.com/wultra/powerauth-mobile-sdk/blob/develop/docs/PowerAuth-SDK-for-Android.md#working-with-invalid-ssl-certificates).
<!-- end -->

## Checking the Activation Status

After a user launches an application, you need to determine which user interface to show. Should you display a login UI? Or a new activation flow? Or is an information that the device is blocked or removed?

Luckily, we have a simple method to obtain a detailed activation status:

{% codetabs %}
{% codetab Kotlin %}
```kotlin
// Check if there is some activation on the device
if (powerAuthSDK.hasValidActivation()) {
    // If there is an activation on the device, check the status with server
    powerAuthSDK.fetchActivationStatusWithCallback(context, object: IActivationStatusListener {
        override fun onActivationStatusSucceed(status: ActivationStatus) {
            // process the state
        }

        override fun onActivationStatusFailed(t: Throwable) {
            // Network error occurred, report it to the user
        }
    })
} else {
    // No activation present on device
}
```
{% endcodetab %}
{% endcodetabs %}

##### Mockups

Here is an example mockup of the screens that need to be implemented:

![ Activation Status Check on Android ](./03.png)

## New Activation

In the case that no usable activation is available on the Android device, you can guide the user through the steps to create it. Each activation has two major flows on the mobile device:

- **Creating the Activation** - Exchanging the user's identity proof with the server for the cryptographic activation data.
- **Persisting the Activation** - Storing the cryptographic activation data on the device using the user's local credentials.

<!-- begin box info -->
Thanks to this step, the user credentials, such as PIN code or biometric information, never leave the mobile device and are only used locally to obtain cryptographic data required during the transaction signing.
<!-- end -->

### Creating the Activation

In the first step of a new activation process, you need to exchange the user's proof of identity for the cryptographic activation data. There are several ways to accomplish this.

#### Using Activation Code

The easiest way to create an activation is using the PowerAuth activation code that you will obtain from the back-end part (for example, via a QR code shown in the Internet banking):

{% codetabs %}
{% codetab Kotlin %}
```kotlin
val deviceName = "Petr's Leagoo T5C"
val activationCode = "VVVVV-VVVVV-VVVVV-VTFVA" // let user type or QR-scan this value

// Create activation object with given activation code.
val activation: PowerAuthActivation
try {
    activation = PowerAuthActivation.Builder.activation(activationCode, deviceName).build()
} catch (e: PowerAuthErrorException) {
    // Invalid activation code
}

// Create a new activation with the given activation object
powerAuthSDK.createActivation(activation, object: ICreateActivationListener {
    override fun onActivationCreateSucceed(result: CreateActivationResult) {
        val fingerprint = result.activationFingerprint
        // No error occurred, proceed to credentials entry (PIN prompt, Enable "Fingerprint Authentication" switch, ...) and persist
        // The 'fingerprint' value represents the combination of device and server public keys - it may be used as visual confirmation
    }

    override fun onActivationCreateFailed(t: Throwable) {
        // Error occurred, report it to the user
    }
})
```
{% endcodetab %}
{% endcodetabs %}


<!-- begin box info -->
You can let the user scan the activation code from a QR code to make the process faster and improve user convenience, but you should also allow manual entry as a backup.
<!-- end -->

##### Mockups

Here is an example mockup of the screens that need to be implemented:

![ Activation Status Check on Android ](./04.png)

#### Using Custom Credentials

Alternatively, you can use some other credentials your server supports to create a new activation. You always need to spend some thought on which credentials you should use. You do not want to use credentials that are too weak. **The authentication proof resulting from transaction signing is only as strong as the credentials that were used during the activation flow.**

However, once you have the credentials that are sufficiently strong, you can create an activation easily:

{% codetabs %}
{% codetab Kotlin %}
```kotlin
// Create a new activation with a given device name and login credentials
val deviceName = "Juraj's JiaYu S3"
val credentials = mapOf("username" to "john.doe@example.com", "password" to "YBzBEM")

// Create an activation object with the given credentials.
val activation: PowerAuthActivation
try {
    activation = PowerAuthActivation.Builder.customActivation(credentials, deviceName).build()
} catch (e: PowerAuthErrorException) {
    // Credentials dictionary is empty
}

// Create a new activation with the given activation object
powerAuthSDK.createActivation(activation, object: ICreateActivationListener {
    override fun onActivationCreateSucceed(result: CreateActivationResult) {
        val fingerprint = result.activationFingerprint
        // No error occurred, proceed to credentials entry (PIN prompt, Enable "Biometric Authentication" switch, ...) and persist
        // The 'fingerprint' value represents the combination of device and server public keys - it may be used as visual confirmation
    }

    override fun onActivationCreateFailed(t: Throwable) {
        // Error occurred, report it to the user
    }
})
```
{% endcodetab %}
{% endcodetabs %}

In the example, we used a combination of the username, password, and an OTP generated elsewhere (for example, via a hardware token or delivered via SMS). But any credentials that you determined are suitable for activation will work.

##### Mockups

Here is an example mockup of the screens that need to be implemented:

![ Activation Status Check on iOS ](./05.png)

### Persisting the Activation

After you successfully perform the steps for creating the activation, you can prompt the user to enter the new local PIN code / password and allow an opt-in for the biometric authentication (of course, [only in the case the device supports biometrics](https://github.com/wultra/powerauth-mobile-sdk/blob/develop/docs/PowerAuth-SDK-for-Android.md#biometric-authentication-setup)).

You can now persist the newly created activation with the requested authentication factors.

#### Persisting With Biometry

In the case the user decides to opt in for the biometry, use the following code:

{% codetabs %}
{% codetab Kotlin %}
```kotlin
// Persist activation using given PIN
val authentication = PowerAuthAuthentication.persistWithPassword(pin)
val cancelable = powerAuthSDK.persistActivationWithAuthentication(context, authentication, object: IPersistActivationListener {
    override fun onPersistActivationSucceeded() {
        // Success
    }

    override fun onPersistActivationFailed(error: PowerAuthErrorException) {
        // Failure
    }

    override fun onPersistActivationCancelled(userCancel: Boolean) {
        if (userCancel) {
            // user cancelled the biometric authentication dialog
        } else {
            // Your application canceled the provided cancelable object
        }
    }
})
```
{% endcodetab %}
{% endcodetabs %}

Note that we automatically handle the selection of the appropriate biometric approach. We use [Unified biometric authentication dialog](https://developer.android.com/about/versions/pie/android-9.0#biometric-auth) if available, and we fall back to the fingerprint authentication on older devices or devices that are known to have issues with the newer biometric approaches.

#### Persisting Without Biometry

In the case the user decides to decline biometry or in the case biometry is not available on the device, you can use this simpler call:

{% codetabs %}
{% codetab Kotlin %}
```kotlin
// persist activation using given PIN
val result = powerAuthSDK.persistActivationWithPassword(appContext, pin)
if (result != PowerAuthErrorCodes.SUCCEED)
    // happens only in case the SDK was not configured or activation is not in a state to be persisted
}
```
{% endcodetab %}
{% endcodetabs %}

## Transaction Signing

In case you successfully activated the device, you can use the new activation for transaction signing. This is achieved by signing the full request data. In case of the HTTP request with a body (`POST`, `PUT`, `DELETE`), the HTTP request body is used. In case of the `GET` request, the SDK builds canonical data from the URL query parameters.

The transaction signing requires absolute precision. Every single bit makes a huge difference, just one step further in the process. Be patient and do not worry if transaction signing doesn't work the first time you try. In case you are having issues, do not hesitate to ask our engineers for help.

### User Experience Perspective

To make the user experience consistent, we recommend making a solid UI abstraction on top of the transaction signing logic. This usually means implementing the transaction signing logic inside a **unified PIN keyboard**. Such a keyboard would then handle the typical use-cases people expect to see when working with PIN keyboards, such as:

- Entering a PIN code for the purpose of transaction signing.
- Allowing to use of biometrics as a faster alternative to the PIN code.
- Checking the number of remaining authentication attempts.
- Showing the transaction signing progress.
- Error reporting, invalid activation status handling, etc.

The following picture shows the anatomy of a well-designed PIN keyboard:

![ Anatomy of a PIN keyboard ](./06.png)

Another thing to consider is the high-level user flow. The overview of the flow stages is captured in the following diagram:

![ Authentication flow ](./07.png)

You can read more information about the authentication flow in the chapters below.

### Configuring the Unified PIN Keyboard

To cover the typical use-cases efficiently, the unified PIN keyboard should be configurable with at least the following attributes:

- **The URI Address** - Basically, the service location that will be called while sending the signed request using an HTTP client of your choice. We use the `uri` variable in the example.
- **The URI ID** - Identifier of the URI / service. **Be very careful here!** In the examples below, we use a value of `uriId` for this value. While it is remarkably similar to the end of an actual URI (`uri`), this value is in fact an arbitrarily chosen constant that the client and server must agree on beforehand for a particular server-side operation represented. You need to ask your server developer for the exact value.
- **The HTTP request data** - This is basically the data that will be signed. For the `POST` requests (that are the most common in the case of transaction signing), this value represents simply the HTTP request body bytes.
- **The HTTP method** - (Optional) The HTTP method to be used for the call. In most cases, the calls should be made via the `POST` value, and hence, the `POST` value should be the default.
- **The HTTP headers** - (Optional) Value of any other HTTP headers you need to use when calling your service.

### Checking the Activation Status Before Signing

We covered a similar use case earlier in the context of the new activation flow. However, you should also check for the activation status before every attempt to use the transaction signing, since the activation might have been blocked or removed on the server side.

In case you check the activation status and the result is anything else than `State_Active`, you should cancel the transaction signing flow and redirect the user to the appropriate alternate flow, such as a new activation wizard, unblocking tutorial, etc.

For the `State_Active` status, you should check if the number of failed attempts is greater than zero and show the UI for the number of remaining attempts in such a case. The outline of the logic is the following:

{% codetabs %}
{% codetab Kotlin %}
```kotlin
// Check if there is some activation on the device
if (powerAuthSDK.hasValidActivation()) {
    // If there is an activation on the device, check the status with server
    powerAuthSDK.fetchActivationStatusWithCallback(context, object: IActivationStatusListener {
        override fun onActivationStatusSucceed(status: ActivationStatus) {
            // Activation states are explained in detail in "Activation states" chapter below
            when (status.state) {
                ActivationStatus.State_Pending_Commit ->
                    Log.i(TAG, "Waiting for commit")
                ActivationStatus.State_Active ->
                    Log.i(TAG, "Activation is active")
                ActivationStatus.State_Blocked ->
                    Log.i(TAG, "Activation is blocked")
                ActivationStatus.State_Removed -> {
                    Log.i(TAG, "Activation is no longer valid")
                    powerAuthSDK.removeActivationLocal(context)
                }
                ActivationStatus.State_Deadlock -> {
                    Log.i(TAG, "Activation is technically blocked")
                    powerAuthSDK.removeActivationLocal(context)
                }
                ActivationStatus.State_Created -> Log.i(TAG, "Unknown state")
                else -> Log.i(TAG, "Unknown state")
            }

            // Failed login attempts, remaining = max - current
            val currentFailCount: Int = status.failCount
            val maxAllowedFailCount: Int = status.maxFailCount
            val remainingFailCount: Int = status.remainingAttempts
            if (status.customObject != null) {
                // Custom object contains any proprietary server-specific data
            }
        }

        override fun onActivationStatusFailed(t: Throwable) {
            // Network error occurred, report it to the user
        }
    })
} else {
    // No activation present on device
}
```
{% endcodetab %}
{% endcodetabs %}

### Determining the Biometry Status

In case the biometry is present and allowed by the user, you should trigger transaction signing using the biometry right away when the unified PIN keyboard is first shown.

To check the status of the biometry, you can use the following logic:

{% codetabs %}
{% codetab Kotlin %}
```kotlin
if (BiometricAuthentication.isBiometricAuthenticationAvailable(context) && status.remainingAttempts > 2 && this.autoTriggerBiometry) {
    this.enableBiometryButton(true)
    this.signWithBiometry()
} else {
    // Let the user enter the PIN code
    this.enableBiometryButton(false)
}
this.autoTriggerBiometry = false
```
{% endcodetab %}
{% endcodetabs %}

Note that we decided to not only check for the mere availability of the biometry, but we also check for the remaining attempt count to make sure the user is not blocked in case of "wet fingers". Also, you can notice the property `autoTriggerBiometry` that we use to automatically launch the biometry only the first time a unified PIN keyboard is opened. For 2nd and further authentication attempts, we want the user to trigger the biometric authentication manually.

### Request Signing

To sign the request data, you first need to prepare a `PowerAuthAuthentication` instance that specifies the authentication factors that you want to use. After that, you can compute the HTTP header with the signature and send the request to the server.

{% codetabs %}
{% codetab Kotlin %}
```kotlin
// Transaction signing with biometrics
fun signWithBiometry(listener: IMyAuthListener) {
    // Authenticate user with biometry and obtain encrypted biometry factor-related key.
    powerAuthSDK.authenticateUsingBiometrics(context, fragmentManager, "Sign in", "Use the biometric sensor on your device to continue", object: IAuthenticateWithBiometricsListener {
        override fun onBiometricDialogCancelled(userCancel: Boolean) {
            // User cancelled the operation
            listener.authenticationCancelled();
        }

        override fun onBiometricDialogSuccess(authentication: PowerAuthAuthentication) {
            signWithAuthentication(auth, listener);
        }

        override fun onBiometricDialogFailed(error: PowerAuthErrorException) {
            listener.authenticationFailed(error);
        }
    })
}

// Transaction signing with a password or a PIN code
fun signWithPassword(password: Password, listener: IMyAuthListener) {
    val  authentication = PowerAuthAuthentication.possessionWithPassword(password)
    signWithAuthentication(auth, listener)
}

// Transaction signing with an authentication object
fun signWithAuthentication(auth: PowerAuthAuthentication, listener: IMyAuthListener) {
    // Get the request attributes
    val uri    = this.uri    // "https://my.server.example.com/payment"
    val uriId  = this.uriId  // "/payment"
    val method = this.method // "POST"
    val body   = this.body   // the serialized bytes of HTTP request body

    // Compute the signature header
    val header = powerAuthSDK.requestSignatureWithAuthentication(context, auth, method, uriId, body)
    if (!header.isValid) {
        listener.authenticationFailed(Reason.CRYPTO);
        return
    }
    val header = HttpHeader(header.key, header.value)

    // Send an HTTP request with the HTTP header computed above.
    // Note that we are sending the POST call to the service URI, with
    // a computed signature HTTP header and the request body bytes
    this.httpClient.post(uri, header, body, object: IMyHttpClientListener {
        fun networkSuccess(statusCode: Int, headers: List<Header>, responseBody: ByteArray) {
            if (statusCode == 200) {
                listener.authenticationSuccess(headers, responseBody)
            } else if (statusCode == 401 || statusCode == 403) {
                listener.authenticationFailed(Reason.UNAUTHORIZED)
            } else {
                listener.authenticationFailed(Reason.OTHER)
            }
        }

        fun networkFailed() {
            // Networking failed
            listener.authenticationFailed(Reason.NETWORKING)
        }
    })
}
```
{% endcodetab %}
{% endcodetabs %}

You can hook the `signWithBiometry` method to the button for the biometric authentication and the `signWithPassword` method to the PIN keyboard (for example, to be triggered when a sufficiently long PIN code is entered by the user).

Note that the method in our example uses an `IMyAuthListener` interface. This is a placeholder interface example, you should implement your own interface that will allow you to process the authentication result asynchronously, including handling the biometric results and the entire networking cycle.

On the networking layer, we used a similar `IMyHttpClientListener` placeholder interface, you should handle the following business logic when calling the PowerAuth protected resources:

- In case the HTTP status is `401` or `403`, it means that the transaction signing failed, and in such a case, you can simply restart the loop of checking the activation status, displaying the failure count, etc.
- In case the HTTP status is `200`, it means that the transaction signing was successful. You can retrieve any data that you need from the response and close the PIN keyboard (ideally, with some nice "victory animation").


## Resources

You can find more details about the Android SDK in our reference documentation:

- [Mobile SDK for iOS and Android Documentation](https://github.com/wultra/powerauth-mobile-sdk)

## Continue Reading

Proceed with one of the following chapters:

- [Mobile Authentication Overview](Readme.md)
- [Tutorial for Server Side Developers](Server-Side-Tutorial.md)
- [Tutorial for iOS Developers](iOS-Tutorial.md)
