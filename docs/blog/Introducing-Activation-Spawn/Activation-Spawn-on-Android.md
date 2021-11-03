# Activation Spawn on Android SDK
<!-- AUTHOR joshis_tweets 2021-09-17T00:00:00Z -->
<!-- SIDEBAR _Sidebar.md sticky -->
<!-- TEMPLATE post -->

This tutorial contains information on how to implement both parts of activation spawn (main and secondary, read [the activation spawn overview](./Readme.md) for more information) on Android.

## Installation

Add new Maven repositories pointing to Wultra Artifactory.

```groovy
repositories {
    maven {
        url 'https://wultra.jfrog.io/artifactory/activation-spawn-android/'
        credentials {
            username = wultraArtifactoryUsername
            password = wultraArtifactoryPass
        }
    }
    maven {
        url 'https://wultra.jfrog.io/artifactory/device-fingerprint-android/'
        credentials {
            username = wultraArtifactoryUsername
            password = wultraArtifactoryPass
        }
    }
}
```

Once you add the Maven repository, you can add activation spawn dependency to your Android project.

```groovy
android {
    dependencies {
        implementation "com.wultra.android.activationspawn:activation-spawn:${WULTRA_ACTIVATION_SPAWN_MANAGER}"
    }
}
```

## Main Application

### Secondary App Definition

If your application targets Android 11+ (SDK 30+), you need to declare the query "permissions" for the package name of Target App in the `AndroidManifest.xml` of the Source App. Otherwise, you won't be able to detect if the app is already installed.

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
package="com.yourcompany.yourapp">

    <queries>
         <!-- package name of your secondary app -->
        <package android:name="com.mycompany.secondaryapplication"/>
    </queries>

    <!-- rest of your manifest -->

</manifest>
```

Define the secondary apps that are available for activation. You can create the app representation by instantiating the `SpawnableApplication` class. It provides a package name, a deep link scheme, and a backend ID.

```kotlin
import com.wultra.android.activationspawn.SpawnableApplication

val application = SpawnableApplication(
    // package name of the app
    "com.company.android.application",
    // url deeplink scheme (for example from myAppScheme://settings)
    "myAppScheme",
    // needs to be provided by the powerauth backend administrator at your company
    "AA1122XYZ"
)
```

At any point in time, you can check if the secondary app is installed and if not, request the installation:

```kotlin
// manager is instance of ApplicationSpawnManager

try {
    if (manager.isInstalled(application)) {
        // app is installed
    } else {
        // when app is not installed, open it's store page
        manager.openStore(app)

    }
} catch (t: Throwable) {
   //handle isInstalled exception
}
```

### Obtaining Activation Data

Create `ActivationSpawnManager` instance.

```kotlin
import com.wultra.android.devicefingerprint.DeviceFingerprintGenerator
import com.wultra.android.activationspawn.createSpawnManager

// Additional data for generator (must be the same for both Source and Target App).
val additionalData: ByteArray? = null

// Note that the generator configuration must be the same
// for both Target and Source App. Please consult with Wultra
// what configuration suits your needs.
private val generator = DeviceFingerprintGenerator.getSemiStable(appContext, false, 10, additionalData)

// manager that handles the activation
// powerAuth is configured PowerAuthSDK instance
// appContext is application `Context` instance
private val manager = powerAuth.createSpawnManager(appContext, SSLValidationStrategy.default(), generator, "https://your-domain.com/your-app")
```

Retrieve activation data for the user:

```kotlin
import io.getlime.security.powerauth.sdk.PowerAuthAuthentication
import com.wultra.android.activationspawn.IRetrieveActivationDataListener
import com.wultra.android.activationspawn.ActivationSpawnData
import com.wultra.android.powerauth.networking.error.ApiError

val auth = PowerAuthAuthentication()
// prepare authentication object for 2FA
// ..
// ..

// manager is instance of ApplicationSpawnManager
manager.retrieveActivationData(app, auth, object : IRetrieveActivationDataListener {
    override fun onSuccess(data: ActivationSpawnData) {
        // activation data retrieved
    }

    override fun onError(error: ApiError) {
        // process the error
    }
})
```

### Forward Activation Data via Deep Link

6. Activate the app by forwarding the activation data via URL scheme.

```kotlin
try {
    // sharedInfo is a pre-shared value
    manager.transportDataToApp(data, application, sharedInfo)
} catch (t: Throwable) {
    // process the error
}
```

## Secondary App

### Defining Deep Link Scheme

Declare a deep link scheme in the `AndroidManifest.xml` for the application to enable digesting the deep link.

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.mycompany.secondaryapplication">
    <application>
        <activity
            android:name=".MyActivity">
            <intent-filter>
                <action android:name="android.intent.action.VIEW" />
                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.BROWSABLE" />
                <data
                    android:scheme="myAppScheme" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

### Performing Device Activation

When the secondary app is opened from the activation deep link, it needs to parse it using our helper methods. If the data is correctly retrieved, the app can proceed with a standard PowerAuth activation (under the hood, it is a standard activation via activation code and activation OTP):

```kotlin
import android.app.Activity
import android.os.Bundle
import com.wultra.android.activationspawn.*
import io.getlime.security.powerauth.networking.response.CreateActivationResult
import io.getlime.security.powerauth.networking.response.ICreateActivationListener
import io.getlime.security.powerauth.sdk.PowerAuthSDK

class TestActivity: Activity() {

    // dependencies you need to prepare
    private lateinit var manager: ActivationSpawnManager
    private lateinit var powerAuth: PowerAuthSDK

    // Additional data for transport (must be the same for both Source and Target App).
    private val sharedInfo: ByteArray? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val data = intent.data
        if (data != null) {
            try {
                val activationData = manager.processDeeplink(data, sharedInfo)
                powerAuth.createActivation(activationData, "Simon's phone", object : ICreateActivationListener {
                    override fun onActivationCreateSucceed(result: CreateActivationResult) {
                        // continue with the activation
                    }

                    override fun onActivationCreateFailed(t: Throwable) {
                        // process the error
                    }
                })
            } catch (t: Throwable) {
                // deeplink cannot be processed
            }
        }
    }
}
```

## Continue Reading

- [Overview](Readme.md#)
- [Activation Spawn on iOS](Activation-Spawn-on-iOS.md#)

## Reference

- [Activation Spawn API Reference](Activation-Spawn-API-Reference.md)
