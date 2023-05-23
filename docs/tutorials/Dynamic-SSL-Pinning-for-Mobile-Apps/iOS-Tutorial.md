# Dynamic SSL pinning for iOS

<!-- AUTHOR joshis_tweets 2023-05-23T00:00:00Z -->
<!-- SIDEBAR _Sidebar_iOS.md sticky -->
<!-- TEMPLATE tutorial -->

The `WultraSSLPinning` library manages the dynamic list of certificates downloaded from the remote server and provides easy-to-use fingerprint validation on the TLS handshake.

The library provides the following core types:

- `CertStore` - the main class which provides all tasks for dynamic pinning  
- `CertStoreConfiguration` - the configuration structure for `CertStore` class

This document will explain how to install and configure the library and use `CertStore` class for the SSL pinning purposes.


## Installation

### Requirements

- iOS 11.0+
- tvOS 11.0+
- Xcode 14+
- Swift 5.0+

### Swift Package Manager

The [Swift Package Manager](https://swift.org/package-manager) is a tool for automating the distribution of Swift code and is integrated into the `swift` compiler. 

Once you have your Swift package set up, adding this lbirary as a dependency is as easy as adding it to the `dependencies` value of your `Package.swift`.

```swift
dependencies: [
    .package(url: "https://github.com/wultra/ssl-pinning-ios.git", .upToNextMajor(from: "1.5.0"))
]
```

### CocoaPods

[CocoaPods](https://cocoapods.org) is a dependency manager for Cocoa projects. You can install it with the following command:

```bash
$ gem install cocoapods
```

To integrate framework into your Xcode project using CocoaPods, specify it in your `Podfile`:

```ruby
platform :ios, '11.0'
target '<Your Target App>' do
  pod 'WultraSSLPinning/PowerAuthIntegration'
end
```

The current version of the library depends on [PowerAuth2](https://github.com/wultra/powerauth-mobile-sdk) framework, version `0.19.1` and greater.


## Configuration

The following code will configure `CertStore` object with basic configuration (with `PowerAuth2` as cryptographic provider and secure storage provider):

```swift
import WultraSSLPinning

let configuration = CertStoreConfiguration(
    serviceUrl: URL(string: "https://...")!,
    publicKey: "BMne....kdh2ak=",
    useChallenge: true
)
let certStore = CertStore.powerAuthCertStore(configuration: configuration)
```

<!-- begin box info -->
We'll use `certStore` variable in the rest of the documentation as a reference to already configured `CertStore` instance.
<!-- end -->


## Update Fingerprints

To update list of fingerprints from the remote server (ideally, during the app start), use the following code:

```swift
certStore.update { (result, error) in
   if result == .ok {
       // everything's OK, 
       // No action is required, or silent update was started
   } else if result == .storeIsEmpty {
       // Update succeeded, but it looks like the remote list contains
       // already expired fingerprints. The certStore will probably not be able
       // to validate the fingerprints.
   } else {
       // Other error. See `CertStore.UpdateResult` for details.
       // The "error" variable is set in case of a network error.
   }
}
```

## Fingerprint Validation

The `CertStore` provides several methods for certificate fingerprint validation. The easiest in case of iOS is to use the one that accepts `URLAuthenticationChallenge` instance:

```swift
let validationResult = certStore.validate(challenge: challenge)
```

The `validate` method returns `CertStore.ValidationResult` enumeration with the following results:

- `trusted` - the server certificate is trusted. You can continue with the communication

  The right response on this situation is to continue with the ongoing TLS handshake (e.g. report
  [.performDefaultHandling](https://developer.apple.com/documentation/foundation/urlsession/authchallengedisposition)
  to the completion callback)
   
- `untrusted` - the server certificate is not trusted. You should cancel the ongoing challenge.

  The untrusted result means that `CertStore` has some fingerprints stored in its
  database, but none matches the value you requested for validation. The right
  response on this situation is always to cancel the ongoing TLS handshake (e.g. report
  [.cancelAuthenticationChallenge](https://developer.apple.com/documentation/foundation/urlsession/authchallengedisposition)
  to the completion callback)

- `empty` - the fingerprints database is empty, or there's no fingerprint for the validated common name.

  The "empty" validation result typically means that the `CertStore` should update
  the list of certificates immediately. Before you do this, you should check whether
  the requested common name is what's you're expecting. To simplify this step, you can set 
  the list of expected common names in the `CertStoreConfiguration` and treat all others as untrusted.
    
  For all situations, the right response on this situation is always to cancel the ongoing
  TLS handshake (e.g. report [.cancelAuthenticationChallenge](https://developer.apple.com/documentation/foundation/urlsession/authchallengedisposition)
  to the completion callback)


The full challenge handling in your app may look like this:

```swift
class YourUrlSessionDelegate: NSObject, URLSessionDelegate {
    
    let certStore: CertStore
    
    init(certStore: CertStore) {
        self.certStore = certStore
    }
    
    func urlSession(_ session: URLSession, didReceive challenge: URLAuthenticationChallenge, completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void) {
        switch certStore.validate(challenge: challenge) {
        case .trusted:
            // Accept challenge with a default handling
            completionHandler(.performDefaultHandling, nil)
        case .untrusted, .empty:
            /// Reject challenge
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }
}
```

## Resources

You can find more details in the following documentation:

- [Dynamic SSL Pinning SDK for iOS](https://github.com/wultra/ssl-pinning-ios)
- [Dynamic SSL Pinning SDK for Android](https://github.com/wultra/ssl-pinning-android)
- [Mobile Utility Server](https://github.com/wultra/mobile-utility-server)


## Continue Reading

Proceed with one of the following chapters:

- [Dynamic SSL Pinning Overview](./Readme.md)
- [Dynamic SSL Pinning on Android](./Android-Tutorial.md)
- [Dynemic SSL Pinning Server](./Server-Side-Tutorial.md)