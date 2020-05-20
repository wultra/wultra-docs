# Implementing the Server-Side for Authentication in Mobile Banking Apps (SCA)

<!-- AUTHOR joshis_tweets 2020-05-04T00:00:00Z -->

In this tutorial, we will show you how to deploy and implement back-end components for authentication in mobile banking or fintech apps.

This tutorial has four parts:

- [Mobile Authentication Overview](Readme.md)
- **Tutorial for Server Side Developers**
- [Tutorial for iOS Developers](iOS-Tutorial.md)
- [Tutorial for Android Developers](Android-Tutorial.md)

## Prerequisites

This tutorial assumes, that you have:

- Read and understood the [Mobile Authentication Overview](Readme.md)
- [Java 11](https://jdk.java.net/) installation, or newer.
- [Apache Tomcat 9.0](https://tomcat.apache.org/download-90.cgi) installation.
- [PostgreSQL 12 database](https://www.postgresql.org/) installation.
- Java IDE for developing Spring Boot applications (we will use [IntelliJ Idea](https://www.jetbrains.com/idea/)).

_Note: PowerAuth Server supports various application servers and JPA 2.x compatible database engines. For the sake of simplicity, we decided to go with setup on Tomcat and PostgreSQL._

## Introduction

When implementing a server-side support for the Mobile Security Suite, you need to perform several tasks:

- Prepare the server infrastructure and database schema.
- Deploy and explore the server-side components.
- Implement mobile app management in the Internet banking.
- Customize the enrollment server to allow custom activations.
- Prepare an API resource server with protected resources.

## Preparing the Infrastructure

Besides the Apache Tomcat 9.0 and PostgreSQL database installation and configuration, you need to perform several generic configuration tasks:

- Add PostgreSQL JDBC Connector library (JAR) to the server's classpath.
- Add Bouncy Castle JCE Provider library (JAR) to the server's classpath.
- Starting Tomcat, if not running.

### Adding Required Libraries

To add the PostgreSQL JDBC library, copy the [PostgreSQL JDBC Driver JAR file](https://jdbc.postgresql.org/download.html) to `$CATALINA_HOME/lib` folder.

You can add the Bouncy Castle JCE provider the same way. Copy the [Bouncy Castle Provider JAR](https://www.bouncycastle.org/latest_releases.html) (`bcprov-jdk15on-${VERSION}.jar`) to `$CATALINA_HOME/lib` folder.

Restart the Apache Tomcat instance for these changes to take effect:

{% codetabs %}
{% codetab macOS %}
```sh
$CATALINA_HOME/bin/catalina stop
$CATALINA_HOME/bin/catalina start
```
{% endcodetab %}
{% codetab Linux %}
```sh
$CATALINA_HOME/bin/catalina.sh stop
$CATALINA_HOME/bin/catalina.sh start
```
{% endcodetab %}
{% codetab Windows %}
```sh
$CATALINA_HOME/bin/catalina.bat stop
$CATALINA_HOME/bin/catalina.bat start
```
{% endcodetab %}
{% endcodetabs %}

### Preparing the Database Schema

Execute the following scripts in your PostgreSQL database to create the required tables:

- [PostgreSQL - Create Schema Script](https://github.com/wultra/powerauth-server/blob/develop/docs/sql/postgresql/create_schema.sql)

## Deploy the Server-Side Components

For this tutorial, we will deploy two components:

- [PowerAuth Server](https://github.com/wultra/powerauth-server/blob/develop/docs/Readme.md) - The server responsible for mobile device management.
- [PowerAuth Admin](https://github.com/wultra/powerauth-admin/blob/develop/docs/Readme.md) - The administration console for the PowerAuth Server.

### Deploy the PowerAuth Server

First, prepare the required configuration XML file called `powerauth-java-server.xml` ([here](./powerauth-java-server.xml) is a template for download). In a minimal configuration, the only thing you need to configure is the JDBC database connectivity properties:

{% codetabs %}
{% codetab powerauth-java-server.xml %}
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Context>

    <!-- Database Configuration - JDBC -->
    <Parameter name="spring.datasource.driver-class-name" value="org.postgresql.Driver"/>
    <Parameter name="spring.datasource.url" value="jdbc:postgresql://localhost:5432/postgres"/>
    <Parameter name="spring.datasource.username" value="$YOUR_DB_USERNAME"/>
    <Parameter name="spring.datasource.password" value="$YOUR_DB_PASSWORD"/>

</Context>
```
{% endcodetab %}
{% endcodetabs %}

_Note: All our applications are a common Spring Boot applications and therefore, you can configure any other well-known Spring Boot properties._

Next, copy the `powerauth-java-server.xml` configuration file to `$CATALINA_HOME/conf/Catalina/localhost/` folder. Tomcat automatically picks up the file and will use the configuration for the `/powerauth-java-server` context.

[Download the latest PowerAuth Server](https://github.com/wultra/powerauth-server/releases) (`powerauth-java-server.war` file) and copy the WAR file to `$CATALINA_HOME/webapps` folder.

You can now open [PowerAuth Server Welcome Page](http://localhost:8080/powerauth-java-server/) at [http://localhost:8080/powerauth-java-server/](http://localhost:8080/powerauth-java-server/) address.

![ PowerAuth Server Welcome Page ](./08.png)

The welcome page shows the version info, important configuration properties, and links to important resources.

### Deploy the PowerAuth Admin

Deploying PowerAuth Admin, a GUI console for PowerAuth Server, follows a similar pattern as deploying the PowerAuth Server.

First, prepare an XML configuration file `powerauth-admin.xml` ([here](./powerauth-admin.xml) is a template for download). This time, the file only contains a single property: a SOAP interface address of the PowerAuth Server instance running on `localhost:8080` address.

{% codetabs %}
{% codetab powerauth-admin.xml %}
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Context>
    <Parameter name="powerauth.service.url" value="http://localhost:8080/powerauth-java-server/soap"/>
</Context>
```
{% endcodetab %}
{% endcodetabs %}

Next, copy the `powerauth-admin.xml` configuration file to `$CATALINA_HOME/conf/Catalina/localhost/` folder. Tomcat automatically picks up the file and will use the configuration for the `/powerauth-admin` context.

[Download the latest PowerAuth Admin](https://github.com/wultra/powerauth-admin/releases) (`powerauth-admin.war` file) and copy the WAR file to `$CATALINA_HOME/webapps` folder.

You can now open [PowerAuth Admin](http://localhost:8080/powerauth-admin/) console at [http://localhost:8080/powerauth-admin/](http://localhost:8080/powerauth-admin/) address.

#### Preparing the Mobile App Credentials

You can use the PowerAuth Admin to generate the mobile app credentials. Mobile app developers will need those credentials in order to configure the mobile SDK. With the empty system, you can create your application simply by providing an "application name". Use some technical format for the application name, rather than a fancy visual name.

We will use `demo-application` as an app name:

![ PowerAuth Admin - New Application ](./09.png)

After submitting the new application, you will see the values of application key, application secret, and master server public key. Provide your mobile developers with those values, so that they can configure their mobile apps.

![ PowerAuth Admin - Application Credentials ](./10.png)

## Mobile App Management via the Internet Banking

The end user should have an overview of the devices that are activated with his/her account. This is usually done by implementing a specific "self-service" section in the Internet banking. The goal of such section should be to allow typical administrative tasks, such as:

- Creating a new activation via an activation code.
- Listing the current active devices.
- Blocking or removing an active device.

The same functionality is usually implemented in the banking back-office application, so that the bank operators can manage mobile devices for their clients.

### Activation using Activation Code

The easiest way to activate a mobile client app is using an activation code. You can generate a new activation code for a particular user (this is how the user identity is connected with the mobile app) by calling the services published by the PowerAuth Server.

### Listing Active Devices

// TODO:

### Block, Unblock and Remove Devices

// TODO:

## Customizing the Enrollment Server

Enrollment Server is the component that the mobile app actually calls. No calls are performed from the mobile app to the PowerAuth Server - this component should be deployed in a secure infrastructure.

**For the sake of simplicity, we will deploy all components into a single Tomcat instance. However, you should use two Tomcat instances for the production deployment.**

This is the part where you might need to start with an actual programming, since the enrollment process can be customized. For example, you can use custom user credentials as an identity proof for the enrollment - we will show you that part.

### Clone the Git Repository

Start by cloning the Enrollment Server git repository and switching into the project folder:

```sh
git clone https://github.com/wultra/enrollment-server.git
cd enrollment-server
```

You can use the version from the `develop` branch but this might get tricky, since you would have to install our development dependencies. Therefore, we suggest using a version from some of the release branches or - ideally - tags, for example:

```sh
# Replace the version number with the desired version.
git checkout tags/0.24.0 -b tags/0.24.0
```

### Building the Project

The project uses Maven for the dependency management and project builds. You can build project simply by calling a `mvn package` command:

```sh
mvn clean package
```

The resulting output artifact is `./target/enrollment-server-0.24.0.war`. You can rename the file to just `enrollment-server.war`.

### Deploying a Vanilla Enrollment Server

Deploying Enrollment Server follows a similar pattern as deploying the PowerAuth Server and PowerAuth Admin.

First, prepare an XML configuration file `enrollment-server.xml` ([here](./enrollment-server.xml) is a template for download). The file again only contains a single property: a SOAP interface address of the PowerAuth Server instance running on `localhost:8080` address.

{% codetabs %}
{% codetab enrollment-server.xml %}
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Context>
    <Parameter name="powerauth.service.url" value="http://localhost:8080/powerauth-java-server/soap"/>
</Context>
```
{% endcodetab %}
{% endcodetabs %}

Next, copy the `enrollment-server.xml` configuration file to `$CATALINA_HOME/conf/Catalina/localhost/` folder. Tomcat automatically picks up the file and will use the configuration for the `/enrollment-server` context.

Copy the `enrollment-server.war` file you just built to `$CATALINA_HOME/webapps` folder.

You can now open [Enrollment Server Swagger UI](http://localhost:8080/enrollment-server/swagger-ui.html) console at [http://localhost:8080/enrollment-server/swagger-ui.html](http://localhost:8080/enrollment-server/swagger-ui.html) address to see the published resources.

![ Enrollment Server Swagger UI Screen ](./11.png)

You can then provide your mobile app development team with the Enrollment Server base URL, which they need for their mobile app configuration.

### Custom Activation in Enrollment Server

Until now, the Enrollment Server only uses the default (built-in) functionality and activation mechanisms. Specifically, only activation via activation code and activation via recovery code are present. In this part of the tutorial, we will show you how to implement activation using custom credentials.

Specifically, we will implement a mechanism that will allow the user to activate the mobile app via:

- **Username** - Any user identifier, such as e-mail or a client number.
- **Password** - User's password, for example, for the Internet banking.
- **OTP Code** - A one-time password that user obtained elsewhere, for example, from the HW token authenticator or via an SMS message.

Since these credentials are specific for you and your systems, you need to have your own service (denoted as `MyService` in the example below) to verify those credentials and translate them to a user ID in case authentication was successful.

To start with the customizations, open the IDE of your choice and import the project. Most Java IDEs support Maven out of the box, so this part should be easy. After importing the project, you should see the default project structure.

![ Default Enrollment Server Project Structure ](./12.png)

Start by adding a new `com.wultra.app.enrollmentserver.customization` package and create a new `MyActivationProvider` class that implements the `CustomActivationProvider` interface. In this tutorial, we will only implement two overriden methods:

- `lookupUserIdForAttributes` - This is a method that translates provided credentials (user identity proof) to a particular user ID. This method may either return `null` in case credentials do not match, or throw a new `PowerAuthActivationException`.
- `shouldAutoCommitActivation` - This method specifies if the activation should be automatically committed after the key exchange, or if a cooperation of some other system is required (for example, confirming the activation via the Internet banking). In our case, we will implement this method so that it auto-commits activation after custom activation is processed.

{% codetabs %}
{% codetab MyActivationProvider.java %}
```java
public class MyActivationProvider implements CustomActivationProvider {

    /**
     * Your identity service that can be used to verify the user credentials.
     */
    private final MyIdentityService myIdentityService;

    @Override
    public String lookupUserIdForAttributes(Map<String, String> identityAttributes) throws PowerAuthActivationException {
        // Fetch the user's credentials.
        String username = identityAttributes.get("username");
        String password = identityAttributes.get("password");
        String otp      = identityAttributes.get("otp");

        // Verify the credentials using your identity service.
        String userId = myIdentityService.verifyCredentials(username, password, otp);

        // If the credentials verification failed, throw an exception.
        if (userId == null) {
            throw new PowerAuthActivationException("Authentication failed");
        }

        // ... otherwise, return the user ID of an authenticated user.
        return userId;
    }

    @Override
    public boolean shouldAutoCommitActivation(Map<String, String> identityAttributes, Map<String, Object> customAttributes, String activationId, String userId, ActivationType activationType) throws PowerAuthActivationException {
        // Automatically commit activation for a CUSTOM activation type.
        // You can use more request attributes, either in identityAttributes or
        // customAttributes, for a more fine-grained control.
        return ActivationType.CUSTOM.equals(activationType);
    }
}
```
{% endcodetab %}
{% endcodetabs %}

And that's it! Your enrollment server will now process the custom activation credentials sent from the mobile clients, allowing an activation via the custom credentials. You can now build the project again and deploy the Enrollment Server just as you did earlier.

## Preparing API Resource Server With Protected Resources

// TODO:

## Continue Reading

Proceed with one of the following chapters:

- [Mobile Authentication Overview](Readme.md)
- [Tutorial for iOS Developers](iOS-Tutorial.md)
- [Tutorial for Android Developers](Android-Tutorial.md)
