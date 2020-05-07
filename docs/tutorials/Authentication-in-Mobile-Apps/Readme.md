# Authentication in Mobile Banking Apps (SCA)

<!-- AUTHOR joshis_tweets 2020-05-04T00:00:00Z -->

In this tutorial, we will show you how you can implement authentication into your mobile banking or fintech apps. The tutorial has four parts:

- Overview
- [Tutorial for Server Side Developers](./Server-Side-Tutorial.md)
- [Tutorial for iOS Developers](./iOS-Tutorial.md)
- [Tutorial for Android Developers](./Android-Tutorial.md)

## Overview

Authentication is a process of verifying the user's identity, usually performed in order to manage access to an application or in order to approve a transaction, such as payment, using a reliable user identity proof. In case of the banking systems, managing access to systems and transaction signing are two critical requirements from both security and compliance standpoint.

In terms of the security, well-designed authentication and operation approval improves the confidentiality of the information, since only the correct users can access the particular data set. At the same time, it protects from manipulating the financial transactions, since each payment has a strong cryptographic proof of the user who approved it.

In terms of the compliance, these mechanisms are required for example to comply with the European PSD2 legislation and its requirements on Strong Customer Authentication (SCA).

[Mobile Security Suite](https://www.wultra.com/mobile-security-suite) is a simple plugin (SDK) for iOS and Android apps that - together with its server counterparts - covers the authentication functionality, allowing the end-users to login or sign payments using simple PIN code or a biometry. It also covers all related use-cases, such as authentication element lifecycle (active, blocked, removed), PIN code change, biometry settings, etc.


## Activation

When a user downloads a mobile app, be it for iOS or Android, this app is always blank, non-personalized app. Basically the same app for all users. In order to connect the app with the user account, the user needs to "activate" the application first. This requires an interplay between the mobile app and the server components, as well as a user providing an identity proof (some credentials) that could be used to identify the user reliably. This initial process is sometimes called registration, enrollment, personalization. Our term for this process is **"Activation"**, though.

The activation can be either performed fully on the mobile device (it is initiated and completed on the mobile device) or in an "out-of-band" mode (initiated externally, for example, in the Internet banking, and completed on the mobile device and in the external systems).

### On-Device Activation

The high-level overview of the on-device activation steps is captured in this diagram:

![ Activation - Fully on the mobile device ](./01a.png)

As you can see, this diagram is relatively simple and straight-forward.

You have the following actors:

- **The end-user** who downloads an app from App Store or Google Play.
- **The mobile app** with our SDK that sends cryptographic activation payload, alongside with a user's identity proof, to the mobile enrollment server, and stores the resulting transaction signing keys later in the process.
- **The mobile enrollment server** that is responsible for orchestrating the activation process by first verifying the identity proof against some existing system, and then creating a new activation record in the mobile management server.
- **An external system for verifying the credentials** - this is usually a customer specific component, such as CRM, IDM, etc. and connecting to this component requires customization.
- **Mobile management server**, the main component that stores the enrolled mobile devices and their cryptographic material, attached to the user via the user ID value.

### Out-of-Band Activation

The high-level overview of the out-of-band activation steps is captured in this diagram:

![ Activation - In out-of-band mode ](./01b.png)

While this diagram seems a bit more complex, the process is still very simple. The main difference is in the initiation of the process, where some external system, in our case the Internet banking, must first fetch the activation code (usually represented by a QR code, link with the parameter, or a text value). Later in the process, the Internet banking must confirm the activation to make it useable.

You have the following actors:

- **The end-user** who downloads an app from App Store or Google Play.
- **An internet banking application** (client and server side) where the user can sign in, initiate a new activation and after the mobile exchange is complete, confirm the activation.
- **The mobile app** with our SDK that sends cryptographic activation payload, alongside with a user's identity proof, to the mobile enrollment server, and stores the resulting transaction signing keys later in the process.
- **The mobile enrollment server** responsible for orchestrating the activation process by first verifying the identity proof and then creating a new activation record in the mobile management server.
- **Mobile management server** - a component that stores the enrolled mobile devices and their cryptographic material.

Unlike in the on-device activation, no external system for verifying the credentials is needed in this case, since the user is already signed in to the Internet banking, and the activation code is generated for that particular user.

You can look at the activation code in the case of the out-of-band mode is as at a special one-time credentials for a given user that are managed in the mobile device enrollment server.

**Note: In both cases, the mobile app only communicates with the mobile enrollment server, never with mobile management server. Only back-end applications communicate to the mobile management server.**

## Transaction Signing

After the user activates the mobile app, it is ready for it's main use case: **"Transaction signing"**.

The transaction signing is technically used for all types of approvals - login, payments, configuration changes, atd. In all cases, the user needs to enter a PIN code set during the activation process, or use biometry on the mobile device to compute the data signature. This signature is later verified on the server.

In case of the login, signature is computed just based on the user credentials. In the case of the payment or other similar operation that generally has some data assigned to it, the operation data also projects into the resulting signature. This dependency of a signature on the operation data is sometimes referred to as "dynamic linking".

_Note: The PSD2 legislation mandates using at least data based on the other party account number and payment amount for the purpose of the dynamic linking._

The overview of the transaction signing is fairly simple:

![ Transaction Signing ](./02.png)

As you can see, there are the following actors:

- User
- Mobile application
- API Server
- Mobile management server

## Other Use-Cases

Of course, the system already implements some typical use-cases that are usually requested by the customers. In this overview, we will just briefly mention the list of such use-cases:

- Activation status check.
- Activation blocking and removal.
- Password / PIN code change.
- Biometric authentication setup.
- End-to-end payload encryption.

Please refer to the detailed documentation for more information about those.

## Resources

Of course, you can always find more details in our reference documentation:

- [Description of Cryptographic Processes](./TBD)
- [PowerAuth Server Documentation](./TBD)
- [Server Integration Libraries Documentation](./TBD)
- [Mobile SDK Documentation](./TBD)
