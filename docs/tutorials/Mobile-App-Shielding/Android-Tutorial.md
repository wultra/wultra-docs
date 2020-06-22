# Implementing App Shielding on Android

<!-- AUTHOR joshis_tweets 2020-06-22T00:00:00Z -->
<!-- SIDEBAR _Sidebar_Android.md sticky -->

<div id="banner">
    <div class="alert alert-warning">
        <strong>Personalized Configuration Required.</strong><br/>
        <span>In order to follow this tutorial, you need to purchase the App Shielding technology by Wultra and have a tooling as well as custom configuration prepared by Wultra engineers. Both tooling and configuration is specifically designed for your application. Contact your sales representative or technical consultant in order to obtain the required components.</span>
    </div>
</div>

In this tutorial, we will show you how to implement App Shielding in your Android app. This tutorial has three parts:

- [Mobile App Shielding Overview](./Readme.md)
- [App Shielding for iOS](./iOS-Tutorial.md)
- **App Shielding for Android**

## Prerequisites

- App Shielding tools with the configuration prepared by Wultra.
- Your app in the format of `*.apk`.
- Build machine with Java 8 installation.
- Keystore with the signing certificate.

## Running From the Command-Line

To apply App Shielding to an Android app using command-line, you can simply run the shielder script while providing the properties file:

```sh
sh shield-app.sh config.properties
```

The configuration property file is prepared by Wultra and already contains everything you need to successfully shield the app. However, you need to review the config file and customize it to point to your JKS keystore and use your signing credentials.

## Running From the Android Studio

To run App Shielding on your app project from the Android studio, you can apply our Gradle plugin. To do so, you can:

1. Put the App Shielding tools directory to some well-defined location on your machine outside your Android project.
2. Add the `wultra` directory we provided you with into your Android project.
3. In your application's `build.gradle` file, add the following plugin:
    ```rb
    apply from: "${rootProject.rootDir}/wultra/wultra-shielder.gradle"
    ```
4. Enable App Shielding for the product flavor you desire (note that App Shielding only works for the release variants):
    ```rb
    productFlavors {
        flavorWithShield {
            enableAppShield(owner)
        }
    }
    ```
5. Run an `assemble` task for the flavor.

## Continue Reading

Proceed with one of the following chapters:

- [App Shielding Overview](./Readme.md)
- [App Shielding for iOS](./iOS-Tutorial.md)

## Conclusion

In this tutorial, we showed you how to apply App Shielding to your Android app using the command-line script or via Android Studio (using the Gradle integration).
