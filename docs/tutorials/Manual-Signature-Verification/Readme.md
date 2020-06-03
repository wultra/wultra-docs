# Verifying PowerAuth Signatures On The Server

<!-- AUTHOR joshis_tweets 2020-06-04T00:00:00Z -->

In this tutorial, we will show you how to verify PowerAuth signatures manually. While the task is relatively simple, it is very sensitive to any minor inconsistencies. Do not get frustrated if signature verification does not work the first time. If you get stuck, do not hesitate to contact our engineers for help.

## Introduction

When implementing mobile authentication and authorization, you need to implement at least two core processes:

- [Activation](https://github.com/wultra/wultra-docs/blob/develop/docs/tutorials/Authentication-in-Mobile-Apps/Readme.md#activation) - mobile device enrollment
- [Transaction signing](https://github.com/wultra/wultra-docs/blob/develop/docs/tutorials/Authentication-in-Mobile-Apps/Readme.md#transaction-signing) - for example, login or payment approval

The activation process can be externalized into a standalone [Enrollment Server](https://github.com/wultra/wultra-docs/blob/develop/docs/tutorials/Authentication-in-Mobile-Apps/Server-Side-Tutorial.md#deploying-the-enrollment-server) application. It can take over the activation process entirely and it can be deployed fully independently from any systems you already have.

The transaction signing process is much more tightly coupled with your protected API resources. As a result, you usually need to integrate the PowerAuth signature verification logic into your existing systems that publish the protected resources. This is a trivial task if you use Spring framework thanks to our magical Spring `@PowerAuth` annotation, as we show in our [tutorial on mobile authentication and transaction signing](https://github.com/wultra/wultra-docs/blob/develop/docs/tutorials/Authentication-in-Mobile-Apps/Server-Side-Tutorial.md#preparing-protected-api-resources).

However, not all systems use Spring, or even Java. What if you use .NET, Java, Ruby, Python, or any other server-side technology? Do not worry - adding the support for signature verification is not difficult.


## Mobile App Perspective

To understand the server-side signature verification, it is helpful to understand how the signatures are computed on the mobile device, using our iOS or Android SDK.

When computing a signature, the mobile SDK requires the following input values:

- **HTTP method**, usually `POST`.
    - For signed requests, we recommend always using the `POST` method.
- **Resource ID**, this a constant that client and server needs to agree on beforehand.
    - The value of this constant is usually (by convention) the relevant part of the URI that gets called. For example, if the URI is `https://api.myserver.com/application-context/1.0/login`, the resource ID would be set to `/login` value.
    - _Note: We do not use the full URL since it could be difficult to deduce the correct path on the server side. Systems may run in different clusters and in different URI contexts (for example, the server could see `https://internal-machine/domain1/application-context/1.0-rel202005/login` instead of the above-mentioned address)._
- **HTTP request data**
    - for `POST` requests (and other requests with the body), this is just HTTP request body
    - for `GET` requests, we use a canonicalized request query parameters

Based on the data, the mobile app produces an HTTP header with the PowerAuth signature, that looks like this (of course, without the masked value):

```
X-PowerAuth-Authorization: PowerAuth pa_version="3.1",
    pa_activation_id="f8******-****-****-****-**********76",
    pa_application_key="Bz******************iQ==",
    pa_nonce="GM******************ZQ==",
    pa_signature_type="possession_knowledge",
    pa_signature="wXay*************************************U="
```

As you can see, the header contains the:

- **Signature version** - version of the used algorithm
- **Activation ID** - identifier of the cryptographic element
- **Application key** - identifier of the application version
- **Nonce** - a random value used when computing the cryptographic signature
- **Signature type** - the type of the signature, denotes factors that were used when computing the signature
- **Signature** - the actual value of the signature

The goal of the server is to:

1. Accept the HTTP request send by the mobile application.
2. Extract all important data from the HTTP request.
3. Transform the values precisely (bit-by-bit) to an appropriate format.
4. Call the PowerAuth Server method that verifies the signature and returns additional results of the verification, such as (and mainly...) the user ID.
5. Follow-up on the signature verification result with own business logic.

## Parsing the Signature Header

The first task that needs to be carried out is parsing the HTTP header with the PowerAuth signature. Get the value of the `X-PowerAuth-Authorization` header.

This is the first thing that you should check: If you expect the header and it is not there, you should stop the processing and return `HTTP 401`.

The PowerAuth signature header has a fixed prefix `PowerAuth `, followed by the comma separated key-value pairs: `key1="value1", key2="value2"`. Note that the values are always in quotes.

In our Java code, we use the `PowerAuthHttpHeader` class ([link](https://github.com/wultra/powerauth-crypto/blob/develop/powerauth-java-http/src/main/java/io/getlime/security/powerauth/http/PowerAuthHttpHeader.java)) to do this work. The important part of the Java code that we use for the header parsing looks like this:

```java
protected static final String POWERAUTH_PREFIX = ;

protected Map<String, String> parseHttpHeader(String header) {
    // Check if the header is null
    if (header == null) {
        return new HashMap<>();
    }

    // Trim the trailing white characters
    header = header.trim();

    // Check if the header has the right prefix
    if (!header.startsWith(POWERAUTH_PREFIX)) {
        return new HashMap<>();
    }

    // Trim the key-value pairs from the header
    header = header.substring(POWERAUTH_PREFIX.length()).trim();

    // Compile the reqexp and parse the key / value pairs
    Map<String, String> result = new HashMap<>();
    Pattern p = Pattern.compile("(\\w+)=\"*((?<=\")[^\"]+(?=\")|([^\\s]+)),*\"*");
    Matcher m = p.matcher(header);
    while (m.find()) {
        result.put(m.group(1), m.group(2));
    }

    // Return the map with key-value pairs that you can fetch later
    return result;
}
```

Then we have a convenience `PowerAuthSignatureHttpHeader` subclass ([link](https://github.com/wultra/powerauth-crypto/blob/develop/powerauth-java-http/src/main/java/io/getlime/security/powerauth/http/PowerAuthSignatureHttpHeader.java)) for the signature header type that only does the more convenient (and typed) value extraction.

## Obtaining the Request Data

In the case the HTTP request uses any other HTTP method than `GET`, this task is easy: Just take the HTTP request body, byte-by-byte (make sure to use UTF-8 encoding for any conversions that you might need).

For `GET` requests, this gets a bit more tricky, since `GET` requests usually do not have any body. While this may not be a problem in some cases, we include query parameters into the request data in case they are present. To achieve precisely the same value of request data on the server and client, we apply a canonization algorithm on the query attributes:

1. Sort the query attribute by name.
2. In the case there are more than one query attribute with the same name, sort the query attributes also by the value.
3. Create a new query string with ordered query params, such as `param1=value1&param2=value2&...`.

We have a special `PowerAuthRequestCanonizationUtils` for this task ([link](https://github.com/wultra/powerauth-crypto/blob/develop/powerauth-java-http/src/main/java/io/getlime/security/powerauth/http/PowerAuthRequestCanonizationUtils.java)). The code is a bit too long and boring for a copy-paste to this tutorial, in case you need to handle signed `GET` requests, have a look into our implementation.

## Building the Signature Base String

Now, when we have the request data and parsed HTTP header, we may proceed to building the signature base string. This is the string value that was actually signed on the mobile device! `#excided #almostthere`

Building the signature base string is simple if we have all the data in place. We use our own `PowerAuthHttpBody` class ([link](https://github.com/wultra/powerauth-crypto/blob/develop/powerauth-java-http/src/main/java/io/getlime/security/powerauth/http/PowerAuthHttpBody.java)) to do most of the logic.

The important Java code is the following:

```java
public static String getSignatureBaseString(String httpMethod, String requestUri, byte[] nonce, byte[] data) {

    String requestUriHash = "";
    if (requestUri != null) {
        byte[] bytes = requestUri.getBytes(StandardCharsets.UTF_8);
        requestUriHash = BaseEncoding.base64().encode(bytes);
    }

    String dataBase64 = "";
    if (data != null) {
        dataBase64 = BaseEncoding.base64().encode(data);
    }

    String nonceBase64 = "";
    if (nonce != null) {
        nonceBase64 = BaseEncoding.base64().encode(nonce);
    }

    return (httpMethod != null ? httpMethod.toUpperCase() : "GET")
            + "&" + requestUriHash
            + "&" + nonceBase64
            + "&" + dataBase64;
}
```

To call the method, you need to know:

- HTTP request method - Comes with the HTTP request.
- The "request URI" value - The request URI ID, a constant that we talked about earlier and that client and server need to share beforehand.
    - **Note: This is not the absolute HTTP URI path, only a constant that usually resembles it, such as `/login`.**
- The "nonce" - Comes in the HTTP header.
    - **Note: The nonce value should be Base64 encoded only once. In our code, we receive nonce in the HTTP header as a Base64 encoded string, we decode it to the bytes, and when building a signature base string, we encode it to Base64 again. While this looks like extra work, at least we can validate the correct format of the value as well as sufficient length of the underlying `byte[]`.**
- The request data - We obtained those in the section above.

At this moment, we should have everything to call the PowerAuth Server API.

## Verifying the Signatures

This is the moment of truth. After your API server receives the signed request from the mobile app, you need to call the PowerAuth Server API to check that the signature is correct.

You can use the following RESTful API for that:

#### POST /rest/v3/signature/verify

```json
{
  "requestObject": {
    "activationId": "string",
    "applicationKey": "string",
    "data": "string",
    "signature": "string",
    "signatureType": "ENUM_SIGNATURE_TYPE",
    "signatureVersion": "string"
  }
}
```

**Request Body Structure**

| Parameter | Description | Obtained at |
|---|---|---|
| `activationId` | ID of the activation. | Comes in the signature HTTP header. |
| `applicationKey` | Key specific for an application version. | Comes in the signature HTTP header. |
| `data` | Signature base string. | We computed this value during the process. |
| `signature` | Signature value. | Comes in the signature HTTP header. |
| `signatureType` | Signature type. | Comes in the signature HTTP header. Note: The PowerAuth Service uses enum values in upper-case, while the HTTP header comes in lower-case format. The values are otherwise the same. |
| `signatureVersion` | Signature version. | Comes in the signature HTTP header. |

_Note: We also have a SOAP service, if you like this better. [Here is the documentation](https://developers.wultra.com/docs/2020.05/powerauth-server/SOAP-Service-Methods), including the link to WSDL definitions._

You will receive the following response:

```json
{
  "status": "string",
  "responseObject": {
    "signatureValid": true,
    "activationId": "string",
    "activationStatus": "ENUM_ACTIVATION_STATUS",
    "userId": "string",
    "applicationId": 0,
    "blockedReason": "string",
    "remainingAttempts": 0,
    "signatureType": "ENUM_SIGNATURE_TYPE"
  }
}
```

**Response Body Structure**

| Parameter | Description |
|---|---|
| `status` | Status of the request processing, `OK` if the HTTP request executes correctly, `ERROR` if an error occurs. |
| `responseObject.signatureValid` | Boolean indicating if signature was verified correctly or if the signature was invalid. |
| `responseObject.activationId` | Activation ID of the activation used to verify the signature. |
| `responseObject.activationStatus` | Status of the activation during the verification attempt (`ACTIVE`, `BLOCKED`, `REMOVED`, unless you made an implementation error, you should not obtain `CREATED` or `PENDING_COMMIT` values.) |
| `responseObject.userId` | User ID of the user who is associated with this activation. |
| `responseObject.applicationId` | Application ID of an application that was used to compute the signature. |
| `responseObject.blockedReason` | In case the activation is blocked, this value provides more information about the cause. It can be any value provided while calling the service that blocks the activation, or a `MAX_FAILED_ATTEMPTS` value in case the activation was blocked due to too many authentication failed attempts. |
| `responseObject.remainingAttempts` | Number of the remaining authentication attempts. |
| `responseObject.signatureType` | Used signature type. |

The first thing you must do is to check the `responseObject.signatureValid` value. If false, the signature verification failed and you should return `HTTP 401` in the response to the mobile client.

If the signature verification was successful, you can now work with the remaining response values in any way your business logic requires.

For example, you can:

- Obtain the user ID to create authenticated session or issue an access token in case of a login endpoint, or submit a payment while checking the user is allowed to do so in the case of payment endpoint.
- Check if the used signature type can be used for the given operation (for example, to disable biometry for payments above some value).
- Check that the application is allowed to perform the operation (for example, your mobile banking app may be able to call all endpoints, while a mobile brokerage app could be restricted just to some endpoints).

## Summary

We successfully guided you through the signature validation process, step by step. You should now be able to implement signature validation in any system you currently have. While the process is not not as straight-forward in the case of Spring framework, it is still manageable, and of course, we are always here to help.
