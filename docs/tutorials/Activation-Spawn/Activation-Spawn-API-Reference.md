# Server REST API
<!-- TEMPLATE api -->

The activation spawn process assumes a new service that the main application uses to fetch the activation code (to be passed to the secondary app).

The principles of the activation spawn are covered in the following tutorial:

- [Introducing the Activation Spawn](Readme.md#)
- [Activaton Spawn on iOS](Activation-Spawn-on-iOS.md#)
- [Activaton Spawn on Android](Activation-Spawn-on-Android.md#)

## API Reference

<!-- begin api POST /api/activation/code -->
### Get The Activation Code

Fetches the activation code from the server. The service requires 2FA PowerAuth signature (`POSSESSION_KNOWLEDGE` or `POSSESSION_BIOMETRY`) and it uses activation scope payload encryption to protect the server call request and response from being intercepted.

<!-- begin box info -->
This endpoint is published by the [Enrollment Server](https://github.com/wultra/enrollment-server) component, where it must be explicitly enabled.
<!-- end -->


#### Request

Decrypted request payload:

```json
{
  "requestObject": {
    "applicationId": "SECONDARY_APP_ID",
    "otp": "1234567890123456"
  }
}
```

| Attribute       | Description                                  |
|-----------------|----------------------------------------------|
| `applicationId` | Identifier of the secondary to be activated. |
| `otp`           | OTP value, at least 16 characters.           |

#### Response 200

Decrypted response payload:

```json
{
  "status": "OK",
  "responseObject": {
    "activationId": "c7a69c2d-...-...-...-604efc71bc82",
    "activationCode": "11111-...-...-11111",
    "activationSignature": "MII...=="
  }
}
```

| Attribute            | Description                                  |
|----------------------|----------------------------------------------|
| `activationId`       | Identifier of new activation.                |
| `activationCode`     | Activation code value.                       |
| `activationSignature`| Activation code signature.                   |
<!-- end -->
