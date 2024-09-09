Quick Intro to Spring Cloud Configuration
=========================================
*Source: https://www.baeldung.com/spring-cloud-configuration

**1\. Overview**
----------------

**_Spring Cloud Config_** is Spring’s client/server approach for storing and serving distributed configurations across multiple applications and environments.

This configuration store is ideally versioned under _Git_ version control and can be modified at application runtime. While it fits very well in Spring applications using all the supported configuration file formats together with constructs like _Environment_, _[PropertySource, or @Value](/properties-with-spring)_, it can be used in any environment running any programming language.

In this tutorial, we’ll focus on how to set up a _Git_\-backed config server, use it in a simple _REST_ application server, and set up a secure environment including encrypted property values.

**2\. Project Setup and Dependencies**
--------------------------------------

First, we’ll create two new _Maven_ projects. The server project is relying on the _[spring-cloud-config-server](https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-config-server)_ module, as well as the _[spring-boot-starter-security](https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-security)_ and _[spring-boot-starter-web](https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-web)_ starter bundles:

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

However, for the client project, we only need the _[spring-cloud-starter-config](https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-starter-config)_ and the _[spring-boot-starter-web modules](https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-web)_:

freestar.config.enabled\_slots.push({ placementName: "baeldung\_leaderboard\_mid\_1", slotId: "baeldung\_leaderboard\_mid\_1" });

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

**3\. A Config Server Implementation**
--------------------------------------

The main part of the application is a config class, more specifically a [_@SpringBootApplication_](/spring-boot-application-configuration), which pulls in all the required setup through the _auto-configure_ annotation _@EnableConfigServer:_

    @SpringBootApplication
    @EnableConfigServer
    public class ConfigServer {
        
        public static void main(String[] arguments) {
            SpringApplication.run(ConfigServer.class, arguments);
        }
    }
    

Now we need to configure the server _port_ on which our server is listening and a _Git_\-url, which provides our version-controlled configuration content. The latter can be used with protocols like _http_, _ssh,_ or a simple _file_ on a local filesystem.

**Tip:** If we’re planning to use multiple config server instances pointing to the same config repository, we can configure the server to clone our repo into a local temporary folder. But be aware of private repositories with two-factor authentication; they’re difficult to handle! In such a case, it’s easier to clone them on our local filesystem and work with the copy.

There are also some _placeholder variables and search patterns_ for configuring the _repository-url_ available; however, this is beyond the scope of our article. If you’re interested in learning more, the official documentation is a good place to start.

We also need to set a username and password for the _Basic-Authentication_ in our _application.properties_ to avoid an auto-generated password on every application restart:

freestar.config.enabled\_slots.push({ placementName: "baeldung\_leaderboard\_mid\_2", slotId: "baeldung\_leaderboard\_mid\_2" });

    server.port=8888
    spring.cloud.config.server.git.uri=ssh://localhost/config-repo
    spring.cloud.config.server.git.clone-on-start=true
    spring.security.user.name=root
    spring.security.user.password=s3cr3t

**4\. A Git Repository as Configuration Storage**
-------------------------------------------------

To complete our server, we have to initialize a _Git_ repository under the configured url, create some new properties files, and populate them with some values.

The name of the configuration file is composed like a normal Spring _application.properties_, but instead of the word ‘application,’ a configured name, such as the value of the property _‘spring.application.name’,_ of the client is used, followed by a dash and the active profile. For example:

    $> git init
    $> echo 'user.role=Developer' > config-client-development.properties
    $> echo 'user.role=User'      > config-client-production.properties
    $> git add .
    $> git commit -m 'Initial config-client properties'

**Troubleshooting:** If we run into _ssh_\-related authentication issues, we can double check _~/.ssh/known\_hosts_ and _~/.ssh/authorized\_keys_ on our ssh server.

**5\. Querying the Configuration**
----------------------------------

Now we’re able to start our server. The _Git_\-backed configuration API provided by our server can be queried using the following paths:

    /{application}/{profile}[/{label}]
    /{application}-{profile}.yml
    /{label}/{application}-{profile}.yml
    /{application}-{profile}.properties
    /{label}/{application}-{profile}.properties

The _{label}_ placeholder refers to a Git branch, _{application}_ to the client’s application name, and the _{profile}_ to the client’s current active application profile.

freestar.config.enabled\_slots.push({ placementName: "baeldung\_leaderboard\_mid\_3", slotId: "baeldung\_leaderboard\_mid\_3" });

So we can retrieve the configuration for our planned config client running under the development profile in branch _master_ via:

    $> curl http://root:s3cr3t@localhost:8888/config-client/development/master

**6\. The Client Implementation**
---------------------------------

Next, let’s take care of the client. This will be a very simple client application, consisting of a _REST_ controller with one _GET_ method.

To fetch our server, the configuration must be placed in the _application.properties_ file. Spring Boot 2.4 introduced a new way to load configuration data using the _**spring.config.import**_ property, which is now the default way to bind to Config Server:

    @SpringBootApplication
    @RestController
    public class ConfigClient {
        
        @Value("${user.role}")
        private String role;
    
        public static void main(String[] args) {
            SpringApplication.run(ConfigClient.class, args);
        }
    
        @GetMapping(
          value = "/whoami/{username}",  
          produces = MediaType.TEXT_PLAIN_VALUE)
        public String whoami(@PathVariable("username") String username) {
            return String.format("Hello! 
              You're %s and you'll become a(n) %s...\n", username, role);
        }
    }

In addition to the application name, we also put the active profile and the connection-details in our _application.properties_:

    spring.application.name=config-client
    spring.profiles.active=development
    spring.config.import=optional:configserver:http://root:s3cr3t@localhost:8888

This will connect to the Config Server at http://localhost:8888 and will also use HTTP basic security while initiating the connection. We can also set the username and password separately using `spring.cloud.config.username` and `spring.cloud.config.password` properties, respectively.

In some cases, we may want to fail the startup of a service if it isn’t able to connect to the Config Server. If this is the desired behavior, we can remove the **optional:** prefix to make the client halt with an exception.

To test if the configuration is properly received from our server, and the _role value_ gets injected in our controller method, we simply curl it after booting the client:

    $> curl http://localhost:8080/whoami/Mr_Pink

If the response is as follows, our _Spring Cloud Config Server_ and its client are working fine for now:

freestar.config.enabled\_slots.push({ placementName: "baeldung\_incontent\_1", slotId: "baeldung\_incontent\_1" });

    Hello! You're Mr_Pink and you'll become a(n) Developer...

**7\. Encryption and Decryption**
---------------------------------

**Requirement**: To use cryptographically strong keys together with Spring encryption and decryption features, we need the _‘Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files’_ installed in our _JVM._ These can be downloaded, for example, from [Oracle](http://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html). To install, follow the instructions included in the download. Some Linux distributions also provide an install-able package through their package managers.

Since the config server is supporting the encryption and decryption of property values, we can use public repositories as storage for sensitive data, like usernames and passwords. Encrypted values are prefixed with the string _{cipher},_ and can be generated by a REST-call to the path _‘/encrypt’_ if the server is configured to use a symmetric key or a key pair.

An endpoint to decrypt is also available. Both endpoints accept a path containing placeholders for the name of the application and its current profile: _‘/\*/{name}/{profile}’._ This is especially useful for controlling cryptography per client. However, before they can be useful, we have to configure a cryptographic key, which we’ll do in the next section.

**Tip:** If we use curl to call the en-/decryption API, it’s better to use the _–data-urlencode_ option (instead of _–data/-d_), or set the ‘Content-Type’ header explicit to _‘text/plain’_. This ensures a correct handling of special characters like ‘+’ in the encrypted values.

If a value can’t be decrypted automatically while fetching through the client, its _key_ is renamed with the name itself, prefixed by the word ‘invalid.’ This should prevent the use of an encrypted value as password.

**Tip:** When setting-up a repository containing YAML files, we have to surround our encrypted and prefixed values with single-quotes. However, this isn’t the case with Properties.

### 7.1. **CSRF**

By default, Spring Security enables [CSRF](/spring-security-csrf) protection for all the requests sent to our application.

Therefore, to be able to use the _/encrypt_ and _/decrypt_ endpoints, let’s disable the CSRF for them:

freestar.config.enabled\_slots.push({ placementName: "baeldung\_incontent\_2", slotId: "baeldung\_incontent\_2" });

    @Configuration
    public class SecurityConfiguration {
    
        @Bean
        public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
            http.csrf(csrf -> csrf.ignoringRequestMatchers(
               "/encrypt/**", "/decrypt/**"
            ))
    
            //...
        }
    }

### **7.2. Key Management**

By default, the config server is able to encrypt property values in a symmetric or asymmetric way.

**To use symmetric cryptography**, we simply have to set the property _‘encrypt.key’_ in our _application.properties_ to a secret of our choice_._ Alternatively, we can pass-in the environment variable _ENCRYPT\_KEY_.

**For asymmetric cryptography**, we can set _‘encrypt.key’_ to a _PEM_\-encoded string value or configure a _keystore_ to use.

Since we need a highly secured environment for our demo server, we’ll choose the latter option, along with generating a new keystore, including a _RSA_ key-pair, with the Java _keytool_ first:

    $> keytool -genkeypair -alias config-server-key \
           -keyalg RSA -keysize 4096 -sigalg SHA512withRSA \
           -dname 'CN=Config Server,OU=Spring Cloud,O=Baeldung' \
           -keypass my-k34-s3cr3t -keystore config-server.jks \
           -storepass my-s70r3-s3cr3t

Then we’ll add the created keystore to our server’s application_.properties_ and re-run it:

    encrypt.keyStore.location=classpath:/config-server.jks
    encrypt.keyStore.password=my-s70r3-s3cr3t
    encrypt.keyStore.alias=config-server-key
    encrypt.keyStore.secret=my-k34-s3cr3t

Next, we’ll query the encryption-endpoint, and add the response as a value to a configuration in our repository:

    $> export PASSWORD=$(curl -X POST --data-urlencode d3v3L \
           http://root:s3cr3t@localhost:8888/encrypt)
    $> echo "user.password={cipher}$PASSWORD" >> config-client-development.properties
    $> git commit -am 'Added encrypted password'
    $> curl -X POST http://root:s3cr3t@localhost:8888/refresh

To test if our setup works correctly, we’ll modify the _ConfigClient_ class and restart our client:

    @SpringBootApplication
    @RestController
    public class ConfigClient {
    
        ...
        
        @Value("${user.password}")
        private String password;
    
        ...
        public String whoami(@PathVariable("username") String username) {
            return String.format("Hello! 
              You're %s and you'll become a(n) %s, " +
              "but only if your password is '%s'!\n", 
              username, role, password);
        }
    }

Finally, a query against our client will show us if our configuration value is being correctly decrypted:

freestar.config.enabled\_slots.push({ placementName: "baeldung\_incontent\_3", slotId: "baeldung\_incontent\_3" });

    $> curl http://localhost:8080/whoami/Mr_Pink
    Hello! You're Mr_Pink and you'll become a(n) Developer, \
      but only if your password is 'd3v3L'!

### **7.3. Using Multiple Keys**

If we want to use multiple keys for encryption and decryption, such as a dedicated one for each served application, we can add another prefix in the form of _{name:value}_ between the _{cipher}_ prefix and the _BASE64_\-encoded property value.

The config server understands prefixes like _{secret:my-crypto-secret}_ or _{key:my-key-alias}_ almost out-of-the-box. The latter option needs a configured keystore in our _application.properties_. This keystore is searched for a matching key alias. For example:

    user.password={cipher}{secret:my-499-s3cr3t}AgAMirj1DkQC0WjRv...
    user.password={cipher}{key:config-client-key}AgAMirj1DkQC0WjRv...

For scenarios without keystore, we have to implement a _@Bean_ of type _TextEncryptorLocator,_ which handles the lookup and returns a _TextEncryptor_\-Object for each key.

### **7.4. Serving Encrypted Properties**

If we want to disable server-side cryptography and handle the decryption of property-values locally, we can put the following in our server’s _application.properties_:

    spring.cloud.config.server.encrypt.enabled=false

Furthermore, we can delete all the other ‘encrypt.\*’ properties to disable the _REST_ endpoints.

**8\. Conclusion**
------------------

Now we’re able to create a configuration server to provide a set of configuration files from a _Git_ repository to client applications. There are also a few other things we can do with such a server.

For example:

*   Serve configuration in _YAML_ or _Properties_ format instead of _JSON,_ also with placeholders resolved. This can be useful when using it in non-Spring environments, where the configuration isn’t directly mapped to a _PropertySource_.
*   Serve plain text configuration files in turn, optionally with resolved placeholders. For instance, this can be useful to provide an environment-dependent logging-configuration.
*   Embed the config server into an application, where it configures itself from a _Git_ repository, instead of running as a standalone application serving clients. Therefore, we must set some properties and/or we must remove the _@EnableConfigServer_ annotation, which depends on the use case.
*   Make the config server available at Spring Netflix Eureka service discovery and enable automatic server discovery in config clients. This becomes important if the server has no fixed location or it moves in its location.