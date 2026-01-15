# Cheat-Sheet


## Table of Contents
* [Applicatipn Properties](#application-properties)
* [Kafka setup](#kafka-setup)
* [Local Eureka Service](#local-eureka-service)
* [Feign](#feign)
* [Grpc](#grpc)
* [Gatewat](#gateway)
* [Global Exception Handler](#global-exception-handler)
* [Jwt](#jwt)
* [Security](#security)

## Application properties

* Postgresql connection


```properties
spring.datasource.url=${DB_URL:jdbc:postgresql://localhost:5433/users}
spring.datasource.username=${DB_USERNAME:postgres}
spring.datasource.password=${DB_PASSWORD:postgres}
spring.datasource.driver-class-name=org.postgresql.Driver

spring.jpa.database=postgresql
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.open-in-view=false
```

* Eureka Server


```properties
eureka.instance.hostname=localhost
eureka.instance.prefer-ip-address=true
eureka.client.fetch-registry=false
eureka.client.register-with-eureka=false
eureka.server.enable-self-preservation=false
```

## Kafka setup

Latest used kafka docker image : confluentinc/cp-kafka:latest

* Env variables

```bash
KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,EXTERNAL://localhost:9094;
KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER;
KAFKA_CONTROLLER_QUORUM_VOTERS=0@kafka:9093;
KAFKA_DEFAULT_REPLICATION_FACTOR=1;
KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT,PLAINTEXT:PLAINTEXT;
KAFKA_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093,EXTERNAL://:9094;
KAFKA_MIN_INSYNC_REPLICAS=1;
KAFKA_NODE_ID=0;
KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1;
KAFKA_PROCESS_ROLES=controller,broker;
KAFKA_TRANSACTION_STATE_LOG_MIN_ISR=1;
KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=1
```

* Kafka Producer sample config class

```java
import lombok.RequiredArgsConstructor;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.ByteArraySerializer;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.DefaultKafkaProducerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.core.ProducerFactory;

import java.util.HashMap;
import java.util.Map;

@Configuration
@RequiredArgsConstructor
public class KafkaConfig {

    @Value("${spring.kafka.bootstrap.servers}")
    private String bootstrapServers;

    private Map<String, Object> senderProps() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
//        props.put(ProducerConfig.LINGER_MS_CONFIG, 10);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class);
        return props;
    }

    @Bean
    public ProducerFactory<String, byte[]> producerFactory() {
        return new DefaultKafkaProducerFactory<>(senderProps());
    }

    @Bean
    public KafkaTemplate<String, byte[]> kafkaTemplate(ProducerFactory<String, byte[]> producerFactory) {
        return new KafkaTemplate<>(producerFactory);
    }

}
```

* Kafka Producer sample class


```java
import com.farid.Entity.UserEntity;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import user.events.UserEvent;

@Slf4j
@Service
@RequiredArgsConstructor
public class KafkaProducer {

    private final KafkaTemplate<String, byte[]> kafkaTemplate;

    public void sendEvent(UserEntity user) {
        UserEvent event = UserEvent.newBuilder()
                .setId(user.getId())
                .setMail(user.getMail())
                .setUsername(user.getUsername())
                .setEventType("ACCOUNT_CREATED")
                .build();
        try {
            kafkaTemplate.send("user", event.toByteArray());
        }catch (Exception e) {
            log.error("Error sending ACCOUNT_CREATED event: {}",event);
        }
    }

}
```

* Kafka Consumer sample config class


```java
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.ByteArrayDeserializer;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.config.KafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.listener.ConcurrentMessageListenerContainer;

import java.util.HashMap;
import java.util.Map;

@Configuration
@EnableKafka
public class KafkaConfig {

    @Value("${spring.kafka.bootstrap.servers}")
    private String bootstrapServers;


    @Bean
    public Map<String, Object> consumerProps() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        configProps.put(ConsumerConfig.GROUP_ID_CONFIG, "user-consumer-group-v2");
        configProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        configProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ByteArrayDeserializer.class);
        configProps.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);
        configProps.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        return configProps;
    }

    @Bean
    public ConsumerFactory<String, byte[]> consumerFactory() {
        return new DefaultKafkaConsumerFactory<>(consumerProps());
    }

    @Bean
    public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, byte[]>> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, byte[]> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());

        return factory;
    }



}
```

* Kafka Consumer sample class


```java
import com.google.protobuf.InvalidProtocolBufferException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;
import user.events.UserEvent;

@Service
public class KafkaConsumer {

    private static final Logger log = LoggerFactory.getLogger(
            KafkaConsumer.class);

    @KafkaListener(topics="user", groupId = "user-consumer-group-v2")
    public void consumeEvent(byte[] event) {
        try {
            UserEvent patientEvent = UserEvent.parseFrom(event);
            // ... perform any business related to analytics here

            log.info("Received Patient Event: [PatientId={},PatientName={},PatientEmail={}]",
                    patientEvent.getId(),
                    patientEvent.getUsername(),
                    patientEvent.getMail());
        } catch (InvalidProtocolBufferException e) {
            log.error("Error deserializing event {}", e.getMessage());
        }
    }
}
```

## Local Eureka Service

To create Eureka service you need to use `@EnableEurekaServer` annotation on top of main method in spring-boot application.
Note that if the default port for eureka server isn't changed, eureka client only need to eureka client dependency and the `spring.application.name` for it to connect to the eureka server.

## Feign

Feign is the tool that used to communicate between microsrevices via http.

* Feign setup

The name of the service is stated inside of the `@FeignClient` annotation, and the desired endpoints are initialized inside of interface without body.
```java
import com.farid.DTO.Request.SignInCheckRequest;
import com.farid.DTO.Request.SignInRequest;
import com.farid.DTO.Request.SignUpRequest;
import com.farid.DTO.UserEntity;
import jakarta.validation.Valid;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@FeignClient("USER-MICROSERVICE")
public interface UsersInterface {

    @PostMapping("/users/v1/get")
    UserEntity loadUserByUsername(@RequestParam("username") String username);

    @PostMapping("/users/v1/registration/check")
    Boolean registrationCheck(@RequestBody @Valid SignInCheckRequest signInCheckRequest);

    @PostMapping("/users/v1/signIn")
    Boolean isCredentialsCorrect(@RequestBody @Valid SignInRequest signInRequest);

    @PostMapping("/users/v1/signUp")
    ResponseEntity<String> signUp(@RequestBody @Valid SignUpRequest signUpRequest);

    @GetMapping("/users/v1/test")
    ResponseEntity<String> test();

}
```

## Grpc

* Grpc dependencies

```xml

<properties>
        <grpc.version>1.76.0</grpc.version>
        <protobuf-java.version>4.33.0</protobuf-java.version>
        <spring-grpc.version>0.12.0</spring-grpc.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>net.devh</groupId>
            <artifactId>grpc-server-spring-boot-starter</artifactId>
            <version>3.1.0.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>com.google.protobuf</groupId>
            <artifactId>protobuf-java</artifactId>
            <version>${protobuf-java.version}</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-netty-shaded</artifactId>
            <version>${grpc.version}</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-stub</artifactId>
            <version>${grpc.version}</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-services</artifactId>
            <version>${grpc.version}</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-protobuf</artifactId>
            <version>${grpc.version}</version>
        </dependency>
    </dependencies>


<dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.grpc</groupId>
                <artifactId>spring-grpc-dependencies</artifactId>
                <version>${spring-grpc.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
    </dependencyManagement>


<build>
        <plugins>
            <!-- Spring boot / maven  -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>io.github.ascopes</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>3.8.0</version>
                <configuration>
                    <protocVersion>${protobuf-java.version}</protocVersion>
                    <binaryMavenPlugins>
                        <binaryMavenPlugin>
                            <groupId>io.grpc</groupId>
                            <artifactId>protoc-gen-grpc-java</artifactId>
                            <version>${grpc.version}</version>
                            <options>@generated=omit</options>
                        </binaryMavenPlugin>
                    </binaryMavenPlugins>
                </configuration>
                <executions>
                    <execution>
                        <id>generate</id>
                        <goals>
                            <goal>generate</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

        </plugins>
    </build>
```

* Grpc sample proto file

In order to generate code from proto file we need to clean+compile the project.
```proto
syntax="proto3";

option java_package="users";
option java_multiple_files=true;


service userService {
  rpc loadUserByUsername(username) returns (UserEntity);
  rpc loadUserById(id) returns (UserEntity);
  rpc deleteUserByUsername(username) returns (Empty);
  rpc signUp(SignUpRequest) returns (UserModel);
  rpc isCredentialsCorrect(SignInRequest) returns (decision);
  rpc registrationCheck(SignInCheckRequest) returns (decision);
}

enum ROLE {
  USER = 0;
  ADMIN = 1;
  SUPERADMIN = 2;
}

message Empty {

}

message id{
  int64 id = 1;
}

message username{
  string username = 1;
}

message decision{
  bool decision = 1;
}

message UserEntity{
  int64 id = 1;
  string mail = 2;
  string username = 3;
  string password = 4;
  ROLE role = 5;
}

message SignUpRequest{
  string mail = 1;
  string username = 2;
  string password = 3;
}

message SignInRequest{
  string username = 1;
  string password = 2;
}

message SignInCheckRequest{
  string username = 1;
  string mail = 2;
}

message UserModel{
  string mail = 1;
  string username = 2;
  ROLE role = 3;

}
```

* Grpc service sample class

```java
import com.farid.Mapper.DtoToGrpcRequest;
import io.grpc.stub.StreamObserver;
import lombok.RequiredArgsConstructor;
import net.devh.boot.grpc.server.service.GrpcService;
import users.*;

@GrpcService
@RequiredArgsConstructor
public class UserGrpcService extends userServiceGrpc.userServiceImplBase {

    private final UserService userService;


    @Override
    public void loadUserByUsername(username request, StreamObserver<UserEntity> responseObserver) {
        com.farid.Entity.UserEntity user = userService.loadUserByUsername(request.getUsername());

        UserEntity response = UserEntity.newBuilder()
                .setId(user.getId())
                .setUsername(user.getUsername())
                .setPassword(user.getPassword())
                .setMail(user.getMail())
                .setRole(ROLE.valueOf(user.getRole().name()))
                .build();

        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
}

```

* Grpc client sample class

```java
import com.farid.DTO.UserEntity;
import com.farid.Mapper.GrpcResponseToDTO;
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import users.userServiceGrpc;

@Service
public class UserGrpcServiceClient {

    private final userServiceGrpc.userServiceBlockingStub blockingStub;

    public UserGrpcServiceClient(@Value("${user.service.address:localhost}") String serverAddress, @Value("${user.service.grpc.port:9095}") int serverPort) {
        ManagedChannel channel = ManagedChannelBuilder.forAddress(serverAddress, serverPort).usePlaintext().build();

        blockingStub = userServiceGrpc.newBlockingStub(channel);
    }

    public UserEntity loadUserByUsername(String username){
        return GrpcResponseToDTO.toUserEntity(blockingStub.loadUserByUsername(users.username.newBuilder()
                .setUsername(username)
                .build()));
    }

}
```

## Gateway

* Gateway config

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.client.WebClient;

@Configuration
public class GatewayConfig {

    @Bean
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }

}
```

* Jwt validation gatewat filter

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.WebClient;


@Component
public class JwtValidationGatewayFilterFactory extends AbstractGatewayFilterFactory<Object> {

    private final WebClient webClient;

    public JwtValidationGatewayFilterFactory(WebClient.Builder webClientBuilder, @Value("${auth.service.url:http://authentication-service:8085/auth}") String authServiceUrl) {
        this.webClient = webClientBuilder.baseUrl(authServiceUrl).build();
    }

    @Override
    public GatewayFilter apply(Object config) {
        return (exchange, chain) -> {
            String token = exchange.getRequest().getHeaders().getFirst(HttpHeaders.AUTHORIZATION);

            if (token == null || !token.startsWith("Bearer ")) {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }


            return webClient.get()
                    .uri("/validate")
                    .header(HttpHeaders.AUTHORIZATION,token)
                    .retrieve()
                    .toEntity(Void.class)
                    .flatMap(validationResponse -> {

                        String userSubject = validationResponse.getHeaders().getFirst("X-Validated-User-Subject");

                        if (userSubject == null) {
                            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                            return exchange.getResponse().setComplete();
                        }

                        ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
                                .header("X-User", userSubject)
                                .build();

                        return chain.filter(exchange.mutate().request(mutatedRequest).build());
                    })
                    .onErrorResume(e -> {
                        System.err.println("JWT Validation Failed. Error: " + e.getMessage());
                        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                        return exchange.getResponse().setComplete();
                    });
        };
    }
}
```

* Gateway routing sample application.yml

```yml
spring:
  application:
    name: Gateway-Service
  cloud:
    gateway:
      server:
        webflux:
          discovery:
            locator:
              enabled:
                true
          routes:
            - id: auth-service-route
              uri: http://authentication-service:8085
              predicates:
                - Path=/auth/**
              filters:
#                - RewritePath=/auth/?(?<segment>.*), /auth/v1/$\{segment}

            - id: api-docs-auth-service-route
              uri: http://authentication-service:8085
              predicates:
                - Path=/api-docs/auth
              filters:
                - RewritePath=/api-docs/auth, /v3/api-docs

            - id: user-service-route
              uri: http://user-service:8095
              predicates:
                - Path=/users/**
              filters:
#                - RewritePath=/users/?(?<segment>.*), /users/v1/$\{segment}
                - JwtValidation

            - id: exercise-service-route
              uri: http://exercise-service:8105
              predicates:
                - Path=/exercise/**
              filters:
#                - RewritePath=/exercise/?(?<segment>.*), /exercise/v1/$\{segment}
                - JwtValidation

            - id: api-docs-exercise-service-route
              uri: http://exercise-service:8105
              predicates:
                - Path=/api-docs/exercise
              filters:
                - RewritePath=/api-docs/exercise, /v3/api-docs
```


## Global Exception Handler

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ExerciseNotFoundException.class)
    public ResponseEntity<Map<String, String>> handleExerciseNotFound(ExerciseNotFoundException ex) {
        Map<String, String> body = new HashMap<>();
        body.put("error", "NOT_FOUND");
        body.put("message", ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(body);
    }
}

```

## JWT

* Jwt service sample class


```java
import com.farid.DTO.TokenPair;
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.JwtException;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.io.Decoders;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Service;

import javax.crypto.SecretKey;
import java.util.*;

@Service
public class JwtService {

    @Value("${jwt.app.secret}")
    private String secret;

    @Value("${jwt.app.access.token.lifetime}")
    private int accessTokenLifetime;

    @Value("${jwt.app.refresh.token.lifetime}")
    private int refreshTokenLifetime;

    public TokenPair generateTokenPair(Authentication auth) {
        String accessToken = generateAccessToken(auth);
        String refreshToken = generateRefreshToken(auth);
        return new TokenPair(accessToken, refreshToken);
    }

    public String generateAccessToken(Authentication auth) {
        return generateToken(auth,accessTokenLifetime,new HashMap<>());
    }

    public String generateRefreshToken(Authentication auth) {
        HashMap<String,String> claims = new HashMap<>();
        claims.put("tokenType","refresh");
        return generateToken(auth,refreshTokenLifetime,claims);
    }

    private String generateToken(Authentication auth, long lifetime, Map<String,String> claims) {
        UserDetailsImpl userDetails = (UserDetailsImpl) auth.getPrincipal();
        return Jwts.builder()
                .header()
                .add("typ","JWT")
                .and()
                .subject(userDetails.getId() + "::" + userDetails.getRole() + "::" + userDetails.getUsername())
                .claims(claims)
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + lifetime))
                .signWith(getSignInKey())
                .compact();
    }

    public String getSubjectFromJwt(String token) {
        Claims claims = extractAllClaims(token);

        if(claims != null) {
            return claims.getSubject();
        }
        return null;
    }

    public boolean validateTokenForUser(String token, UserDetails userDetails) {
        String username = getSubjectFromJwt(token).split("::")[2];
        return username != null
                && username.equals(userDetails.getUsername());
    }

    public boolean isValidToken(String token) {
        return extractAllClaims(token) != null;
    }

    public boolean isRefreshToken(String token) {
        Claims claims = extractAllClaims(token);
        if(claims != null) {
            return "refresh".equals(claims.get("tokenType"));
        }
        return false;
    }

    private Claims extractAllClaims(String token) {
        Claims claims = null;

        try {
            claims = Jwts.parser()
                    .verifyWith(getSignInKey())
                    .build()
                    .parseSignedClaims(token)
                    .getPayload();
        } catch (JwtException | IllegalArgumentException e) {
            throw new RuntimeException(e);
        }

        return claims;
    }

    private SecretKey getSignInKey() {
        byte [] keyBytes = Decoders.BASE64.decode(secret);
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

* Jwt user details sample class


```java
import com.farid.DTO.ROLE;
import com.farid.DTO.UserEntity;
import lombok.AllArgsConstructor;
import lombok.Data;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.ArrayList;
import java.util.Collection;
import java.util.List;

@Data
@AllArgsConstructor
public class UserDetailsImpl implements UserDetails {

    private Long id;
    private String username;
    private String mail;
    private String password;
    private ROLE role;


    public static UserDetailsImpl build(UserEntity user){
        return new UserDetailsImpl(
                user.getId(),
                user.getUsername(),
                user.getMail(),
                user.getPassword(),
                user.getRole()
        );
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        List<GrantedAuthority> list = new ArrayList<>();

        list.add(new SimpleGrantedAuthority(role.toString()));

        return list;
    }

    @Override
    public String getPassword() {
        return password;
    }

    @Override
    public String getUsername() {
        return username;
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}
```

* Jwt user details service sample class


```java
//import com.farid.Feign.UsersInterface;
import com.farid.Jwt.UserDetailsImpl;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class UserDetailsServiceImpl implements UserDetailsService {

    private final UserGrpcServiceClient userGrpcServiceClient;
//    private final UsersInterface usersInterface;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
//        return UserDetailsImpl.build(userGrpcServiceClient.loadUserByUsername(username));
        return UserDetailsImpl.build(userGrpcServiceClient.loadUserByUsername(username));
    }
}
```


## Security

* Simple security config


```java
import com.farid.Service.UserDetailsServiceImpl;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpStatus;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.HttpStatusEntryPoint;
import org.springframework.web.cors.CorsConfiguration;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfiguration {

    private final UserDetailsServiceImpl userDetailsService;

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationProvider AuthenticationProvider() {
        DaoAuthenticationProvider authenticationProvider = new DaoAuthenticationProvider(userDetailsService);
        authenticationProvider.setPasswordEncoder(passwordEncoder());
        return authenticationProvider;
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration authenticationConfiguration) throws Exception {
        return authenticationConfiguration.getAuthenticationManager();
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
                .csrf(AbstractHttpConfigurer::disable)
                .cors(httpSecurityCorsConfigurer ->
                        httpSecurityCorsConfigurer.configurationSource(request ->
                                new CorsConfiguration().applyPermitDefaultValues()))
                .exceptionHandling(exceptions -> exceptions
                        .authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED)))
                .sessionManagement(session -> session
                        .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(authorize -> authorize
                        .anyRequest().permitAll()
                );

        return http.build();
    }

}
```
