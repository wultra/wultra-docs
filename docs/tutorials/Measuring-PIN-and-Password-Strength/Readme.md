# Measuring PIN and Password Strength with Wultra Passphrase Meter

<!-- AUTHOR realKober 2020-05-18T00:00:00Z -->
<!-- SIDEBAR _Sidebar.md sticky -->
<!-- TEMPLATE tutorial -->

This tutorial will show you how to add a password and a PIN strength meter feature to your Android or iOS app using the open-source [Wultra Passphrase Meter](https://github.com/wultra/passphrase-meter#docucheck-keep-link).

Parts of the repository are [iOS](https://github.com/wultra/passphrase-meter/blob/develop/docs/Platform-iOS.md#example-project#docucheck-keep-link) and [Android](https://github.com/wultra/passphrase-meter/blob/develop/docs/Platform-Android.md#example-project#docucheck-keep-link) demo apps that you can try.

## Installation 

{% codetabs %}
{% codetab Android %}

Add following dependency in your `gradle.build` file:

```groovy
repositories {
    jcenter() // if not defined elsewhere...
}

dependencies {
    implementation "com.wultra.android.passphrasemeter:passphrasemeter-core:1.0.0"
    implementation "com.wultra.android.passphrasemeter:passphrasemeter-dictionary-en:1.0.0" // For english speaking people
}
```
{% endcodetab %}
{% codetab iOS %}

iOS integration is done via Cocoapods. Simply add the following lines to your `Podfile`:

```rb
pod 'WultraPassphraseMeter'
pod 'WultraPassphraseMeter/Dictionary_en' # for english dictionary
```

_Note: For more detailed instructions on how to install the library, follow [the installation guide in the project documentation](https://github.com/wultra/passphrase-meter/blob/develop/docs/Platform-iOS.md#installation)._
{% endcodetab %}
{% endcodetabs %}

## Testing Passwords

In order to test the password strength, use the following code:

{% codetabs %}
{% codetab Kotlin %}
```kotlin
import com.wultra.android.passphrasemeter.*
import com.wultra.android.passphrasemeter.exceptions.*

// If your app has an additional dependency on english dictionary,  
// then you need to load that dictionary first.
// assets is property of ContextThemeWrapper
try {
    PasswordTester.getInstance().loadDictionary(assets, "en.dct")
    // Test the password
    val result = PasswordTester.getInstance().testPassword("test")
    print("Strength", result.name)
} catch (e: WrongPasswordException) {
    // Password check failed
}
```
{% endcodetab %}
{% codetab Swift %}
```swift
import WultraPassphraseMeter

// If your app has an additional dependency on english dictionary,  
// then you need to load that dictionary first.
PasswordTester.shared.loadDictionary(.en)
// Test the password
let strength = PasswordTester.shared.testPassword("test")
print(strength)
```
{% endcodetab %}
{% endcodetabs %}

You can evaluate any password. The result of such operation is a strength of the password with the following levels:

- **Very Weak**
- **Weak**
- **Moderate**
- **Good**
- **Strong**

## Testing PIN Codes

The PIN code testing is slightly different from the password testing. The result of a PIN code strength test is a list of findings, that you can use to evaluate the PIN code strength, rather than a strength itself.

Here is an example code of such evaluation:

{% codetabs %}
{% codetab Kotlin %}
```kotlin
import com.wultra.android.passphrasemeter.*
import com.wultra.android.passphrasemeter.exceptions.*

try {
    val passcode = "1456"
    val result = PasswordTester.getInstance().testPin(passcode)
    var isWeak = false

    if (passcode.length <= 4) {
        isWeak = result.contains(PinTestResult.FREQUENTLY_USED) || result.contains(PinTestResult.NOT_UNIQUE)
    } else if (passcode.length <= 6) {
        isWeak = result.contains(PinTestResult.FREQUENTLY_USED) || result.contains(PinTestResult.NOT_UNIQUE) || result.contains(PinTestResult.REPEATING_CHARACTERS)
    } else {
        isWeak = result.contains(PinTestResult.FREQUENTLY_USED) || result.contains(PinTestResult.NOT_UNIQUE) || result.contains(PinTestResult.REPEATING_CHARACTERS) || result.contains(PinTestResult.HAS_PATTERN)
    }

    if (isWeak) {
        print("This PIN is WEAK. Use different one.")
    } else {
        print("PIN OK")
    }
} catch (e: WrongPinException) {
    // PIN format error
}
```
{% endcodetab %}
{% codetab Swift %}
```swift
import WultraPassphraseMeter

let passcode = "1456"
let result = PasswordTester.shared.testPin(passcode)
var isWeak = false

if passcode.count <= 4 {
    isWeak = result.contains(.frequentlyUsed) || result.contains(.notUnique)
} else if passcode.count <= 6 {
    isWeak = result.contains(.frequentlyUsed) || result.contains(.notUnique) || result.contains(.repeatingCharacters)
} else {
    isWeak = result.contains(.frequentlyUsed) || result.contains(.notUnique) || result.contains(.repeatingCharacters) || result.contains(.patternFound)
}

if isWeak {
    print("This PIN is WEAK. Use different one.")
} else {
    print("PIN OK")
}
```
{% endcodetab %}
{% endcodetabs %}

You can evaluate any PIN. The result of the testing is a collection of issues that were found in PIN. This issues can be:

- **Not Unique** - The passcode doesn't have enough unique digits.
- **Repeating Digits** - There is a significant amount of repeating digits in the passcode.
- **Has Pattern** - Well-known pattern was found in the passcode - 1357 for example.
- **Possibly Date** - This passcode can be a date, such as the birthday of the user.
- **Frequently Used** - The passcode is in the list of the most frequently used passcodes.
- **Wrong Input** - Wrong input - the passcode must contain digits only.

## Asynchronous Usage

Password and PIN strength checking might be heavy on the CPU in some cases. To avoid any UI shuttering, we recommend to avoid calling `testPassword` and `testPin` on the main thread.

{% codetabs %}
{% codetab Kotlin %}
```kotlin
import android.support.annotation.WorkerThread

@WorkerThread
private fun processPassword(password: String) {
    // ...
}
```
{% endcodetab %}
{% codetab Swift %}
```swift
import WultraPassphraseMeter

private var queue: OperationQueue =  {
   let q = OperationQueue()
    q.name = "PassMeterQueue"
    q.maxConcurrentOperationCount = 1
    return q
}()

// ...

func processPassword(_ password : String) {
    // if the user types too fast, cancel waiting operations and add new one
    queue.cancelAllOperations()
    queue.addOperation {
        let result = PasswordTester.shared.testPassword(password)
        print(result)
    }
}
```
{% endcodetab %}
{% endcodetabs %}

## Summary

Adding a password or a PIN code strength check to your mobile app is fast, easy, and *free!* with the Wultra Passphrase Meter. You can improve your app security and UX with no more than a few lines of code, and we encourage you to give our library a try.
