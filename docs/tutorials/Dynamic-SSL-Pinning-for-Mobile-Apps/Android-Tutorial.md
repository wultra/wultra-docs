# Dynamic SSL pinning for iOS

<!-- AUTHOR joshis_tweets 2023-05-23T00:00:00Z -->
<!-- SIDEBAR _Sidebar_Android.md sticky -->
<!-- TEMPLATE tutorial -->

The `WultraSSLPinning` library manages the dynamic list of certificates downloaded from the remote server and provides easy-to-use fingerprint validation on the TLS handshake.

The library provides the following core types:

- `CertStore` - the main class which provides all tasks for dynamic pinning  
- `CertStoreConfiguration` - the configuration structure for `CertStore` class

This document will explain how to install and configure the library and use `CertStore` class for the SSL pinning purposes.


## Installation

### Requirements

- `minSdkVersion 16` (Android 4.1 Jelly Bean)

### Gradle

To use the library, add this dependency in your Android app:

```gradle
implementation 'com.wultra.android.sslpinning:wultra-ssl-pinning:1.x.y'
```

<!-- begin box info -->
Note that this documentation is using version `1.x.y` as an example. You can find the latest version at [Github's release](https://github.com/wultra/ssl-pinning-android/releases#docucheck-keep-link) page.
<!-- end -->


## Configuration

The following code will configure `CertStore` object with basic configuration:

```kotlin
val publicKey: ByteArray = Base64.decode("BMne....kdh2ak=", Base64.NO_WRAP)
val configuration = CertStoreConfiguration.Builder(serviceUrl = URL("https://..."), publicKey = publicKey)
    .useChallenge(true)
    .build()
val certStore = CertStore.powerAuthCertStore(configuration = configuration, appContext)
```

<!-- begin box info -->
We'll use `certStore` variable in the rest of the documentation as a reference to already configured `CertStore` instance.
<!-- end -->


## Update Fingerprints

To update list of fingerprints from the remote server (ideally, during the app start), use the following code:

```kotlin
certStore.update(UpdateMode.DEFAULT, object: DefaultUpdateObserver() {
    override fun continueExecution() {
        // Certstore is likely up-to-date, you can resume execution of your code.
    }

    override fun handleFailedUpdate(type: UpdateType, result: UpdateResult) {
        // There was an error during the update, present an error to the user.
    }

})
```


## Fingerprint Validation

While the `CertStore` provides several methods for direct certificate fingerprint validation, the easiest in case of Android is to use one of the pre-built integrations.

### Integration With `OkHttp`

To integrate with OkHttp, use the following code:
 
```kotlin
val sslSocketFactory = SSLPinningIntegration.createSSLPinningSocketFactory(certStore);
val trustManager = SSLPinningX509TrustManager(certStore)

val okhttpClient = OkHttpClient.Builder()
                .sslSocketFactory(sslSocketFactory, trustManager)
                .build()
```

In the code above, use `SSLSocketFactory` provided by `SSLPinningIntegration.createSSLPinningSocketFactory(...)` and an instance of `SSLPinningX509TrustManager`.

### Integration With `HttpsUrlConnection`

For integration with `HttpsUrlConnection` use `SSLSocketFactory` provided by `SSLPinningIntegration.createSSLPinningSocketFactory(...)` methods. The code should look like this:

```kotlin
val url = URL(...)
val connection = url.openConnection() as HttpsURLConnection
connection.sslSocketFactory = SSLPinningIntegration.createSSLPinningSocketFactory(certStore)
connection.connect()
```


## Resources

You can find more details in the following documentation:

- [Dynamic SSL Pinning SDK for iOS](https://github.com/wultra/ssl-pinning-ios)
- [Dynamic SSL Pinning SDK for Android](https://github.com/wultra/ssl-pinning-android)
- [Mobile Utility Server](https://github.com/wultra/mobile-utility-server)


## Continue Reading

Proceed with one of the following chapters:

- [Dynamic SSL Pinning Overview](./Readme.md)
- [Dynamic SSL Pinning on iOS](./iOS-Tutorial.md)
- [Dynemic SSL Pinning Server](./Server-Side-Tutorial.md)