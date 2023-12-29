# Implementing Malwarelytics on Android

<!-- AUTHOR joshis_tweets 2020-05-04T00:00:00Z -->
<!-- SIDEBAR _auto -->
<!-- TEMPLATE tutorial -->

This tutorial explains how you can integrate Malwarelytics - an advanced mobile threat intelligence and in-app protection - into your Android app.

<!-- begin box warning -->
**Personalized Configuration Required.**<br/>
In order to use Malwarelytics for Android, you need a custom configuration and access credentials for both the service and artifact repository. Contact your sales representative or technical consultant in order to obtain the required prerequisites.
<!-- end -->

## Get the Malwarelytics SDK

You can obtain the Malwarelytics SDK for Android by clicking the **Get the SDK!** item in the left navigation. In the **Download the SDK** section, you will find your Artifactory credentials. To add a new Maven repository pointing to the Wultra Artifactory, include the following snippet in your project Gradle file:

```gradle
repositories {
    maven {
        url 'https://wultra.jfrog.io/artifactory/malwarelytics-android/'
        credentials {
            username = wultraArtifactoryUsername
            password = wultraArtifactoryPass
        }
    }
}
```

You can then add the app protection dependency to your Android project.

```gradle
android {
    dependencies {
        implementation "com.wultra.android.antimalware:antivirus:${WULTRA_APP_PROTECTION_VERSION}"
    }
}
```

## Obtain the API Credentials

In the **Set Up Your API Keys** section in the Malwarelytics Console, select the application for which you need to obtain the configuration.

![ Downloading the SDK ](./03.png)

You will find the following values:

- `API_USERNAME` - Username associated with your Android application ID.
- `API_PASSWORD` - API password value.
- `API_SIGNATURE_PUBLIC_KEY` - Public key that the SDK uses to verify authenticity of data received from the server.

## Integrate the SDK

To integrate the SDK in your app, insert the following code snippet to your `Application.onCreate()` method:

```kotlin
class MyApplication : Application() {

    override fun onCreate() {
        // Prepare minimal configuration
        val config = AppProtectionConfig.Builder(appContext)
            .apiUsername(API_USERNAME)
            .apiPassword(API_PASSWORD)
            .apiSignaturePublicKey(API_SIGNATURE_PUBLIC_KEY)
            .clientAppUserId(INTERNAL_CLIENT_ID) // Use if the internal user ID is available at config time
            .antivirusConfig(
                AntivirusConfig.Builder()
                        .build()
            )
            .raspConfig(
                RaspConfig.Builder()
                        // Configure RASP features here
                        .signatureHash(SIGNATURE_HASH)
                        .build()
            )
            .build()

        // Initialize AppProtection class
        val appProtection = AppProtection.getInstance()
        appProtection.initializeAsync(config, object: AppProtection.InitializationObserver {
            // App Protection is fully ready to be used now
            override fun onInitialized() {
                // Setup internal user ID if you are able to obtain it
                appProtection.updateClientAppUserId(INTERNAL_CLIENT_ID)
            }
        })   
        // ...           
    }
}
```

<!-- begin box info -->
Do not forget to set your internal user ID. Otherwise, it will be difficult to identify which users devices are insecure, i.e., infected with malware. In the ideal situation, you can use an opaque random device identifier as the `INTERNAL_USER_ID`. However, any identifier that you can use to look up the infected device or the user will do the trick.
<!-- end -->

After you compile and launch the Android application, Malwarelytics scan automatically starts and it sends the first impulse from your active device to the Malwarelytics service.

![ Downloading the SDK ](./04.png)

## Summary

We just integrated the Malwarelytics SDK into your Android app and sent the first data sample to the Malwarelytics service. After you launch your app to the public, your Malwarelytics console will start to fill up with useful data about active insecure devices and mobile malware.

## Read Next

- [Malwarelytics for Android Documentation](https://github.com/wultra/antivirus)
