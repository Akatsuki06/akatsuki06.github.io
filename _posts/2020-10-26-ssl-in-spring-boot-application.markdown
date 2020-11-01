---
layout: post
title:  SSL in a Spring Boot application
date:   2020-10-26 15:10:56 +0530
categories: ssl security springboot
---


SSL (or its successor TLS) is a communication  protocol used to transfer encrypted data over a network. It involves  <ins>Authentication</ins> of the entities and <ins>Encryption/Decryption</ins>  of data being shared between a web server and client.

This article talks about SSL and how we can setup it on a spring boot application.

### **Authentication**

Authentication of entities on a network is established using digital certificate. Typically in a one way handshake, the server has a keystore which holds a private key, a public key and a *x509*  certificate where the public and private keys form a pair, private key is used for decryption and public key is used for encryption of data.

There is a truststore which stores all trusted certificates signed by CA. Client/Server refers to it to validate a certificate.

When a client sends a request to server, the server responds back with the certificate and the public key. 

Client authenticates the server by validating if the certificate received is present in truststore or not. In many cases the server would also validate its client that process is referred to as two way handshake.

### **Encryption**

Once the server is found authentic the client generates a shared key and encrypts it using server's public key. It sends the encrypted shared key to the server where the server decrypts it using its private key. Now both server and client have a shared session key which can be used for encryption and decryption of data being transferred between them.

![image](https://user-images.githubusercontent.com/16136908/97805333-72f2c500-1c7b-11eb-83a3-49ce02376c9b.png)

>> The server presents its public key and certificates from keystore to the client, the client validates the certificate is present in truststore so that the server can be trusted.

<br>  

## Setting up SSL in spring boot application

### Generating the keystore and truststore

```bash

# Generate a private key
 openssl genrsa -des3 -out privatekey.key 1024

# Generate Certificate signeding Request (CSR)
# CN should be the domain-name or localhost for running in local
openssl req -new -key privatekey.key -out csrfile.csr

# Generate self signed certificate
openssl x509 -req -days 365 -in csrfile.csr -signkey privatekey.key -out public.crt

# Import the self signed certificate to a truststore file
keytool -import -file public.crt -alias exampleCA -keystore truststore.jks

# Import the keypair and certificates to a keystore file
openssl pkcs12 -export -in public.crt -inkey privatekey.key -certfile public.crt -name "certs" -out keystore.p12

# to convert p12 to jks
keytool -importkeystore -srckeystore keystore.p12 -srcstoretype pkcs12 -destkeystore keystore.jks -deststoretype JKS

```



## Configuring the server

Once the certificates are generated copy the truststore and keystore .jks files to the classpath of your spring boot application.
Here I am using spring `2.3.4.RELEASE`, spring-security dependency is required in the pom file.

```xml
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

```

Add a `WebSecurityConfig` extending `WebSecurityConfigurerAdapter` class to allow request on https
and block any request coming from a non-secure HTTP channel. 

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.requiresChannel()
                .anyRequest()
                .requiresSecure();
    }
}
```

Update the spring application yaml file as below.

```yaml
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:keystore.jks
    key-store-password: password
    trust-store: classpath:truststore.jks
    trust-store-password: password
    client-auth: need

```

### Test it out

Create one example controller:

```java
@RestController
@RequestMapping("/ssl")
public class ExampleController {

    @RequestMapping(path="/example", method= RequestMethod.GET)
    public String apiRequest() {
        return "Successfully Validated!";
    }
}
```

In the application tests, create a RestClient configuration which would access the server with ssl.
`SSLContextBuilder` and `HttpClients` are part of apache httpclients library.

```java

@Configuration
public class RestClientTest {

    private String password = "password";

    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) throws Exception {
        SSLContext sslContext = SSLContextBuilder
                .create()
                .loadKeyMaterial(ResourceUtils.getFile("classpath:keystore.jks"),
                                         password.toCharArray(), password.toCharArray())
                .loadTrustMaterial(ResourceUtils.getFile("classpath:truststore.jks"), password.toCharArray())
                .build();

        return builder
                .requestFactory(
                        ()->new HttpComponentsClientHttpRequestFactory(
                                HttpClients.custom()
                                .setSSLContext(sslContext)
                                .build()
                        )
                )
                .build();
    }
}

```

A sample junit test to test the endpoint. Note the host url should have `https` protocol or it will fail during SSL handshake.

```java
@RunWith(SpringRunner.class)
@SpringBootTest(
    classes = Application.class,
    webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT
)
public class ApplicationTests {

    public ApplicationTests(){
    }

    @LocalServerPort
    private int port;

    @Autowired
    private RestTemplate restTemplate;

    @Test
    public void withSSL() {
      String response = restTemplate.getForObject("https://localhost:" + port + "/ssl/example", String.class);
      Assert.assertEquals("Successfully Validated!", response);
    }

}

```

