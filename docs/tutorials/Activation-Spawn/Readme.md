# Activation Spawn SDK

The purpose of the activation spawn process is to enable creating a new activation for an secondary app using another, main app. From the customer perspective, this can be communicated, for example, through a message: "Activate mobile banking using our mobile token app".

The process has two actors:

- **Main app** - The app used for activation of other apps. The app fetches the activation code from a secure endpoint (providing activation OTP in request) and passes the encrypted value (protected using a materialized white-box crypto key) via the URL scheme to the secondary app.
- **Secondary app** - The app that needs to be activated. It receives the encrypted activation code and activation OTP via the URL scheme and uses this to launch a regular app activation.

## Continue Reading

[Activaton Spawn on iOS](Activation-Spawn-on-iOS.md#)
[Activaton Spawn on Android](Activation-Spawn-on-Android.md#)
[Activation Spawn API Reference](Activation-Spawn-API-Reference.md)
