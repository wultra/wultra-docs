# Dynamic TLS/SSL Certificate Pinning

<!-- AUTHOR joshis_tweets 2023-05-23T00:00:00Z -->
<!-- SIDEBAR _Sidebar_Server.md sticky -->
<!-- TEMPLATE tutorial -->

To deploy the server-side components, you have two options:

- Use a hosted service by Wultra at `*.sslpinning.com` sub-domain.
- Deploy the service on your own domain (other than any of the pinned domains).

## Create Database Structure

The database structure is extremely simple, we provide a PostgreSQL snippets to create it:

- [Database Structure](https://github.com/wultra/mobile-utility-server/blob/develop/docs/Database-Structure.md)

## Run Docker Image

Create an `env.list` file with the following contents (adjust the values according to your setup):

```rb
MOBILE_UTILITY_SERVER_DATASOURCE_URL=jdbc:postgresql://host.docker.internal:5432/postgres
MOBILE_UTILITY_SERVER_DATASOURCE_USERNAME=username
MOBILE_UTILITY_SERVER_DATASOURCE_PASSWORD=Pa5sw0rd
```

You can now run Docker image via:

```sh
docker login wultra.jfrog.io
docker pull wultra.jfrog.io/wultra-docker/mobile-utility-server
docker run --env-file deploy/env.list -d -it -p 8080:8000 --name=mobile-utility-server mobile-utility-server
```

Alternatively, you can build and deploy own Docker image or a Spring Boot app, using sources available on Github:

- https://github.com/wultra/mobile-utility-server


## Configuration

Once the server infrastructure is running, you can easily add a new mobile application by running the following API call:

```sh
curl --request POST \
  --url http://localhost:8080/admin/apps \
  --header 'Authorization: Basic YWRtaW46YWRtaW4=' \
  --header 'Content-Type: application/json' \
  --data '{
  "name": "mobile-app",
  "displayName": "My Mobile App"
}'
```

To set initial certificate values, you can let our systems do the heavy lifting and fetch the SSL certificate automatically:

```sh
curl --request POST \
  --url http://localhost:8080/admin/apps/mobile-app/certificates/auto \
  --header 'Authorization: Basic YWRtaW46YWRtaW4=' \
  --header 'Content-Type: application/json' \
  --data '{
  "domain": "google.com"
}'
```

You can then easily add or update certificates by importing PEM format:

```sh
curl --request POST \
  --url http://localhost:8080/admin/apps/mobile-app/certificates/pem \
  --header 'Authorization: Basic YWRtaW46YWRtaW4=' \
  --header 'Content-Type: application/json' \
  --data '{
  "domain": "google.com",
  "pem": "-----BEGIN CERTIFICATE-----\nMIIOOzCCDSOgAwIBAgI...YgSeDAIcsw=\n-----END CERTIFICATE-----"
}'
```

The application publishes a Swagger UI documentation at the `/swagger-ui.html` path with up-to-date information about published endpoints.

## Resources

You can find more details in the following documentation:

- [Dynamic SSL Pinning SDK for iOS](https://github.com/wultra/ssl-pinning-ios)
- [Dynamic SSL Pinning SDK for Android](https://github.com/wultra/ssl-pinning-android)
- [Mobile Utility Server](https://github.com/wultra/mobile-utility-server)

## Continue Reading

Proceed with one of the following chapters:

- [Dynamic SSL Pinning Overview](./Readme.md)
- [Dynamic SSL Pinning on iOS](./iOS-Tutorial.md)
- [Dynamic SSL Pinning on Android](./Android-Tutorial.md)
