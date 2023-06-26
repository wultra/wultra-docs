# Configuring Private Cocoapods Repository
<!-- AUTHOR joshis_tweets 2023-06-26T00:00:00Z -->
<!-- SIDEBAR _Sidebar.md sticky -->
<!-- TEMPLATE tutorial -->

Activation Spawn iOS SDK is distributed through the Wultra Artifactory which provides a private Cocoapods repository. To be able to use this repository, you need to configure your Podfile to be able to retrieve the SDK.

## How to Integrate

1. Install cocoapods-art plugin to be able to access our private repository  

   ```sh
   gem install cocoapods-art
   ```

2. Create (or append to if already exists) `~/.netrc` file in your home directory with the credentials you were provided by Wultra:

   ```
   machine wultra.jfrog.io
         login [name@yourcompany.com]
         password [password]
   ```

3. Synchronize remote repositories locally:

   ```sh
   pod repo-art add device_fingerprint_apple https://wultra.jfrog.io/artifactory/api/pods/device-fingerprint-apple
   pod repo-art add activation_spawn_apple https://wultra.jfrog.io/artifactory/api/pods/activation-spawn-apple
   ```

   <!-- begin box info -->
   To synchronize repositories and receive new versions in the future, use the following commands:

   ```sh
   pod repo-art update device_fingerprint_apple
   pod repo-art update activation_spawn_apple
   ```
   <!-- end -->

4. To enable cocoapods-art plugin in your project `Podfile`, add the following code somewhere at the beginning of the file:

   ```rb
   plugin 'cocoapods-art', :sources => [
       'activation_spawn_apple','device_fingerprint_apple'
   ]
   ```

5. Add pod to your `Podfile`:

   ```rb
   pod 'WultraActivationSpawn'
   pod 'WultraDeviceFingerprint'
   pod 'PowerAuth2' # if not added yet
   ```

6. Run `pod install` in your project dictionary to make the `WultraActivationSpawn` framework available in your project.

## Continue Reading

- [Activaton Spawn on iOS](Activation-Spawn-on-iOS.md#)
