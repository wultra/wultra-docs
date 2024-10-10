# Biometric Liveness Verification

<!-- AUTHOR romanstrobl 2024-09-26T00:00:00Z -->
<!-- SIDEBAR _Sidebar.md sticky -->
<!-- TEMPLATE tutorial -->

This tutorial explains how Wultra's Biometric Liveness Verification components can be deployed, configured and integrated into your application. This solution enables to verify users using advanced biometric verification techniques against their static photos, such as passport photos.

## Introduction

Wultra's Biometric Liveness Verification solution consists of two components which require to be deployed. The components are available as Docker images in Wultra JFrog Artifactory.

### User Data Store

The component can be used as a generic secure storage for any document type. When used within the Biometric Liveness Verification solution, User Data Store is used to store securely user photos. You will not need to integrate with this component, it's used within the Liveness Check Proxy component as a secure storage. However, the component will be used to upload or import user profile photos which serve a basis for the liveness verification. 

### Liveness Check Proxy

The component orchestrates the biometric liveness check and provides a simple REST API for integration with your application. The Liveness Check Proxy uses the User Data Store to retrieve user photos when performing the biometric verification. Furthermore, the component provides an audit trail of performed liveness checks.

## Deploying Backend Components

Both components can be easily deployed using Docker. Use the following commands to download the Docker images:

User Data Store:
```shell
docker pull wultra.jfrog.io/wultra-docker/user-data-store:1.3.0-a2871e2f8d033dd4d00bf1a39456fc503be79194
```

```shell
docker pull wultra.jfrog.io/wultra-docker/liveness-check-proxy:0.1.0-SNAPSHOT-2024.06.26-0f5031f9d5ca260aa40c28305ddb2e93a7f4e74e
```

TODO: use the release version of Docker image

You can use one of the supported databases:
- PostgreSQL 13 or newer
- Oracle Database 19c, or 21c
- MSSQL 2019 or newer

Both components manage the database schemas automatically using Liquibase. So you only need to create a database and set up username and password with privileges to create tables, sequences and indexes within this database.

Detailed documentation for each of the components is available at:

https://developers.wultra.com/components/user-data-store/1.3.x/documentation/
https://developers.wultra.com/components/liveness-check-proxy/1.0.x/documentation/

## Configuring Backend Components

Let's configure each of the components.

### User Data Store Configuration

#### Creating the Database

At first, you will need to set up the database for User Data Store. You can use this example for PostgreSQL, for other databases, see their documentation.

Run as database superuser:
```sql
CREATE DATABASE uds;
CREATE USER uds WITH PASSWORD 'uds';
GRANT ALL PRIVILEGES ON DATABASE uds TO uds;
```

Connect to the created `uds` database:
```shell
\c uds
```

Grant privileges on the `public` schema within this database:
```sql
GRANT USAGE ON SCHEMA public TO uds;
GRANT CREATE ON SCHEMA public TO uds;
```

#### Docker Configuration

Following environmental variables need to be configured for User Data Store:
- `USER_DATA_STORE_DATASOURCE_URL` — database JDBC URL, such as `jdbc:postgresql://host.docker.internal:5432/uds`
- `USER_DATA_STORE_DATASOURCE_USERNAME`  – database username you specified before
- `USER_DATA_STORE_DATASOURCE_PASSWORD` – database password you specified before
- `USER_DATA_STORE_MASTER_ENCRYPTION_KEY` – optional AES-256 key in Base-64 format, specify the key if photos stored in the database should be encrypted

In order to generate the master encryption key, you can use the following command:
```shell
openssl rand -base64 32
```

To specify the environmental variables for Docker, you can use an `.env-uds` file with the following content:
```bash
USER_DATA_STORE_DATASOURCE_URL=your-db-url
USER_DATA_STORE_DATASOURCE_USERNAME=your-db-username
USER_DATA_STORE_DATASOURCE_PASSWORD=your-db-password
USER_DATA_STORE_MASTER_ENCRYPTION_KEY=your-encryption-key
```

To start the Docker container, use:

```shell
docker run --env-file .env-uds -p 8081:8080 wultra.jfrog.io/wultra-docker/user-data-store:1.3.0-a2871e2f8d033dd4d00bf1a39456fc503be79194
```

You can specify a different port mapping in case the port 8081 on localhost is already used.

You can also use the alternative Docker configuration approaches, such as using the environmental variables directly as parameters or using Docker Compose, however be careful about not exposing the database credentials which are sensitive.

#### Securing the REST API

Now it's time to set up credentials for accessing the REST API.

At first, hash a password using following command (change the password in the command):

```shell
echo -n "password" | openssl dgst -sha256 -r
```

```sql
INSERT INTO uds_users (username, password, enabled) VALUES ('admin', '5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8', true);
INSERT INTO public.uds_authorities (username, authority) VALUES ('admin', 'ROLE_READ');
INSERT INTO public.uds_authorities (username, authority) VALUES ('admin', 'ROLE_WRITE');
```

For REST API authentication using OAuth2 / OIDC, see chapter [OAuth2.x / OpenID Connect (OIDC)](https://developers.wultra.com/components/user-data-store/1.3.x/documentation/Configuration-Properties#oauth2x--openid-connect-oidc).

#### Verification

You can verify that the user data store application is running by accessing http://localhost:8081/user-data-store/ and signing in with the `admin` username and generated password.

For accessing the User Data Store REST API documentation, use the following URL: http://localhost:8081/user-data-store/swagger-ui/index.html

### Liveness Check Proxy Configuration

Two providers of biometric liveness verification are supported:
- [iProov](https://www.iproov.com/)
- [Innovatrics](https://www.innovatrics.com/)

You will need to choose one of the providers and obtain respective API keys and configure them. Please contact Wultra support for obtaining the API keys.

#### Creating the Database

The steps for creating the database for Liveness Check Proxy are very similar to the User Data Store. In fact, you could use the same database, however we recommend to use two databases due to different sensitivity of stored records.

Run as database superuser:
```sql
CREATE DATABASE lcp;
CREATE USER lcp WITH PASSWORD 'lcp';
GRANT ALL PRIVILEGES ON DATABASE lcp TO lcp;
```

Connect to the created `lcp` database:
```shell
\c lcp
```

Grant privileges on the `public` schema within this database:
```sql
GRANT USAGE ON SCHEMA public TO lcp;
GRANT CREATE ON SCHEMA public TO lcp;
```

#### Docker Configuration

Following environmental variables need to be configured for Liveness Check Proxy:
- `LCP_DATASOURCE_URL` — database JDBC URL, such as `jdbc:postgresql://host.docker.internal:5432/lcp`
- `LCP_DATASOURCE_USERNAME` – database username you specified before
- `LCP_DATASOURCE_PASSWORD` – database password you specified before
- `LCP_USER_DETAILS_PROVIDER` - definition of photo storage, use value `user-data-store`
- `LCP_VERIFICATION_PROVIDER` - definition of biometry verification provider, you can use value `mock` for testing or `iproov` / `innovatrics` once access to the liveness verification provider is granted by Wultra
- `LCP_UDS_BASE_URL` – address to UDS component, e.g. `http://localhost:8081/user-data-store/`
- `LCP_UDS_REST_BASIC_ENABLED` – use true for enabled REST API authentication
- `LCP_UDS_REST_BASIC_USERNAME` – configured REST API username
- `LCP_UDS_REST_BASIC_PASSWORD` – configured REST API password

To specify the environmental variables for Docker, you can use an `.env-uds` file with the following content:
```shell
LCP_DATASOURCE_URL=your-db-url
LCP_DATASOURCE_USERNAME=your-db-username
LCP_DATASOURCE_PASSWORD=your-db-password
LCP_USER_DETAILS_PROVIDER=user-data-store
LCP_VERIFICATION_PROVIDER=mock
LCP_UDS_BASE_URL=http://localhost:8081/user-data-store/
LCP_UDS_REST_BASIC_ENABLED=true
LCP_UDS_REST_BASIC_USERNAME=configured-username-rest-auth
LCP_UDS_REST_BASIC_PASSWORD=configured-password-rest-auth
```

To start the Docker container, use:

```shell
docker run --env-file .env-lcp -p 8082:8080 wultra.jfrog.io/wultra-docker/liveness-check-proxy:0.1.0-SNAPSHOT-2024.06.26-0f5031f9d5ca260aa40c28305ddb2e93a7f4e74e
```
TODO: use the final image

You can specify a different port mapping in case the port 8082 on localhost is already used.

You can also use the alternative Docker configuration approaches, such as using the environmental variables directly as parameters or using Docker Compose, however be careful about not exposing the database credentials which are sensitive.

#### Configuration of Innovatrics Provider

In case the biometric verification provider is Innovatrics, configure following environmental variables:
```shell
LCP_INNOVATRICS_BASE_URL=innovatrics-base-url
LCP_INNOVATRICS_TOKEN=api-token
```

#### Configuration of iProov Provider

In case the biometric verification provider is iProov, configure following environmental variables:
```shell
LCP_IPROOV_BASE_URL=iproov-base-url
LCP_IPROOV_API_KEY=iproov-api-key
LCP_IPROOV_API_SECRET=iproov-api-secret
LCP_IPROOV_ASSURANCE_TYPE=liveness
LCP_IPROOV_OAUTH_CLIENT_USERNAME=iproov-oauth-client-username
LCP_IPROOV_OAUTH_CLIENT_PASSWORD=iproov-oauth-client-password
```

#### Securing the REST API

Now it's time to set up credentials for accessing the REST API.

At first, hash a password using following command (change the password in the command):

```shell
echo -n "password" | openssl dgst -sha256 -r
```

```sql
INSERT INTO lcp_user (username, password, enabled) VALUES ('admin', '5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8', true);
INSERT INTO lcp_authority (username, authority) VALUES ('admin', 'ROLE_ADMIN');
```

For more details, see the [REST Service Authentication Configuration](https://developers.wultra.com/components/liveness-check-proxy/1.0.x/documentation/Configuration-Properties#rest-service-authentication-configuration).

#### Verification

You can verify that the user data store application is running by accessing http://localhost:8082/

For accessing the User Data Store REST API documentation, use the following URL: http://localhost:8082/swagger-ui/index.html

## REST API Usage

In this chapter we will go through the main REST API endpoints which are required to perform a successful user biometric verification.

Before the biometric liveness verification can be initiated, static profile photos of users must be stored in the User Data Store component. They will serve as a basis for the liveness verification.

The actual biometric liveness verification has three distinct phases. At first, the liveness check must be initialized on the backend. Second, the mobile application must call the chosen verification provider SDK (iProov or Innovatrics). The third phase is again on the backend when the result of the check is acquired and evaluated.

### Uploading User Photos to User Data Store

Before using the biometric verification, you will need to import portrait photos into User Data Store, so that they can be used by Liveness Check Proxy as the basis for biometric verification. The Liveness Check Proxy calls the biometric liveness verification provider which compares the live check with the stored user picture to validate the user.

The User Data Store is designed to accommodate a variety of use cases across different projects, therefore is structured around generic structure called document. The Liveness Check Proxy specifically requires the presence of a document of type `photo` for each user to be verified, use the `photoType` with value `person` for the uploaded photo used for verification.

You can upload the photos individually by calling the `POST /admin/documents` endpoint of User Data Store:

```
{
  "requestObject": {
    "userId": "<user id>",
    "documentType": "photo",
    "dataType": "image_base64",
    "documentData": "{}",
    "attributes": {},
    "photos": [
      {
        "photoType": "person",
        "photoData": "<base64 encoded image>"
      }
    ]
  }
}
```

The `userId` parameter contains the unique identifier of the user within your system. The parameter `photoData` should contain the user profile photo encoded in Base-64 encoding. After calling this endpoint, you will obtain identifiers of created documents and photos.

For additional details, see the [User Data Store REST API documentation](https://developers.wultra.com/components/user-data-store/develop/documentation/User-Data-Store-API), particularly the Document API and Photo API documentation.

In case you need to upload many user photos at once, you can use the photo import endpoints which can be used for uploading many photos at once with multiple options for the format of stored photos and accessing these photos:

- Import Photos endpoint: [POST /admin/photos/import](https://developers.wultra.com/components/user-data-store/1.3.x/documentation/User-Data-Store-API#post-adminphotosimport)
- Import Photos from CVS endpoint: [POST /admin/photos/import/csv](https://developers.wultra.com/components/user-data-store/1.3.x/documentation/User-Data-Store-API#post-adminphotosimportcsv)

### Initiating Biometric Liveness Verification

You can initiate liveness verification for a user by calling the `POST /liveness/init` endpoint with following parameters:

```json
{
  "userId": "<user id>"
}
```

The `userId` parameter contains the unique identifier of the user within your system and should correspond to the value used when importing the user profile photo.

From the response, you will need to obtain generate `TOKEN` value in case iProov is used as a provider.
```json
{
  "initiationType": "TOKEN",
  "reference": "46a0c1fba98b9a1cae88353ae8279503"
}
```

You will need to use the `reference` value in the liveness verification step.

For Innovatrics, no token is generated and the response is following:
```json
{
  "initiationType": "NONE",
  "reference": null
}
```

For more details, see REST API documentation for endpoint [POST /liveness/init](https://developers.wultra.com/components/liveness-check-proxy/1.0.x/documentation/Liveness-Check-Proxy-REST-API#post-livenessinit). 

### Calling the Verification Provider SDK

See the verification provider documentation for calling the respective SDK.

For iProov, use the generated token (`reference` value) in the previous step.
https://github.com/iProov/ios/?tab=readme-ov-file#launch-the-sdk

For Innovatrics, the documentation is available here:
https://developers.innovatrics.com/digital-onboarding/technical/samples

### Verifying Biometric Liveness for a User

Starting the user liveness verification differs per provider. In case of iProov, the request is following:
```json
{
  "userId": "<user id>",
  "livenessRecord": {
    "reference": "46a0c1fba98b9a1cae88353ae8279503"
  }
}
```

Use the `userId` value used in first step and the `reference` value generated in the first step.

In case of Innovatrics, the request is following:
```json
{
  "userId": "<user id>",
  "livenessRecord": {
    "image": "c2VsZmllSW1hZ2U="
  }
}
```

The parameter `image` is the serialized face image captured by Innovatrics SDK during first step.

You can check whether the liveness check succeeded from the response, successful verification has value `ACCEPTED` for the `result` parameter:
```json
{
  "result": "ACCEPTED",
  "selfie": null
}
```

For more details, see REST API documentation for endpoint [POST /liveness/verify](https://developers.wultra.com/components/liveness-check-proxy/1.0.x/documentation/Liveness-Check-Proxy-REST-API#post-livenessverify).

## Summary

In this tutorial we have shown how to use User Data Store and Liveness Check Proxy components. At first, user photos were uploaded or imported, then a liveness check was initiated, followed by a liveness verification step. Additional details including advanced configuration parameters can be found in documentation of both of the components.
