# In-App Protection&#58; Implementing on Android

<!-- AUTHOR joshis_tweets 2023-12-29T00:00:00Z -->
<!-- SIDEBAR _auto -->
<!-- TEMPLATE tutorial -->

This tutorial summarizes how to integrate Wultra's in-app protection into your iOS app.

<!-- begin box info -->
In-app protection for Android supports Android 5.1 (SDK version 22) and above.
<!-- end -->

<!-- begin box warning -->
**Personalized Configuration Required.**<br/>
In order to use in-app protection for Android, you need a custom configuration and access credentials. Contact our sales representatives or technical consultant in order to obtain the required prerequisites.
<!-- end -->

## Get the In-App Protection SDK

You can obtain the in-app protection SDK for Apple from our Artifactory.

To add a Maven repository pointing to the Wultra Artifactory, include the following snippet in your project Gradle file:

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

## Implement Configuration in Your App

The main class for in-app protection SDK for Android is `AppProtection`. You are responsible for configuring the instance as soon as possible after the application launch. You should also implement the `RaspObserver` to handle RASP callbacks. The easiest way to initialize the in-app protection SDK in the `Application.onCreate()` system callback.

The following snippet contains the minimal configuration with default detection behaviors:

```java
class MyApplication : Application() {

    override fun onCreate() {
        val config = AppProtectionConfig.Builder(appContext)
            .raspConfig(
                RaspConfig.Builder()
                    .repackage(
                        RepackageDetectionConfig.Builder()
                            .signatureHash(listOf(SIGNATURE_HASH))
                            .build()
                    )
                    .build()
            )
            .build()

        val appProtection = AppProtection.getInstance()
        appProtection.initializeAsync(config, object: AppProtection.InitializationObserver {
            override fun onInitialized(initializationResult: InitializationResult) {
                // App Protection is fully ready to be used now
            }
        })
    }
}
```

The configuration can be extended with the following builder methods:

```java
val config = AppProtectionConfig.Builder(appContext)
    .raspConfig(
        RaspConfig.Builder()
            .emulator(DetectionConfig.Notify)
            .root(RootDetectionConfig.Notify)
            .debugger(
                DebuggerDetectionConfig.Builder()
                    .action(DetectionConfig.Nofify)
                    .debuggerTypes(DebuggerType.values().toList())
                    .build()
            )
            .repackage(
                RepackageDetectionConfig.Builder()
                    .action(DetectionConfig.Exit())
                    .signatureHash(listOf("SHA1-SIGNER-CERTIFICATE"))
                    .build()
            )
            .screenSharing(DetectionConfig.Notify)
            .screenshot(BlockConfig.Block)
            .screenReader(
                ScreenReaderBlockConfig.Builder()
                    .action(BlockConfig.Block)
                    .allowedScreenReaders(ScreenReaderBlockConfig.DEFAULT_ALLOWED_SCREEN_READERS)
                    .build()
            )
            .processName(ProcessNameConfig.UseStealthy(null))
            .tapjacking(
                TapjackingBlockConfig.Builder()
                    .action(BlockConfig.Block)
                    .ignoreTapjackingSystemApps(false)
                    .blockTapjackingSensitivity(ThreatIndex.HIGHLY_DANGEROUS)
                    .allowedTapjackingApps(setOf())
                    .build()
            )
            .httpProxy(DetectionConfig.Notify)
            .vpn(DetectionConfig.Notify)
            .adb(AdbDetectionConfig.Notify)
            .activeCall(SimpleDetectionConfig.Notify)
            .appPresence(
                AppPresenceDetectionConfig.Builder()
                    .action(DetectionConfig.Notify)
                    .remoteDesktopApps(AppPresenceDetectionConfig.DEFAULT_REMOTE_DESKTOP_APPS)
                    .build()
            )
            .build()
    )
    .build()
```

### Configuring Repackaging Detection

Repackaging detection is a security feature that detects if the application was modified and resigned with a different signing certificate.

To properly configure the repackage detection, you need to get the SHA1 hash your signing certificate. The process generally depends on how your application is signed. We detail the topic in the [reference documentation](/components/malwarelytics-android/develop/documentation/Repackaging-Detection).

Once you obtain the certificate hash, you can set the value to the configuration class:

```java
val raspConfig = RaspConfig.Builder()
    .repackage(
        RepackageDetectionConfig.Builder()
            .action(DetectionConfig.Exit())
            .signatureHash(listOf("SHA1-SIGNER-CERTIFICATE"))
            .build()
    )
    // configuration of other RASP features
    .build()
```

### Obtaining the Detection Results

You can observe for the detection results by registering the `RaspObserver` class:

```java
val raspObserver = object : RaspObserver {
    override fun onEmulatorDetected(emulatorDetection: EmulatorDetection) {
        // handle emulator detection
    }

    override fun onRootDetected(rootDetection: RootDetection) {
        // handle root detection
    }

    override fun onDebuggerDetected(debuggerDetected: Boolean) {
        // handle debugger detection
    }

    override fun onRepackagingDetected(repackagingResult: RepackagingResult) {
        // handle repackaging detection
    }

    override fun onScreenSharingDetected(screenSharingDetected: Boolean) {
        // handle screen sharing detection
    }

    override fun onTapjackingDetected(tapjackingDetection: TapjackingDetection) {
        // handle tapjacking detection
    }

    override fun onHttpProxyDetected(httpProxyDetected: Boolean) {
        // handle HTTP proxy detection
    }

    override fun onVpnDetected(vpnEnabled: Boolean) {
        // handle VPN detection
    }

    override fun onAdbStatusDetected(adbStatus: Boolean) {
        // handle ADB status detection
    }
    
    override fun onActiveCallDetected(activeCallDetection: ActiveCallDetection) {
        // handle active call detection
    }

    override fun onAppPresenceChanged(appPresenceDetection: AppPresenceDetection) {
        // handle app presence detection
    }
}

appProtection.getRaspManager().registerRaspObserver(raspObserver)
```

Besides responding to callbacks in the observer, you can also access any RASP detections just-in-time, like so:

```java
// root detection
val rootDetection = raspManager.getRootDetection()
val isRooted = raspManager.isDeviceRooted()

// emulator detection
val emulatorDetection = raspManager.getEmulatorDetection()
val isDeviceEmulator = raspManager.isDeviceEmulator()

// debugger
val debuggerDetection = raspManager.getDebuggerDetection()
val isDebuggerAttached = raspManager.isDebuggerAttached()

// repackaging
val repackagingResult = raspManager.isAppRepackaged()

// screen sharing
val screenSharingDetection = raspManager.getScreenSharingDetection()
val isScreenShared = raspManager.isScreenShared()

// screen lock usage
val isDeviceUsingScreenLock = raspManager.isDeviceUsingScreenLock()

// Play Protect status
val isPlayProtectEnabled = raspManager.isPlayProtectEnabled()

// tapjacking
val isBadTapjackingCapableAppPresent = raspManager.isBadTapjackingCapableAppPresent()
val tapjackingDetection = raspManager.getTapjackingDetection()

// http proxy detection
val isHttpProxyEnabled = raspManager.isHttpProxyEnabled()
val httpProxyDetection = raspManager.getHttpProxyDetection()

// VPN
val isVpnEnabled = raspManager.isVpnEnabled()

// ADB status
val isAdbEnabled = raspManager.isAdbEnabled()

// developer options status
val isDeveloperOptionsEnabled = raspManager.isDeveloperOptionsEnabled()

// biometry enrollment status
val biometryDetection = raspManager.getBiometryDetection()

// active call
val isCallActive = raspManager.isCallActive()
val activeCallDetection = raspManager.getActiveCallDetection()
```

## Connect to Our Threat Intelligence Cloud

_(optional)_ In case you have access to our online console, you can configure credentials in our SDK so that the mobile app sends signals whenever a security policy violation is detected.

The credential consist of the following values:

- `API_USERNAME` - Username associated with your Android application ID.
- `API_PASSWORD` - API password value.
- `API_SIGNATURE_PUBLIC_KEY` - Public key that the SDK uses to verify authenticity of data received from the server.

You need to adjust your configuration in code the following way:

```java
val config = AppProtectionConfig.Builder(appContext)
    .apiUsername(API_USERNAME)
    .apiPassword(API_PASSWORD)
    .apiSignaturePublicKey(API_SIGNATURE_PUBLIC_KEY)
    .clientAppUserId(INTERNAL_CLIENT_USER_ID) // Use if an internal user ID is available at config time
    .clientAppDeviceId(INTERNAL_CLIENT_DEVICE_ID) // Use if an internal device ID is available at config time
    .antivirusConfig(
        AntivirusConfig.Builder()
                .build()
    )
    .raspConfig(
        RaspConfig.Builder()
            .repackage(
                RepackageDetectionConfig.Builder()
                    .signatureHash(listOf(SIGNATURE_HASH))
                    .build()
            )
            .build()
    )
    .build()

// Initialize AppProtection class
val appProtection = AppProtection.getInstance()
appProtection.initializeAsync(config, object: AppProtection.InitializationObserver {
    // App Protection is fully ready to be used now
    override fun onInitialized(initializationResult: InitializationResult) {
        // Setup internal IDs when you are able to obtain them
        appProtection.updateClientAppUserId(INTERNAL_CLIENT_USER_ID)
        appProtection.updateClientAppDeviceId(INTERNAL_CLIENT_DEVICE_ID)
    }
})
```
After you compile and launch the Android application with the online console access configured, the `AppProtection` class will scan the device and automatically send the impulse from an active device to the online service.

## App Hardening With App Shielding

_(optional)_ As an additional code obfuscation and app hardening step, you can include the additional compile step to your project. This process is referred to as "app shielding". To add the step in your Gradle project, you need to:

1. Copy the provided tooling to some well-defined location on your machine outside your Android project.
1. Add the configuration files to your Android project.
1. In your application’s build.gradle file, add the following plugin:
    ```groovy
    apply from: "${rootProject.rootDir}/wultra/wultra-shielder.gradle"
    ```
1. Enable App Shielding for the product flavor you desire (note that App Shielding only works for the release variants):
    ```
    productFlavors {
        flavorWithShield {
            enableAppShield(owner)
        }
    }
    ```
1. Run an assemble task for the flavor.

## Summary

We just integrated the in-app protection into your Android app and - if you configured the online access - sent the first data sample to the online service. After you launch your app to the public, the console will start to fill up with useful data about active insecure devices.

## Read Next

- [In-app protection for Android](/components/malwarelytics-android/)
