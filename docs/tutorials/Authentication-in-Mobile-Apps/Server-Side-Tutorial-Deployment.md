# Mobile-First Authentication&#58; Installing Server-Side Components

<!-- AUTHOR joshis_tweets 2023-12-29T00:00:00Z -->
<!-- SIDEBAR _Sidebar_Server.md sticky -->
<!-- TEMPLATE tutorial -->

In this tutorial, we will show you how to install back-end components for mobile-first authentication in banking or fintech apps.

## Prerequisites

This tutorial assumes that you have:

- Read and understood the [Problem Overview](Readme.md) chapter
- [Docker](https://www.docker.com/) installation (`amd64` or `arm64` architectures)
- [PostgreSQL 12](https://www.postgresql.org/) installation

## Task Overview

When installing the authentication back-end, you need to perform several tasks:

- Pull the Docker image
- Configure the environment
- Run the Docker image
- Setup the first system users
- Create application and assign the permissions
- Configuring push notifications

## Pull the Docker Image

<!-- begin box warning -->
To access our Docker container repository, you need to have an access granted from our administrators. [Contact us](https://wultra.com/contact) to obtain the access.
<!-- end -->

You can obtain the Docker images by simply pulling them from our Artifactory:

```sh
docker login wultra.jfrog.io
docker pull wultra.jfrog.io/wultra-docker/powerauth-cloud:${VERSION}
```

Make sure to replace the `${VERSION}` placeholder with the last available version.

## Configure the Environment

<!-- begin box warning -->
Our Docker images automatically manage the database schema. As a result, the database user must have permissions to manage the schema in the database. For information about our database schema, please refer to [our documentation](/components/powerauth-cloud/develop/documentation/Database-Structure).
<!-- end -->

<!-- begin box warning -->
<strong>Set The Right Database URL</strong><br/>The datasource URL for our Docker container follows the structure of the JDBC connectivity. Make sure to provide a valid JDBC URL to the configuration (starting with `jdbc:` prefix). Be especially careful when working on localhost! From the Docker container perspective, `localhost` is in the internal network. To connect to your host’s localhost, use `host.docker.internal` host name.
<!-- end -->

In order to be able to run the Docker image, you need to set it environment variables that define the database scheme connectivity. You can do so, for example, by creating the `env.list` file with the following content:

```
POWERAUTH_SERVER_DATASOURCE_URL=jdbc:postgresql://host.docker.internal:5432/powerauth
POWERAUTH_SERVER_DATASOURCE_USERNAME=powerauth
POWERAUTH_SERVER_DATASOURCE_PASSWORD=$PASSWORD$

PUSH_SERVER_DATASOURCE_URL=jdbc:postgresql://host.docker.internal:5432/powerauth
PUSH_SERVER_DATASOURCE_USERNAME=powerauth
PUSH_SERVER_DATASOURCE_PASSWORD=$PASSWORD$

POWERAUTH_CLOUD_DATASOURCE_URL=jdbc:postgresql://host.docker.internal:5432/powerauth
POWERAUTH_CLOUD_DATASOURCE_USERNAME=powerauth
POWERAUTH_CLOUD_DATASOURCE_PASSWORD=$PASSWORD$

ENROLLMENT_SERVER_DATASOURCE_URL=jdbc:postgresql://host.docker.internal:5432/powerauth
ENROLLMENT_SERVER_DATASOURCE_USERNAME=powerauth
ENROLLMENT_SERVER_DATASOURCE_PASSWORD=$PASSWORD$
```

## Run the Docker Image

You can now run the Docker image using any techniques typically used for this task. For example, you can run it in Kubernetes or via Docker compose. As a simple quick start, you can just use `docker run` command, like so:

```sh
docker run --env-file env.list -d -it -p 8080:8000 \
    --name=powerauth-cloud wultra.jfrog.io/wultra-docker/powerauth-cloud:${VERSION}
```

## Setup the First System Users

**System users** represent systems, such as back-end applications, that are calling the protected API resources in our back-end components. **Admin system users** can call resources intended for the system administration (available on `/admin/**` context), while **integration system users** can only call resources intended for integration.

The first admin system user has to be created directly in the database. You need to create a `bcrypt` hash of a password to insert the right record using the following SQL commands:

```sql
INSERT INTO pa_cloud_user (id, username, password, enabled)
VALUES (nextval('pa_cloud_user_seq'), 'system-admin', '${BCRYPT_PASSWORD_HASH}', true);

INSERT INTO pa_cloud_user_authority (id, user_id, authority)
VALUES (nextval('pa_cloud_user_seq'), (SELECT id FROM pa_cloud_user WHERE username = 'system-admin'), 'ROLE_ADMIN');
```

Creating an integration system user is easier, you just need to call the API endpoint using the admin system user credentials:

```sh
curl -X 'POST' \
  'http://localhost:8080/powerauth-cloud/admin/users' \
  -u system-admin:${PASSWORD} \
  -H 'Content-Type: application/json' \
  -d '{
  "username": "integration-user"
}'
```

<!-- begin box success -->
The API automatically generates a strong password for the integration user.
<!-- end -->

## Create an Application and Assign Permissions

**Application** in the back-end system represents a particular mobile application. You should create a new application for any mobile app, but **not** a separate one for each platform (iOS/Android). Application can represent, for example, your retail mobile banking, corporate mobile banking, mobile token, investment app, etc.

To create an application, call our API using the admin system user credentials:

```sh
curl -X 'POST' \
  'http://localhost:8080/powerauth-cloud/admin/applications' \
  -u system-admin:${PASSWORD} \
  -H 'Content-Type: application/json' \
  -d '{
  "id": "my-application"
}'
```

<!-- begin box info -->
Pass the response of the call above to your mobile application developers - the values from the response must be embedded in the mobile app configuration.
<!-- end -->

In order for an integration system user to be able to access resources related to the newly created application, you must assign the integration system user permissions to the application by calling the following API endpoint:

```sh
curl -X 'POST' \
  'http://localhost:8080/powerauth-cloud/admin/users/integration-user/applications/my-application' \
  -u system-admin:${PASSWORD} \
  -d ''
```

## Configuring Push Notifications

Our system bundles a basic push server. You can use it to deliver push notifications to Apple or Android devices. We also use it under the hood for various use-cases, such as out-of-band operation approval, or lifecycle management (silent push whenever the device is blocked or removed).

To configure push notification delivery for the application created above, you need to post credentials specific for APNS and FCM services (using your admin system user). You can obtain those credentials at the [Apple Developer Portal](https://developer.apple.com/) or in [Firebase Console](https://firebase.google.com/). Then, you can post them to the application using the following commands specific for each platform.

### Apple Push Notification Service (APNS)

<!-- begin box info -->
Note that Apple provides development or production environment. You should use a specific one based on how you sign your applications. Apps published on App Store and Testflight use production environment, those signed during development and published via services such as Microsoft's App Center typically use development environment.
<!-- end -->

```sh
curl -X 'POST' \
  'http://localhost:8080/powerauth-cloud/admin/applications/my-application/push/apns' \
  -u system-admin:${PASSWORD} \
  -H 'Content-Type: application/json' \
  -d '{
    "topic": "${IOS_TOPIC}",
    "keyId": "${IOS_KEY_ID}",
    "teamId": "${IOS_TEAM_ID}",
    "environment": "development|production",
    "privateKeyBase64": "${IOS_PRIVATE_KEY_BASE64}"
}'
```

### Firebase Cloud Messaging (FCM)

```sh
curl -X 'POST' \
  'http://localhost:8080/powerauth-cloud/admin/applications/my-application/push/fcm' \
  -u system-admin:${PASSWORD} \
  -H 'Content-Type: application/json' \
  -d '{
    "projectId": "${ANDROID_PROCECT_ID}",
    "privateKeyBase64": "ANDROID_PRIVATE_KEY_BASE64"
}'
```

## Continue Reading

You can proceed with one of the following chapters:

- [Problem Overview](Readme.md)
- [Installing Server-Side Components](Server-Side-Tutorial-Deployment.md)
- [Integrating with Your Back-End Applications](Server-Side-Tutorial-Integration.md)
- [Implementing Mobile-First Authentication on iOS](iOS-Tutorial.md)
- [Implementing Mobile-First Authentication on Android](Android-Tutorial.md)

## Resources

You can find more details our reference documentation:

- [Cryptography Specification](/components/powerauth-crypto)
- [PowerAuth Cloud](/components/powerauth-cloud)
- [Mobile Authentication SDK for iOS and Android](/components/powerauth-mobile-sdk)
- [Mobile Token SDK for iOS](/components/mtoken-sdk-ios)
- [Mobile Token SDK for Android](/components/mtoken-sdk-android)
