# Spring Boot and Redis Exercise

## Pre-Requisites

### Cloud Foundry Command Line Interface (CLI)

1. Install the Cloud Foundry CLI from this URL: https://docs.cloudfoundry.org/cf-cli/install-go-cli.html
1. Verify the install by opening a terminal or command window and typing `cf --version`. You should see a version string to match the version you installed

### Install an IDE

Install and configure a Java IDE you are comfortable with. Good options include:

- Eclipse: https://www.eclipse.org/
- IntelliJ: https://www.jetbrains.com/idea/
- Visual Studio Code: https://visualstudio.microsoft.com/

If you install Visual Studio Code, then add the following extensions:

- (Microsoft) Java Extension Pack
- (Pivotal) Spring Boot Extension Pack

### Obtain PCF Credentials

If you are using a private installation of PCF, then obtain credentials and API enpoint information from your PCF platform team. If you are using Pivotal Web Services (the public PCF instance hosted by Pivotal), then go to [https://run.pivotal.io/](https://run.pivotal.io/) and register for a free account.

Once you have credentials, login with the CLI...

1. Open a terminal or command window and login to PCF with the command `cf login -a api.run.pivotal.io` (or whatever API endpoint you are using if not Pivotal Web Services)
1. Enter the email you registered and the password you set

## Create the Basic Application

1. Navigate to [https://start.spring.io](https://start.spring.io)
1. Create a Maven project with Java and the latest version of Spring Boot (2.3.3 at the time of writing)
1. Specify group: `microservice.workshop`
1. Specify artifact: `redis-demo`
1. Specify packaging: Jar
1. Specify Java Version to match what you have installed
1. For dependencies, add the following:
   - Spring Web Starter
   - Spring Boot Actuator
   - Spring Data Redis
1. Generate the project (causes a download)
1. Unzip the downloaded file somewhere convenient
1. Add the new project to your IDE workspace
   - Eclipse: File->Import->Existing Maven Project
   - IntelliJ: File->New->Module From Existing Sources...
   - VS Code: File->Add Folder to Workspace (or just open the folder by navigating to it and entering the command `code .`)

## Configure The Info Actuator

1. Rename `application.properties` in `src/main/resources` to `application.yml`
1. Open `application.yml` in `src/main/resources`
1. Add this value

   ```yml
   info:
     app:
       name: Payment Service

   management:
     endpoint:
       health:
         show-details: always
   ```

1. Create a file called `application-default.yml` in `src/main/resources`
1. Set its content to the following:

   ```yml
   spring:
     autoconfigure:
       exclude:
         - org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
   ```

   This will tell SpringBoot not to configure Redis when we're running locally - even though Redis is on the classpath. Failure to do this will not stop the application from starting and running successfully. But the health actuator will show the application being down.

1. Create a file called `application-cloud.yml` in `src/main/resources`
1. Set its content to the following:

   ```yml
   spring:
     autoconfigure:
       exclude:
         - org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
   ```

   This will tell SpringBoot not to configure Redis when we're running in the cloud - even though Redis is on the classpath. We will remove this file once we're ready to work with Redis.

## Configure Swagger

1. Open `pom.xml`, add the following dependencies:

   ```xml
   <dependency>
     <groupId>io.springfox</groupId>
     <artifactId>springfox-boot-starter</artifactId>
     <version>3.0.0</version>
   </dependency>
   ```

1. Create a class `SwaggerConfiguration` in the `micoservice.workshop.redisdemo` package. Add the following:

   ```java
   package microservice.workshop.redisdemo;

   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;
   import org.springframework.stereotype.Controller;
   import org.springframework.web.bind.annotation.RequestMapping;
   import org.springframework.web.bind.annotation.RestController;
   import org.springframework.web.servlet.view.RedirectView;
   import springfox.documentation.builders.RequestHandlerSelectors;
   import springfox.documentation.spi.DocumentationType;
   import springfox.documentation.spring.web.plugins.Docket;
   import springfox.documentation.swagger2.annotations.EnableSwagger2;

   @Configuration
   @EnableSwagger2
   @Controller
   public class SwaggerConfiguration {

       @RequestMapping("/")
       public RedirectView redirectToSwagger() {
           return new RedirectView("swagger-ui/");
       }

       @Bean
       public Docket api() {
           return new Docket(DocumentationType.SWAGGER_2)
                   .select()
                   .apis(RequestHandlerSelectors.withClassAnnotation(RestController.class))
                   .build();
       }
   }
   ```

   This configuration does three important things:

   1. It enables Swagger
   1. It redirects the root URL to the Swagger UI. I find this convenient, but YMMV
   1. It tells Springfox that we only want to use Swagger for REST controllers. Without this there will be Swagger documentation for the redirect controller, as well as the basic Spring error controller and we usually don't want this.

## Create a Payment Service

1. Create a package `microservice.workshop.redisdemo.service`
1. Create a class in the new package called `PaymentService`
1. Set the content of `PaymentService` to the following:

   ```java
   package microservice.workshop.redisdemo.service;

   import java.math.BigDecimal;
   import java.math.RoundingMode;

   import org.springframework.stereotype.Service;

   @Service
   public class PaymentService {

       public BigDecimal calculate(double amount, double rate, int years) {
           if (rate == 0.0) {
               return calculateWithoutInterest(amount, years);
           } else {
               return calculateWithInterest(amount, rate, years);
           }
       }

       private BigDecimal calculateWithInterest(double amount, double rate, int years) {
           double monthlyRate = rate / 100.0 / 12.0;
           int numberOfPayments = years * 12;
           double payment = (monthlyRate * amount) / (1.0 - Math.pow(1.0 + monthlyRate, -numberOfPayments));
           return toMoney(payment);
       }

       private BigDecimal calculateWithoutInterest(double amount, int years) {
           int numberOfPayments = years * 12;
           return toMoney(amount / numberOfPayments);
       }

       private BigDecimal toMoney(double d) {
           BigDecimal bd = new BigDecimal(d);
           return bd.setScale(2, RoundingMode.HALF_UP);
       }
   }
   ```

## Create a Hit Counter Service

1. Create an interface in the `microservice.workshop.redisdemo.service` package called `HitCounterService`
1. Set the content of `HitCounterService` to the following:

   ```java
   package microservice.workshop.redisdemo.service;

   public interface HitCounterService {
       long incrementCounter();
       void resetCount();
   }
   ```

1. Create a class in the `microservice.workshop.redisdemo.service` package called `MemoryHitCounterService`
1. Set the content of `MemoryHitCounterService` to the following:

   ```java
   package microservice.workshop.redisdemo.service;

   import org.springframework.stereotype.Service;

   @Service
   public class MemoryHitCounterService implements HitCounterService {

       private long hitCount = 0;

       @Override
       public long incrementCounter() {
           return ++hitCount;
       }

       @Override
       public void resetCount() {
           hitCount = 0;
       }
   }
   ```

## Create a Crash Service

1. Create a class in the `microservice.workshop.redisdemo.service` package called `CrashService`
1. Set the content of `CrashService` to the following:

   ```java
   package microservice.workshop.redisdemo.service;

   import java.util.concurrent.Executors;
   import java.util.concurrent.ScheduledExecutorService;
   import java.util.concurrent.TimeUnit;

   import org.springframework.stereotype.Service;

   @Service
   public class CrashService {
       private ScheduledExecutorService executer = Executors.newScheduledThreadPool(1);

       // calls System.exit after a 2 second delay
       public void crashIt() {
           executer.schedule(() -> System.exit(22), 2000, TimeUnit.MILLISECONDS);
       }
   }
   ```

## Create a Return Model

1. Create a package `microservice.workshop.redisdemo.model`
1. Create a class in the new package called `CalculatedPayment`
1. Set the content of `CalculatedPayment` to the following:

   ```java
   package microservice.workshop.redisdemo.model;

   import java.math.BigDecimal;

   public class CalculatedPayment {
       private double amount;
       private double rate;
       private int years;
       private BigDecimal payment;
       private String instance;
       private Long count;

       // TODO: add getters and setters for all fields...
   }
   ```

## Create a REST Controller for the Payment Service

1. Create a package `microservice.workshop.redisdemo.http`
1. Create a class in the new package called `PaymentController`
1. Set the content of `PaymentController` to the following:

   ```java
   package microservice.workshop.redisdemo.http;

   import java.math.BigDecimal;

   import org.slf4j.Logger;
   import org.slf4j.LoggerFactory;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.beans.factory.annotation.Value;
   import org.springframework.web.bind.annotation.CrossOrigin;
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.RequestMapping;
   import org.springframework.web.bind.annotation.RequestParam;
   import org.springframework.web.bind.annotation.RestController;

   import microservice.workshop.redisdemo.model.CalculatedPayment;
   import microservice.workshop.redisdemo.service.HitCounterService;
   import microservice.workshop.redisdemo.service.PaymentService;

   @CrossOrigin(origins = "*")
   @RestController
   @RequestMapping("/payment")
   public class PaymentController {

       @Value("${cloud.application.instance_index:local}")
       private String instance;

       @Autowired
       private HitCounterService hitCounterService;

       @Autowired
       private PaymentService paymentService;

       private static final Logger logger = LoggerFactory.getLogger(PaymentController.class);

       @GetMapping()
       public CalculatedPayment calculatePayment(@RequestParam("amount") double amount, @RequestParam("rate") double rate,
               @RequestParam("years") int years) {

           BigDecimal payment = paymentService.calculate(amount, rate, years);

           logger.debug("Calculated payment of {} for input amount: {}, rate: {}, years: {}",
               payment, amount, rate, years);

           CalculatedPayment calculatedPayment = new CalculatedPayment();
           calculatedPayment.setAmount(amount);
           calculatedPayment.setRate(rate);
           calculatedPayment.setYears(years);
           calculatedPayment.setPayment(payment);
           calculatedPayment.setInstance(instance);
           calculatedPayment.setCount(hitCounterService.incrementCounter());

           return calculatedPayment;
       }
   }
   ```

## Create a REST Controller to Reset the Hit Count

This is needed for the unit tests - it will reset the hit counter to a known state for each test.

1. Create a class `ResetHitCounterController` in package `microservice.workshop.redisdemo.http`
1. Set the content of `ResetHitCounterController` to the following:

   ```java
   package microservice.workshop.redisdemo.http;

   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.web.bind.annotation.CrossOrigin;
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.RequestMapping;
   import org.springframework.web.bind.annotation.RestController;

   import microservice.workshop.redisdemo.service.HitCounterService;

   @CrossOrigin(origins = "*")
   @RestController
   @RequestMapping("/resetCount")
   public class ResetHitCounterController {

       @Autowired
       private HitCounterService hitCounterService;

       @GetMapping
       public void reset() {
           hitCounterService.resetCount();
       }
   }
   ```

## Create a REST Controller to Crash the Application

This is needed to demonstrate Cloud Foundriy's self-healing capabilities.

1. Create a class `CrashController` in package `microservice.workshop.redisdemo.http`
1. Set the content of `CrashController` to the following:

   ```java
   package microservice.workshop.redisdemo.http;

   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.web.bind.annotation.CrossOrigin;
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.RequestMapping;
   import org.springframework.web.bind.annotation.RestController;

   import io.swagger.annotations.ApiOperation;
   import microservice.workshop.redisdemo.service.CrashService;

   @CrossOrigin(origins = "*")
   @RestController
   @RequestMapping("/crash")
   public class CrashController {

       @Autowired
       private CrashService crashService;

       @ApiOperation("Warning! The application will crash 2 seconds after this method is called")
       @GetMapping()
       public String crashIt() {
           crashService.crashIt();
           return "OK";
       }
   }
   ```

## Unit Tests

1. Make a new package `microservice.workshop.redisdemo.http` in the `src/test/java` tree
1. Create a class in the new package called `PaymentControllerTest`
1. Set the content of `PaymentControllerTest` to the following:

   ```java
   package microservice.workshop.redisdemo.http;

   import static org.hamcrest.Matchers.*;
   import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
   import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
   import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
   import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
   import static org.springframework.test.web.servlet.setup.MockMvcBuilders.webAppContextSetup;

   import org.junit.jupiter.api.BeforeEach;
   import org.junit.jupiter.api.Test;
   import org.junit.jupiter.api.extension.ExtendWith;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.boot.test.context.SpringBootTest;
   import org.springframework.http.HttpStatus;
   import org.springframework.http.MediaType;
   import org.springframework.test.context.junit.jupiter.SpringExtension;
   import org.springframework.test.web.servlet.MockMvc;
   import org.springframework.web.context.WebApplicationContext;

   @ExtendWith(SpringExtension.class)
   @SpringBootTest
   public class PaymentControllerTest {
       private MockMvc mockMvc;

       @Autowired
       private WebApplicationContext webApplicationContext;

       @BeforeEach
       public void setup() {
           this.mockMvc = webAppContextSetup(webApplicationContext).build();
       }

       @Test
       public void testWithInterest() throws Exception {
           mockMvc.perform(get("/resetCount"))
           .andExpect(status().is(HttpStatus.OK.value()));

           mockMvc.perform(get("/payment?amount=100000&rate=3.5&years=30"))
           .andExpect(status().is(HttpStatus.OK.value()))
           .andExpect(content().contentType(MediaType.APPLICATION_JSON))
           .andExpect(jsonPath("$.payment", is(449.04)))
           .andExpect(jsonPath("$.count", is(1)));
       }

       @Test
       public void testZeroInterest() throws Exception {
           mockMvc.perform(get("/resetCount"))
           .andExpect(status().is(HttpStatus.OK.value()));

           mockMvc.perform(get("/payment?amount=100000&rate=0&years=30"))
           .andExpect(status().is(HttpStatus.OK.value()))
           .andExpect(content().contentType(MediaType.APPLICATION_JSON))
           .andExpect(jsonPath("$.payment", is(277.78)))
           .andExpect(jsonPath("$.count", is(1)));
       }

       @Test
       public void testThatHitCounterIncrements() throws Exception {
           mockMvc.perform(get("/resetCount"))
           .andExpect(status().is(HttpStatus.OK.value()));

           mockMvc.perform(get("/payment?amount=100000&rate=3.5&years=30"))
           .andExpect(status().is(HttpStatus.OK.value()))
           .andExpect(content().contentType(MediaType.APPLICATION_JSON))
           .andExpect(jsonPath("$.payment", is(449.04)))
           .andExpect(jsonPath("$.count", is(1)));

           mockMvc.perform(get("/payment?amount=100000&rate=0&years=30"))
           .andExpect(status().is(HttpStatus.OK.value()))
           .andExpect(content().contentType(MediaType.APPLICATION_JSON))
           .andExpect(jsonPath("$.payment", is(277.78)))
           .andExpect(jsonPath("$.count", is(2)));
       }
   }
   ```

## Testing

1. Run the unit tests:

   - (Windows Command Prompt) `mvnw clean test`
   - (Windows Powershell) `.\mvnw clean test`
   - (Mac/Linux) `./mvnw clean test`
   - Or your IDE's method of running tests

1. Start the application:

   - (Windows Command Prompt) `mvnw spring-boot:run`
   - (Windows Powershell) `.\mvnw spring-boot:run`
   - (Mac/Linux) `./mvnw spring-boot:run`
   - Or your IDE's method of running the main application class

1. Test Swagger [http://localhost:8080](http://localhost:8080)
1. Test the acuator health endpoint [http://localhost:8080/actuator/health](http://localhost:8080/actuator/health)
1. Test the acuator info endpoint [http://localhost:8080/actuator/info](http://localhost:8080/actuator/info)

## Deploy to Cloud Foundry

1. Create a file `manifest.yml` in the project root directory. Set it's contents to the following:

   ```yaml
   applications:
     - name: RedisDemo-1.0
       path: target/redis-demo-0.0.1-SNAPSHOT.jar
       random-route: true
       env:
         JBP_CONFIG_OPEN_JDK_JRE: "{jre: {version: 11.+}}"
   ```

   Note that this manifest specifies JRE 11+. This is not required if you are using Java 8 (the Java buildpack default). If you are using a different version of Java, you can change the environment variable accordingly.

1. Build the application JAR file:

   - (Windows Command Prompt) `mvnw clean package`
   - (Windows Powershell) `.\mvnw clean package`
   - (Mac/Linux) `./mvnw clean package`
   - Or your IDE's method of running the Maven build

1. `cf push`

You should now be able to test the app with Swagger at the route created by PCF. One thing to note is that the hit counter will reset everytime you deploy (because it is stored in the app state). Also, when the app is scaled the hit counter will not be shared across instances.

## Exercise the Application

### Inspect the Application With the Apps Manager UI

1. Login to Pivotal Apps Manager at https://run.pivotal.io/
1. Inspect the application...specifically:
   - On the app overview page, you should see the Spring Boot logo
   - On the app overview page you should be able to inspect details of the app health
   - On the logs page you should see the recent logs for the application, and be able to change logging levels
   - On the threads page you should be able to obtain a thread dump
   - On the settings page, there should be a Spring Info section. In that section you should be able to see the text you added to the info actuator by pressing the "View Raw JSON" button
   - On the settings page you should be able to see the environment variable we specified in the manifest file, as well as environment variable supplied by Cloud Foundry

### Inspect the Application With the CLI

Some interesting CLI commands...

- `cf target` shows information about your current connection, also used to change org/space if you have more than one
- `cf apps` will show all apps deployed in your current org/space
- `cf routes` will show all routes in your current org/space
- `cf services` will show all services allocated to your current org/space
- `cf app RedisDemo-1.0` will show detailed information about the application
- `cf stop RedisDemo-1.0` will start the app
- `cf start RedisDemo-1.0` will start the app
- `cf events RedisDemo-1.0` will sho recent events in the applications (starts, stops, etc.)
- `cf logs RedisDemo-1.0 --recent` will dump recent logging information to the screen
- `cf logs RedisDemo-1.0` will tail the application log
- `cf ssh-enabled RedisDemo-1.0` will report whether it is possible to SSH into the application container
- `cf ssh RedisDemo-1.0` will SSH into the application container

### Scale the Application

Applications can be scaled in two ways - through the app manager UI, or through the CLI. We will use the CLI.

1. Start the loan-calculator-client web page by using this URL: https://jeffgbutler.github.io/payment-calculator-client/
1. Enter the URL to your application (like https://redisdemo-10-shy-zebra.cfapps.io) in the Base URL textbox
1. Press the "Start" button. You should see random traffic being generated to the application. You should also see the all traffic is routed to instance 0
1. Scale the app by entering the command `cf scale RedisDemo-1.0 -i 2` - this will request two instances of the app running. Eventually you should see traffic being routed to the two app instances. Notice that the hit count is not consistent. Why?
1. If you press the "Crash It!" button on the client page, then one of the app instances will crash. Which one depends on how the request was routed. Cloud Foundry will notice that an instance has crashed and will start a new instance automatically - this may take a few minutes to show
1. You can scale the app back down by entering `cf scale RedisDemo-1.0 -i 1`

You should have noticed that the hit counter is not consistent among the instances, and that it is reset when an app instance crashes. This will demonstrate the idea of epehemeral containers and that Cloud Foundry is designed for stateless applications. We will store the hit count in an external Redis cache in the next section to correct this issue.

## Configure Redis on Cloud Foundry

### Create a Redis Cache Instance

1. Login to Pivotal Apps Manager at https://run.pivotal.io/
1. Navigate to your org/space
1. Select the "services" tab
1. Press the "Add a Service" button
1. Create a new service...
   - Select "Redis Cache"
   - Select plan type "30 MB" - the free plan
   - Set the instance name to "xxxredis" where "xxx" are your initials

### Add a Redis Based Hit Counter

1. Create a new class `RedisHitCounterService` in the `microservice.workshop.redisdemo.service` package
1. Set the contents of `RedisHitCounterService` to the following:

   ```java
   package microservice.workshop.redisdemo.service;

   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.context.annotation.Profile;
   import org.springframework.data.redis.core.RedisTemplate;
   import org.springframework.stereotype.Service;

   @Service
   @Profile("cloud")
   public class RedisHitCounterService implements HitCounterService {

       private static final String REDIS_KEY = "payment-calculator";
       private static final int DEFAULT_VALUE = 5000;

       @Autowired
       private RedisTemplate<String, Integer> redisTemplate;

       @Override
       public long incrementCounter() {
           redisTemplate.opsForValue().setIfAbsent(REDIS_KEY, DEFAULT_VALUE);
           return redisTemplate.opsForValue().increment(REDIS_KEY);
       }

       @Override
       public void resetCount() {
           redisTemplate.opsForValue().set(REDIS_KEY, DEFAULT_VALUE);
       }
   }
   ```

1. Create a class `CloudConfiguration` in the `microservice.workshop.redisdemo` package. Add the following:

   ```java
   package microservice.workshop.redisdemo;

   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;
   import org.springframework.context.annotation.Profile;
   import org.springframework.data.redis.connection.RedisConnectionFactory;
   import org.springframework.data.redis.core.RedisTemplate;
   import org.springframework.data.redis.serializer.GenericToStringSerializer;

   @Configuration
   @Profile("cloud")
   public class CloudConfiguration {
       @Bean
       public RedisTemplate<String, Integer> redisTemplate(RedisConnectionFactory redisFactory) {
           RedisTemplate<String, Integer> template = new RedisTemplate<>();
           template.setConnectionFactory(redisFactory);
           template.setValueSerializer(new GenericToStringSerializer<>(Integer.class));
           return template;
       }
   }
   ```

   This configuration is enabled when the "cloud" profile is enabled only. On Cloud Foundry, the Java build pack enables this profile. When enabled, this configuration will create a Redis connection based on the Redis instance bound to the application.

1. Open `MemoryHitCounterService.java` in the `microservice.workshop.redisdemo.service` package
1. Change the service so that it is only active when not on the cloud:

   ```java
   @Service
   @Profile("!cloud")
   public class MemoryHitCounterService implements HitCounterService {
       ...
   }
   ```

1. Modify the file `manifest.yml` in the project root directory. Set it's contents to the following:

   ```yaml
   applications:
     - name: RedisDemo-1.1
       path: target/redis-demo-0.0.1-SNAPSHOT.jar
       random-route: true
       env:
         JBP_CONFIG_OPEN_JDK_JRE: "{jre: {version: 11.+}}"
       services:
         - xxxredis
   ```

   Where "xxxredis" maps to the name of the redis instance you created above.

   **Important Note:** you should also change the name of the application to denote the new version.

1. Delete the file `application-cloud.yml` in `src/main/resources`

1. Build the application:

   - (Windows Command Prompt) `mvnw clean package`
   - (Windows Powershell) `.\mvnw clean package`
   - (Mac/Linux) `./mvnw clean package`
   - Or your IDE's method of running the Maven build

1. `cf push`

You should now be able to test the app with Swagger at the route created by PCF. The hit counter will now persist across deploymant and will be consistent as the app scales.

## Blue Green Deployments

1. Run `cf target` to determine your space name.
1. `cf create-route <<your_space_name>> cfapps.io --hostname xxx-payment-calculator`
1. `cf map-route RedisDemo-1.0 cfapps.io --hostname xxx-payment-calculator`

Change the URL in the client page to the new route you've created (https://jgb-payment-calculator.cfapps.io). You should see traffic being routed to your app.

Now route traffic to the 1.1 version of the app:

- `cf map-route RedisDemo-1.1 cfapps.io --hostname xxx-payment-calculator`

You should now see traffic being routed to the 1.0 and the 1.1 version of the application.

Now take away the route to the 1.0 version of the app:

- `cf unmap-route RedisDemo-1.0 cfapps.io --hostname xxx-payment-calculator`
