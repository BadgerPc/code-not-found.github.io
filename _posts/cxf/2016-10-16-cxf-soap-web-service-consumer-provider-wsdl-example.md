---
title: CXF - SOAP Web Service Consumer &amp; Provider WSDL Example
permalink: /2016/10/cxf-soap-web-service-consumer-provider-wsdl-example.html
excerpt: A detailed step-by-step tutorial on how to implement a Hello World web service starting from a WSDL and using Apache CXF and Spring Boot.
date: 2016-10-16 21:00
categories: [Apache CXF]
tags: [Apache CXF, Client, Consumer, Contract First, CXF, Endpoint, Example, Hello World, Maven, Provider, Spring Boot, Tutorial, WSDL]
---

<figure>
    <img src="{{ site.url }}/assets/images/logos/apache-cxf-logo.png" alt="apache cxf logo">
</figure>

[Apache CXF](http://cxf.apache.org/) is an open source services framework that helps build and develop services using frontend programming APIs, like JAX-WS. In this tutorial we will take a look at how we can integrate CXF with Spring Boot in order to build and run a Hello World SOAP service. Throughout the example, we will be creating a contract first client and endpoint using a WSDL, CXF, Spring Boot and Maven.

Tools used:
* Apache CXF 3.1
* Spring Boot 1.4
* Maven 3

This tutorial is actually a newer version of an previous [example in which we used Jetty to host a Hello World CXF web service]({{ site.url }}//2014/08/jaxws-cxf-contract-first-hello-world-webservice-tutorial.html). This time we will be using the embedded Tomcat server that ships with Spring Boot as a runtime for the service.

The below code is organized in such a way that you can choose to only run the [client]({{ site.url }}/2016/10/cxf-soap-web-service-consumer-provider-wsdl-example.html#creating-the-client-consumer) (consumer) or [endpoint]({{ site.url }}/2016/10/cxf-soap-web-service-consumer-provider-wsdl-example.html#creating-the-endpoint-provider) (provider) part. In the below example we will setup both parts and then make an end-to-end test in which the client calls the endpoint.

# General Project Setup

In this example we will start from an existing WSDL file (contract-first) which is shown below. The content represent a SOAP service in which a person is sent as input and a greeting is received as a response.

``` xml
<?xml version="1.0"?>
<wsdl:definitions name="HelloWorld"

    targetNamespace="http://codenotfound.com/services/helloworld"
    xmlns:tns="http://codenotfound.com/services/helloworld" xmlns:types="http://codenotfound.com/types/helloworld"
    xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/" xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">

    <wsdl:types>
        <xsd:schema targetNamespace="http://codenotfound.com/types/helloworld"
            xmlns:xsd="http://www.w3.org/2001/XMLSchema"
            elementFormDefault="qualified" attributeFormDefault="unqualified"
            version="1.0">

            <xsd:element name="person">
                <xsd:complexType>
                    <xsd:sequence>
                        <xsd:element name="firstName" type="xsd:string" />
                        <xsd:element name="lastName" type="xsd:string" />
                    </xsd:sequence>
                </xsd:complexType>
            </xsd:element>

            <xsd:element name="greeting">
                <xsd:complexType>
                    <xsd:sequence>
                        <xsd:element name="greeting" type="xsd:string" />
                    </xsd:sequence>
                </xsd:complexType>
            </xsd:element>
        </xsd:schema>
    </wsdl:types>

    <wsdl:message name="SayHelloInput">
        <wsdl:part name="person" element="types:person" />
    </wsdl:message>

    <wsdl:message name="SayHelloOutput">
        <wsdl:part name="greeting" element="types:greeting" />
    </wsdl:message>

    <wsdl:portType name="HelloWorld_PortType">
        <wsdl:operation name="sayHello">
            <wsdl:input message="tns:SayHelloInput" />
            <wsdl:output message="tns:SayHelloOutput" />
        </wsdl:operation>
    </wsdl:portType>

    <wsdl:binding name="HelloWorld_SoapBinding" type="tns:HelloWorld_PortType">
        <soap:binding style="document"
            transport="http://schemas.xmlsoap.org/soap/http" />
        <wsdl:operation name="sayHello">
            <soap:operation
                soapAction="http://codenotfound.com/services/helloworld/sayHello" />
            <wsdl:input>
                <soap:body use="literal" />
            </wsdl:input>
            <wsdl:output>
                <soap:body use="literal" />
            </wsdl:output>
        </wsdl:operation>
    </wsdl:binding>

    <wsdl:service name="HelloWorld_Service">
        <wsdl:documentation>Hello World service</wsdl:documentation>
        <wsdl:port name="HelloWorld_Port" binding="tns:HelloWorld_SoapBinding">
            <soap:address
                location="http://localhost:9090/codenotfound/ws/helloworld" />
        </wsdl:port>
    </wsdl:service>

</wsdl:definitions>
```

Maven is used to build and run the example. The Hello World service endpoint will be hosted on an embedded Apache Tomcat server that ships directly with [Spring Boot](https://projects.spring.io/spring-boot/). To facilitate the management of the different Spring dependencies, [Spring Boot Starters](https://github.com/spring-projects/spring-boot/tree/master/spring-boot-starters) are used which are a set of convenient dependency descriptors that you can include in your application.

To avoid having to manage the version compatibility of the different Spring dependencies, we will inherit the defaults from the `spring-boot-starter-parent` parent POM.

There is actually a [Spring Boot starter specifically for CXF](http://cxf.apache.org/docs/springboot.html) that takes care of importing the needed Spring Boot dependencies. In addition it automatically registers `CXFServlet` with a <var>'/services/'</var> URL pattern for serving CXF JAX-WS endpoints and it offers some properties for configuration of the `CXFServlet`. In order to use the starter we declare a dependency to `cxf-spring-boot-starter-jaxws` in our Maven POM file.

For Unit testing our Spring Boot application we also include the `spring-boot-starter-test` dependency.

To take advantage of Spring Boot's capability to create a single, runnable "über-jar", we also include the `spring-boot-maven-plugin` Maven plugin. This also allows quickly starting the web service via a Maven command.

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.codenotfound</groupId>
    <artifactId>jaxws-cxf-helloworld-example</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>jaxws-cxf-helloworld-example</name>
    <description>CXF - SOAP Web Service Consumer & Provider WSDL Example</description>
    <url>http://</url>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.4.1.RELEASE</version>
        <relativePath /> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>

        <cxf.version>3.1.7</cxf.version>
    </properties>

    <dependencies>
        <!-- Apache CXF -->
        <dependency>
            <groupId>org.apache.cxf</groupId>
            <artifactId>cxf-spring-boot-starter-jaxws</artifactId>
            <version>${cxf.version}</version>
        </dependency>
        <!-- Spring Boot -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>org.apache.cxf</groupId>
                <artifactId>cxf-codegen-plugin</artifactId>
                <version>${cxf.version}</version>
                <executions>
                    <execution>
                        <id>generate-sources</id>
                        <phase>generate-sources</phase>
                        <configuration>
                            <sourceRoot>${project.build.directory}/generated/cxf</sourceRoot>
                            <wsdlOptions>
                                <wsdlOption>
                                    <wsdl>${project.basedir}/src/main/resources/wsdl/helloworld.wsdl</wsdl>
                                    <wsdlLocation>classpath:wsdl/helloworld.wsdl</wsdlLocation>
                                </wsdlOption>
                            </wsdlOptions>
                        </configuration>
                        <goals>
                            <goal>wsdl2java</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>
```

CXF includes a Maven `cxf-codegen-plugin plugin` which can [generate java artifacts from a WSDL file](http://cxf.apache.org/docs/maven-cxf-codegen-plugin-wsdl-to-java.html). In the above plugin configuration we're running the <var>'wsdl2java'</var> goal in the <var>'generate-sources'</var> phase. When executing following Maven command, CXF will generate artifacts in the <var>'&lt;sourceRoot&gt;'</var> directory that we have specified. 

``` plaintext
mvn generate-sources
```

After running above command you should be able to find back a number of auto generated classes among which the `HelloWorldPortType` interface in addition to `Person` and `Greeting` as shown below. 

<figure>
    <img src="{{ site.url }}/assets/images/cxf/hello-world-jaxb-generated-java-classes.png" alt="helloworld jaxb generated java classes">
</figure>

Next we create a `SpringCxfApplication` class. It contain a `main()` method that delegates to Spring Boot’s `SpringApplication` class by calling run. `SpringApplication` will bootstrap our application, starting Spring which will in turn start the auto-configured Tomcat web server.

The `@SpringBootApplication` is a convenience annotation that adds all of the following: `@Configuration`, `@EnableAutoConfiguration` and `@ComponentScan`. Checkout the [Spring Boot getting started guide](https://spring.io/guides/gs/spring-boot/) for more details.

``` java
package com.codenotfound;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringCxfApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCxfApplication.class, args);
    }
}
```

# Creating the Endpoint (Provider)

In order for the CXF framework to be able to process incoming SOAP request over HTTP, we need to setup a `CXFServlet`. In our previous [CXF SOAP Web Service tutorial]({{ site.url }}/2014/08/jaxws-cxf-contract-first-hello-world-webservice-tutorial.html) we did this by using a deployment descriptor file (<var>web.xml</var> file under the <var>WEB-INF</var> directory) or an alternative with Spring is to use a `ServletRegistrationBean`. In this example there is nothing to be done as the `cxf-spring-boot-starter-jaxws` automatically register the `CXFServlet` for us, great!

In this example we want the `CXFServlet` to listen for incoming requests on the following URI: "<kbd>/codenotfound/ws</kbd>", instead of the default value which is: "<kbd>/services/*</kbd>". This can be achieved by setting the <var>'cxf.path'</var> property in the <var>application.properties</var> file located under the <var>src/main/resources</var> folder.

``` properties
# server HTTP port
server.port=9090

# set the CXFServlet URL pattern
cxf.path=/codenotfound/ws

# hello world service address
helloworld.service.address=http://localhost:9090/codenotfound/ws/helloworld
```

Next we create a configuration file that contains the definition of our `Endpoint` at which our Hello World SOAP service will be exposed. The `Endpoint` gets created by passing the [CXF bus, which is the backbone of the CXF architecture](http://cxf.apache.org/docs/cxf-architecture.html#CXFArchitecture-Bus) that manages the respective inbound and outbound message and fault interceptor chains for all client and server endpoints. We use the default CXF bus and get a reference to it via Spring's `@Autowired` annotation.

In addition to the bus we also specify the `HelloWorldImpl` class which contains the actual implementation of the service. Finally we set the URI on which the endpoint will be exposed to "<kbd>/helloworld</kbd>". Together with the <var>'cxf.path'</var> configuration in the <var>application.properties</var> file this result into following URL that clients will need to call: [http://localhost:9090/codenotfound/ws/helloworld](http://localhost:9090/codenotfound/ws/helloworld).

``` java
package com.codenotfound.endpoint;

import javax.xml.ws.Endpoint;

import org.apache.cxf.Bus;
import org.apache.cxf.jaxws.EndpointImpl;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class EndpointConfig {

    @Autowired
    private Bus bus;

    @Bean
    public Endpoint endpoint() {
        EndpointImpl endpoint = new EndpointImpl(bus,
                new HelloWorldImpl());
        endpoint.publish("/helloworld");
        return endpoint;
    }
}
```

The service implementation is specified the `HelloWorldImpl` POJO that implements the `HelloWorldPortType` interface that was generated from the WSDL file earlier. We simply log the name of the received `Person` and then use it to construct a `Greeting` that is logged and then returned.

``` java
package com.codenotfound.endpoint;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.codenotfound.services.helloworld.HelloWorldPortType;
import com.codenotfound.types.helloworld.Greeting;
import com.codenotfound.types.helloworld.ObjectFactory;
import com.codenotfound.types.helloworld.Person;

public class HelloWorldImpl implements HelloWorldPortType {

    private static final Logger LOGGER = LoggerFactory
            .getLogger(HelloWorldImpl.class);

    @Override
    public Greeting sayHello(Person request) {
        LOGGER.info(
                "Endpoint received person=[firstName:{},lastName:{}]",
                request.getFirstName(), request.getLastName());

        String greeting = "Hello " + request.getFirstName() + " "
                + request.getLastName() + "!";

        ObjectFactory factory = new ObjectFactory();
        Greeting response = factory.createGreeting();
        response.setGreeting(greeting);

        LOGGER.info("Endpoint sending greeting=[{}]",
                response.getGreeting());
        return response;
    }
}
```

# Creating the Client (Consumer)

CXF provides a `JaxWsProxyFactoryBean` that will create a Web Service client for you which implements a specified service class.

Let's create a `ClientConfig` class with the `@Configuration` annotation which indicates that the class can be used by the Spring IoC container as a source of bean definitions. Next we create a `JaxWsProxyFactoryBean` and set `HelloWorldPortType` as service class.

The last thing we set is the endpoint at which the Hello World service is available. This endpoint is fetched from the <var>application.properties</var> file so it can easily be changed if needed.

> Don't forget to call the `create()` method on the `JaxWsProxyFactoryBean` in order to have the Factory create a JAX-WS proxy that we can then use to make remote invocations.

``` java
package com.codenotfound.client;

import org.apache.cxf.jaxws.JaxWsProxyFactoryBean;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import com.codenotfound.services.helloworld.HelloWorldPortType;

@Configuration
public class ClientConfig {

    @Value("${helloworld.service.address}")
    private String helloworldServiceAddress;

    @Bean(name = "helloWorldClientProxyBean")
    public HelloWorldPortType opportunityPortType() {
        JaxWsProxyFactoryBean jaxWsProxyFactoryBean = new JaxWsProxyFactoryBean();
        jaxWsProxyFactoryBean.setServiceClass(HelloWorldPortType.class);
        jaxWsProxyFactoryBean.setAddress(helloworldServiceAddress);

        return (HelloWorldPortType) jaxWsProxyFactoryBean.create();
    }

}
```

Below `HelloWorldClient` provides a convenience `sayHello()` method that will create a `Person` object based on a first and last name. It then uses the autowired `helloWorldClientProxyBean` to invoke the Web Service. The result is a `Greeting` that is logged and returned as a `String`.

Annotating our class with the `@Component` annotation will cause Spring to automatically import this bean into the container if automatic component scanning is enabled. This is the case as in the beginning we added the `@SpringBootApplication` annotation to the main `SpringWsApplication` class which is [equivalent](http://docs.spring.io/autorepo/docs/spring-boot/current/reference/html/using-boot-using-springbootapplication-annotation.html) to using `@ComponentScan`.

``` java
package com.codenotfound.client;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import com.codenotfound.services.helloworld.HelloWorldPortType;
import com.codenotfound.types.helloworld.Greeting;
import com.codenotfound.types.helloworld.ObjectFactory;
import com.codenotfound.types.helloworld.Person;

@Component
public class HelloWorldClient {

    private static final Logger LOGGER = LoggerFactory
            .getLogger(HelloWorldClient.class);

    @Autowired
    private HelloWorldPortType helloWorldClientProxyBean;

    public String sayHello(String firstName, String lastName) {

        ObjectFactory factory = new ObjectFactory();
        Person person = factory.createPerson();

        person.setFirstName(firstName);
        person.setLastName(lastName);

        LOGGER.info("Client sending person=[firstName:{},lastName:{}]",
                person.getFirstName(), person.getLastName());

        Greeting greeting = (Greeting) helloWorldClientProxyBean
                .sayHello(person);

        LOGGER.info("Client received greeting=[{}]",
                greeting.getGreeting());
        return greeting.getGreeting();
    }
}
```

# Testing the Client & Endpoint

In order to wrap up the tutorial we will create a basic test case that will use our Hello World client to send a request to the CXF endpoint that is being exposed on Spring Boot's embedded Tomcat.

The new `@RunWith` and `@SpringBootTest` testing annotations, [that are available as of Spring Boot 1.4](https://spring.io/blog/2016/04/15/testing-improvements-in-spring-boot-1-4#spring-boot-1-4-simplifications), are used to tell `JUnit` to run using Spring’s testing support and bootstrap with Spring Boot’s support.

By default the embedded HTTP server will be started on a random port. As we have defined the URL which the client needs to call with a specific port number, we need to set the `DEFINED_PORT` web environment. This will cause Spring to use the <var>'server.port'</var> property from the <var>application.properties</var> file instead of a random one.

``` java
package com.codenotfound;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.test.context.junit4.SpringRunner;

import com.codenotfound.client.HelloWorldClient;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = WebEnvironment.DEFINED_PORT)
public class SpringCxfApplicationTests {

    @Autowired
    private HelloWorldClient helloWorldClient;

    @Test
    public void testSayHello() {

        assertThat(helloWorldClient.sayHello("John", "Doe"))
                .isEqualTo("Hello John Doe!");
    }
}
```

In order to run the above test, open a command prompt in the projects root folder and execute following Maven command: 

``` plaintext
mvn test
```

Maven will download the needed dependencies, compile the code and run the unit test case during which Tomcat is started and a service call is made. If all went well the build is reported as a success: 

``` plaintext
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v1.4.1.RELEASE)

21:19:58.154 INFO  [main][SpringCxfApplicationTests] Starting SpringCxfApplicationTests on cnf-pc with PID 2656 (started by CodeNotFound in c:\code\st\jaxws-cxf\jaxws-cxf-helloworld-example)
21:19:58.154 DEBUG [main][SpringCxfApplicationTests] Running with Spring Boot v1.4.1.RELEASE, Spring v4.3.3.RELEASE
21:19:58.155 INFO  [main][SpringCxfApplicationTests] No active profile set, falling back to default profiles: default
21:19:58.179 INFO  [main][AnnotationConfigEmbeddedWebApplicationContext] Refreshing org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@626abbd0: startup date [Sun Oct 16 21:19:58 CEST 2016]; root of context hierarchy
21:19:59.140 INFO  [main][XmlBeanDefinitionReader] Loading XML bean definitions from class path resource [META-INF/cxf/cxf.xml]
21:20:00.037 INFO  [main][TomcatEmbeddedServletContainer] Tomcat initialized with port(s): 9090 (http)
21:20:00.176 INFO  [localhost-startStop-1][ContextLoader] Root WebApplicationContext: initialization completed in 1997 ms
21:20:00.369 INFO  [localhost-startStop-1][ServletRegistrationBean] Mapping servlet: 'dispatcherServlet' to [/]
21:20:00.371 INFO  [localhost-startStop-1][ServletRegistrationBean] Mapping servlet: 'CXFServlet' to [/codenotfound/ws/*]
21:20:00.375 INFO  [localhost-startStop-1][FilterRegistrationBean] Mapping filter: 'characterEncodingFilter' to: [/*]
21:20:00.375 INFO  [localhost-startStop-1][FilterRegistrationBean] Mapping filter: 'hiddenHttpMethodFilter' to: [/*]
21:20:00.375 INFO  [localhost-startStop-1][FilterRegistrationBean] Mapping filter: 'httpPutFormContentFilter' to: [/*]
21:20:00.376 INFO  [localhost-startStop-1][FilterRegistrationBean] Mapping filter: 'requestContextFilter' to: [/*]
21:20:01.367 INFO  [main][RequestMappingHandlerAdapter] Looking for @ControllerAdvice: org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@626abbd0: startup date [Sun Oct 16 21:19:58 CEST 2016]; root of context hierarchy
21:20:01.432 INFO  [main][RequestMappingHandlerMapping] Mapped "{[/error]}" onto public org.springframework.http.ResponseEntity<java.util.Map<java.lang.String, java.lang.Object>> org.springframework.boot.autoconfigure.web.BasicErrorController.error(javax.servlet.http.HttpServletRequest)
21:20:01.434 INFO  [main][RequestMappingHandlerMapping] Mapped "{[/error],produces=[text/html]}" onto public org.springframework.web.servlet.ModelAndView org.springframework.boot.autoconfigure.web.BasicErrorController.errorHtml(javax.servlet.http.HttpServletRequest,javax.servlet.http.HttpServletResponse)
21:20:01.463 INFO  [main][SimpleUrlHandlerMapping] Mapped URL path [/webjars/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
21:20:01.463 INFO  [main][SimpleUrlHandlerMapping] Mapped URL path [/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
21:20:01.499 INFO  [main][SimpleUrlHandlerMapping] Mapped URL path [/**/favicon.ico] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
21:20:01.654 INFO  [main][TomcatEmbeddedServletContainer] Tomcat started on port(s): 9090 (http)
21:20:01.659 INFO  [main][SpringCxfApplicationTests] Started SpringCxfApplicationTests in 3.963 seconds (JVM running for 4.717)
21:20:01.676 INFO  [main][HelloWorldClient] Client sending person=[firstName:John,lastName:Doe]
21:20:01.943 INFO  [http-nio-9090-exec-1][HelloWorldImpl] Endpoint received person=[firstName:John,lastName:Doe]
21:20:01.944 INFO  [http-nio-9090-exec-1][HelloWorldImpl] Endpoint sending greeting=[Hello John Doe!]
21:20:01.965 INFO  [main][HelloWorldClient] Client received greeting=[Hello John Doe!]
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 4.359 sec - in com.codenotfound.SpringCxfApplicationTests
21:20:02.007 INFO  [Thread-2][AnnotationConfigEmbeddedWebApplicationContext] Closing org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@626abbd0: startup date [Sun
 Oct 16 21:19:58 CEST 2016]; root of context hierarchy

Results :

Tests run: 1, Failures: 0, Errors: 0, Skipped: 0

[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 7.692 s
[INFO] Finished at: 2016-10-16T21:20:02+02:00
[INFO] Final Memory: 20M/227M
[INFO] ------------------------------------------------------------------------
```

If you just want to start Spring Boot so that the endpoint is up and running, execute following Maven commmand: 

``` powershell
mvn spring-boot:run
```

---

{% capture notice-github %}
![github mark](/assets/images/logos/github-mark.png){: .align-left}
If you would like to run the above code sample you can get the full source code [here](https://github.com/code-not-found/jaxws-cxf/tree/master/jaxws-cxf-helloworld-example).
{% endcapture %}
<div class="notice--info">{{ notice-github | markdownify }}</div>

This concludes our example on how to use CXF together with Spring Boot in order to consume and produce a Web Service starting from a WSDL file. Drop a line in case something was unclear or if you just liked the tutorial.