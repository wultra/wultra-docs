# Implementing App Shielding on iOS

<!-- AUTHOR joshis_tweets 2020-06-22T00:00:00Z -->
<!-- SIDEBAR _Sidebar_iOS.md sticky -->
<!-- TEMPLATE tutorial -->

<div id="banner">
    <div class="alert alert-warning">
        <strong>Personalized Configuration Required.</strong><br/>
        <span>In order to follow this tutorial, you need to purchase the App Shielding technology by Wultra and have a tooling as well as custom configuration prepared by Wultra engineers. Both tooling and configuration is specifically designed for your application. Contact your sales representative or technical consultant in order to obtain the required components.</span>
    </div>
</div>

In this tutorial, we will show you how to implement App Shielding in your iOS app. This tutorial has three parts:

- [Mobile App Shielding Overview](./Readme.md)
- **App Shielding for iOS**
- [App Shielding for Android](./Android-Tutorial.md)

## Prerequisites

- App Shielding tools with the configuration prepared by Wultra.
- Your app in the format of `*.app`, `*.ipa` or `*.xcarchive`.
- MacOS machine with Java 8 installation.
- Signer certificate.

## Running From the Command-Line

The easiest way to get familiar with our App Shielding script is to use the interactive mode:

```sh
./shield.sh --interactive
```

However, this is not practical for most of the situations, such as automated app build using a CI tools.

### Shielding For App Store or Testflight

To shield the production app on the same machine as you used to build and sign an app, you can simply run the shielder script:

```sh
./shield.sh path_to_my_app.xcarchive
```

### Shielding for Developer Distribution

To build an app for the testing purposes, for example, to be submitted to App Center or any other app distribution tool, you need to use the `--trustsigner` switch:

```sh
./shield.sh path_to_my_app.xcarchive --trustsigner
```

### Omitting the App Signature

In case you would like to skip the signing step, you can use the `--noresign` switch and sign the app with the correct certificate later.

### Specifying Paths

You can specify additional paths when running the `shield.sh` script:

- `-shielder` - Path to the shielder utility file.
- `-config` - Path to the App Shielding configuration properties.
- `-framework` - Path to the App Shielding SDK.
- `-output` - Path for the resulting output file.

## Running From the Xcode

The integration with Xcode is based on the command-line integration we saw earlier in this tutorial. To add the App Shielding step in your project, you need to:

1. Copy the `shielder` directory into your Xcode project folder (the `$PROJECT_DIR` variable in Xcode).
2. Set `FRAMEWORK_SEARCH_PATHS=$(inherited) $(PROJECT_DIR)/shielder/`
3. Add a "New Run Script Phase" to your target build phases, with the following script as a content (depending on the intent):

```sh
# Development Scheme
/bin/bash "${PROJECT_DIR}/shielder/shield.sh" "--trustsigner"
```

```sh
# App Store Distribution Scheme
/bin/bash "${PROJECT_DIR}/shielder/shield.sh"
```

### More Complex Xcode Scripting

Of course, you can prepare a more complex script to determine which properties should be applied in the particular build setup based on your own variables (defined in the user defined setting, under "Build Settings"). For example, this is a simplified script that we use in our Mobile Token app on iOS:

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

## Continue Reading

Proceed with one of the following chapters:

- [App Shielding Overview](./Readme.md)
- [App Shielding for Android](./Android-Tutorial.md)

## Conclusion

In this tutorial, we showed you how to apply App Shielding to your iOS app using the command-line or via Xcode integration, including a practical example of our custom script we use on the Mobile Token project.
