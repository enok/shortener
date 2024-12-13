# URL Shortener Application

The **URL Shortener Application** is a robust, secure, and scalable system designed to shorten URLs, manage redirects, and deliver insightful usage analytics. Built with **Java 21** and **Spring Boot 3**, it leverages modern AWS cloud-native services to ensure high availability, performance, and security.

---
<br>

## **Table of Contents**

1. [Introduction](#1-introduction)  
2. [Architecture Overview](#2-architecture-overview)  
3. [Design Patterns](#3-design-patterns)  
4. [Code Implementation](#4-code-implementation)  
   - [4.1 Dependencies (`pom.xml`)](#41-dependencies-pomxml)  
   - [4.2 Configuration Properties (`application.yml`)](#42-configuration-properties-applicationyml)  
   - [4.3 Secrets Management (`SecretsService.java`)](#43-secrets-management-secretsservicejava)  
   - [4.4 URL Shortener Service (`UrlShortenerService.java`)](#44-url-shortener-service-urlshortenerservicejava)  
   - [4.5 Redis Configuration (`RedisConfig.java`)](#45-redis-configuration-redisconfigjava)  
   - [4.6 Controller Class (`UrlShortenerController.java`)](#46-controller-class-urlshortenercontrollerjava)  
5. [Infrastructure as Code (Terraform)](#5-infrastructure-as-code-terraform)  
   - [5.1 Terraform Folder Structure](#51-terraform-folder-structure)  
   - [5.2 VPC Configuration + Endpoints](#52-vpc-configuration--endpoints)  
   - [5.3 Network Configuration](#53-network-configuration)  
   - [5.4 SSL Certificates with ACM](#54-ssl-certificates-with-acm)  
   - [5.5 Application Load Balancer (ALB)](#55-application-load-balancer-alb)  
   - [5.6 Domain Configuration](#56-domain-configuration)  
   - [5.7 AWS WAF Configuration](#57-aws-waf-configuration)  
   - [5.8 API Gateway](#58-api-gateway)  
   - [5.9 AWS Lambda Function](#59-aws-lambda-function)  
   - [5.10 DynamoDB Table](#510-dynamodb-table)  
   - [5.11 Redis (ElastiCache)](#511-redis-elasticache)  
   - [5.12 AWS Secrets Manager](#512-aws-secrets-manager)  
   - [5.13 AWS CloudFront Configuration](#513-aws-cloudfront-configuration)  
6. [CI/CD Pipeline](#6-cicd-pipeline)  
7. [Steps to Run the Project](#7-steps-to-run-the-project)  
8. [Roadmap](#8-roadmap)  
9. [Architecture Diagrams](#9-architecture-diagrams)  
10. [Conclusion](#10-conclusion)  

---
<br>

## 1. Introduction

The **URL Shortener Application** allows users to generate short URLs for long links, enabling seamless redirection while providing analytics and scalability. Key highlights include:

- **Shorten URLs** and manage redirects.
- **Caching**: Redis (AWS ElastiCache) ensures low-latency lookups.
- **Persistence**: DynamoDB securely stores the URL mappings.
- **Authentication**: Secures endpoints using OAuth2 (Google).
- **Automation**: CI/CD pipeline with GitHub Actions and Terraform.
- **ETL and Analytics**: AWS Glue and OpenSearch for metadata and usage statistics.

This document outlines the application's **architecture**, key **design patterns**, **code implementation**, **deployment infrastructure**, and instructions for running the project locally or on AWS.

---
<br>

## 2. Architecture Overview

The **URL Shortener Application** follows a cloud-native, serverless architecture using AWS services. This design ensures scalability, high availability, and low operational overhead.

---

### 2.1 High-Level Architecture

1. **API Gateway**: Entry point for all HTTP requests. It routes incoming requests to AWS Lambda.
2. **AWS Lambda**: Processes business logic, such as shortening URLs and managing redirects.
3. **Redis (AWS ElastiCache)**: Provides low-latency caching for frequently accessed URL mappings.
4. **DynamoDB**: Persistent NoSQL database for storing the mapping between short and long URLs.
5. **AWS Secrets Manager**: Manages secure storage and retrieval of sensitive credentials.
6. **AWS WAF**: Adds rate limiting and protects APIs against malicious traffic.
7. **CloudFront**: Caches API responses globally to reduce latency and improve performance.
8. **AWS Glue**: Extracts metadata for ETL processing and stores results in S3.
9. **AWS OpenSearch**: Provides search and visualization of usage analytics.
10. **AWS CloudWatch**: Monitors system health, logs, and metrics to provide observability.

---

### 2.2 Data Flow

The following outlines how data flows through the system:

1. **Request Handling**:
   - A client sends a `POST` or `GET` request to the **API Gateway**.
   - **API Gateway** routes the request to the appropriate **AWS Lambda** function.

2. **Cache Lookup**:
   - For retrieval operations, **AWS Lambda** first checks **Redis (ElastiCache)** for the shortened URL.
   - If the URL exists in Redis, it is returned immediately.

3. **Database Lookup**:
   - If the URL is not found in Redis, Lambda fetches it from **DynamoDB** and updates Redis for future requests.

4. **Short URL Creation**:
   - For a `POST` request, a short URL is generated, stored in **DynamoDB**, and cached in **Redis**.

5. **ETL Processing**:
   - **AWS Glue** periodically extracts metadata (e.g., headers) from the original URLs.
   - Processed data is stored in **Amazon S3**.

6. **Analytics**:
   - Metadata and usage data are indexed in **AWS OpenSearch** for visualization and reporting.

7. **Monitoring**:
   - **CloudWatch** logs all Lambda invocations, errors, and performance metrics.
   - Alerts are configured for failures, high latency, or rate limiting breaches.

---

### 2.3 Diagram

For a visual representation of the architecture:

- **Use draw.io or Lucidchart** to create the diagram.
- Key components to include:
  - API Gateway → Lambda → Redis → DynamoDB
  - Secrets Manager, WAF, CloudFront for supporting infrastructure.
  - AWS Glue → S3 → OpenSearch for ETL pipelines and analytics.

![alt text](image.png)

---

### Summary

The architecture combines serverless computing with AWS managed services to deliver:

1. **Scalability**: Handles increasing workloads using Lambda and DynamoDB.
2. **Performance**: Redis caching and CloudFront reduce response times.
3. **Security**: WAF protects APIs; Secrets Manager manages credentials.
4. **Observability**: CloudWatch logs, metrics, and alarms provide full visibility.
5. **Analytics**: AWS Glue and OpenSearch process and visualize usage data.

---
<br>

## 3. Design Patterns

The application follows industry-standard **design patterns** to ensure modularity, scalability, and clean code architecture.

---

### 3.1 Singleton Pattern

Ensures that only one instance of shared resources, such as clients and connections, is created and reused throughout the application.

- **Use Case**: 
  - Redis connection (`RedisTemplate`).
  - DynamoDB client.
- **Benefit**: Reduces overhead by managing resource initialization efficiently.

---

### 3.2 Builder Pattern

Simplifies the creation of complex AWS SDK requests with a clean and fluent API for improved readability.

- **Use Case**:
  - Constructing **DynamoDB PutItem/GetItem requests**.
  - Building structured requests for **AWS Secrets Manager**.
- **Benefit**: Enhances maintainability and eliminates redundant request-building logic.

---

### 3.3 Factory Pattern

Uses Spring Boot’s **Dependency Injection** to create and manage beans for shared resources.

- **Use Case**:
  - Instantiating Redis clients, DynamoDB clients, and service classes.
- **Benefit**: Decouples object creation from the main application logic and improves testability.

---

### 3.4 Strategy Pattern

Encapsulates multiple strategies for data access and storage, enabling flexible logic execution based on use cases.

- **Use Case**: 
  - **Cache-first strategy**:
    - Attempt to retrieve data from **Redis** first.
    - Fallback to **DynamoDB** if the cache misses.
- **Benefit**: Improves performance by reducing database latency through caching.

---

### 3.5 Template Method Pattern

Provides a common template for Redis operations while allowing for specific implementations.

- **Use Case**: 
  - Using `RedisTemplate` to simplify `GET`, `SET`, and expiry operations for URL lookups.
- **Benefit**: Standardizes interactions with Redis and reduces code duplication.

---

### 3.6 Proxy Pattern

The **API Gateway** acts as a proxy to forward incoming HTTP requests to the AWS Lambda function.

- **Use Case**: Decouples the API consumer (clients) from the internal application logic.
- **Benefit**: Improves scalability and adds a unified entry point for API management.

---

### 3.7 Observer Pattern

**CloudWatch** acts as an observer, monitoring system health, metrics, and logs in real-time.

- **Use Case**:
  - Observing Lambda function metrics like invocation count, latency, and errors.
  - Configuring CloudWatch alarms for failures or anomalies.
- **Benefit**: Ensures proactive system monitoring and automated alerting.

---

### 3.8 Decorator Pattern

Adds additional functionality, such as **security** and **rate limiting**, without modifying the core business logic.

- **Use Case**: 
  - AWS WAF applies rate limiting and IP filtering to secure the **API Gateway**.
- **Benefit**: Enhances security transparently without changing existing logic.

---

### Summary

The use of these design patterns ensures:

1. **Scalability**: Patterns like **Strategy** and **Proxy** allow for flexible logic and clean architecture.
2. **Maintainability**: Patterns such as **Singleton** and **Builder** simplify code structure and resource management.
3. **Performance**: Patterns like **Cache-first (Strategy)** reduce latency and optimize database calls.
4. **Security**: **Decorator Pattern** ensures security features (e.g., WAF) are applied seamlessly.
5. **Observability**: **Observer Pattern** helps track metrics and maintain system health.

---
<br>

## 4. Code Implementation

This section provides the key components of the application, including dependencies, configurations, and core service classes.

---

### 4.1 Dependencies (`pom.xml`)

This project uses **Spring Boot 3**, **AWS SDK**, **Redis**, and **OAuth2** for Google authentication. The dependencies ensure all required functionalities are supported.

Below is the content of `pom.xml` with the necessary dependencies:

```xml
<dependencies>
    <!-- Spring Boot Web for REST API -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Spring Boot Data Redis for caching -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>

    <!-- AWS SDK for DynamoDB -->
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>dynamodb</artifactId>
    </dependency>

    <!-- AWS SDK for Secrets Manager -->
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>secretsmanager</artifactId>
    </dependency>

    <!-- Spring Boot OAuth2 Client for Google Authentication -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-client</artifactId>
    </dependency>

    <!-- Jackson for JSON handling -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>

    <!-- Spring Boot Starter for Testing -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

#### Explanation:

1. **Spring Boot Starter Web**: Enables RESTful web services for API endpoints.
2. **Spring Boot Data Redis**: Provides caching capabilities for optimized URL lookups.
3. **AWS SDK for DynamoDB and Secrets Manager**: Interacts with AWS DynamoDB for persistent storage and Secrets Manager for secure credential retrieval.
4. **OAuth2 Client**: Supports Google Authentication for securing API endpoints.
5. **Jackson Databind**: Handles JSON serialization and deserialization.
6. **Spring Boot Starter Test**: Supports testing capabilities for unit tests and integration tests.

---

### 4.2 Configuration Properties (`application.yml`)

The **`application.yml`** file centralizes all application configurations, ensuring clean and maintainable code.

````yaml
app:
  base-url: https://enok.tech/shortener

aws:
  dynamodb:
    table-name: URLMappings
  secrets:
    name: shortener/secrets
  region: us-east-1

spring:
  redis:
    host: ${REDIS_HOST:redis.shortener-enok.tech}
    port: ${REDIS_PORT:6379}
    timeout: 2000ms
````

---

#### Explanation:

1. **`app.base-url`**:  
   - Defines the base URL for the application endpoints.  
   - This ensures all URL generation within the application uses a consistent base (`https://enok.tech/shortener`).

2. **`aws.dynamodb.table-name`**:  
   - Specifies the DynamoDB table used for storing the URL mappings.

3. **`aws.secrets.name`**:  
   - Path to the AWS Secrets Manager secret (`shortener/secrets`) where sensitive credentials like Redis host, port, and DynamoDB table name are stored.

4. **`aws.region`**:  
   - Defines the AWS region where resources like DynamoDB, ElastiCache, and Secrets Manager are provisioned.

5. **`spring.redis.host`**:  
   - Custom DNS endpoint `redis.shortener-enok.tech` created in GoDaddy as a CNAME record pointing to the AWS ElastiCache Redis endpoint.

6. **`spring.redis.port`**:  
   - The Redis port, defaulting to `6379`.

7. **`spring.redis.timeout`**:  
   - Timeout configuration for Redis operations, set to `2000ms` (2 seconds) for responsiveness.

---

The **`application.yml`** structure ensures that environment-specific configurations, such as Redis host, AWS resource names, and API base URLs, are organized and easily maintainable.

---

### 4.3 Secrets Management (`SecretsService.java`)

The `SecretsService` class retrieves secure credentials, such as database usernames and passwords, from **AWS Secrets Manager**.

```java
package tech.enok.shortener.service;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.secretsmanager.SecretsManagerClient;
import software.amazon.awssdk.services.secretsmanager.model.GetSecretValueRequest;

import java.util.Map;

@Service
public class SecretsService {

    private final SecretsManagerClient secretsClient;

    public SecretsService(SecretsManagerClient secretsClient) {
        this.secretsClient = secretsClient;
    }

    public Map<String, String> getSecrets(String secretName) {
        GetSecretValueRequest request = GetSecretValueRequest.builder()
                .secretId(secretName)
                .build();
        String secretString = secretsClient.getSecretValue(request).secretString();

        try {
            return new ObjectMapper().readValue(secretString, Map.class);
        } catch (Exception e) {
            throw new RuntimeException("Error retrieving secrets from AWS Secrets Manager", e);
        }
    }
}
```

#### Explanation:

1. **SecretsManagerClient**: AWS SDK client used to connect to AWS Secrets Manager.
2. **getSecrets(String secretName)**:
   - Retrieves the secret by its name (`secretId`) from AWS Secrets Manager.
   - Parses the response (`secretString`) as a JSON string and converts it to a `Map` using Jackson’s `ObjectMapper`.
3. **Error Handling**: Ensures that parsing or retrieval errors are gracefully handled with a `RuntimeException`.

#### Benefits:
- **Security**: Keeps sensitive credentials (e.g., database passwords) secure and out of the codebase.
- **Seamless Updates**: Allows credentials to be updated in Secrets Manager without redeploying the application.
- **Integration**: AWS SDK provides built-in support for Secrets Manager, making it easy to integrate.

---

### 4.4 URL Shortener Service (`UrlShortenerService.java`)

The `UrlShortenerService` handles the core logic for generating short URLs, storing them, and managing redirection. It uses Redis for caching and DynamoDB for persistent storage.

```java
package tech.enok.shortener.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.dynamodb.DynamoDbClient;
import software.amazon.awssdk.services.dynamodb.model.*;

import java.util.Map;
import java.util.UUID;

@Service
public class UrlShortenerService {

    @Value("${app.base-url}")
    private String baseUrl;

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    @Autowired
    private DynamoDbClient dynamoDbClient;

    public String shortenUrl(String longUrl) {
        String shortUrl = UUID.randomUUID().toString().substring(0, 8);
        storeMapping(shortUrl, longUrl);
        return String.format("%s/%s", baseUrl, shortUrl);
    }

    public String getLongUrl(String shortUrl) {
        // Check Redis cache first
        String cachedUrl = redisTemplate.opsForValue().get(shortUrl);
        if (cachedUrl != null) {
            return cachedUrl;
        }
        // Fallback to DynamoDB
        return fetchFromDynamoDB(shortUrl);
    }

    private void storeMapping(String shortUrl, String longUrl) {
        // Store in Redis cache
        redisTemplate.opsForValue().set(shortUrl, longUrl);
        // Persist to DynamoDB
        dynamoDbClient.putItem(PutItemRequest.builder()
                .tableName("URLMappings")
                .item(Map.of(
                        "shortUrl", AttributeValue.builder().s(shortUrl).build(),
                        "longUrl", AttributeValue.builder().s(longUrl).build()
                ))
                .build());
    }

    private String fetchFromDynamoDB(String shortUrl) {
        GetItemResponse response = dynamoDbClient.getItem(GetItemRequest.builder()
                .tableName("URLMappings")
                .key(Map.of("shortUrl", AttributeValue.builder().s(shortUrl).build()))
                .build());

        if (response.hasItem()) {
            String longUrl = response.item().get("longUrl").s();
            redisTemplate.opsForValue().set(shortUrl, longUrl); // Cache the result
            return longUrl;
        }
        return null;
    }
}
```

#### Explanation:

1. **shortenUrl(String longUrl)**:
   - Generates a short URL by creating a random 8-character string using `UUID`.
   - Stores the mapping of `shortUrl` → `longUrl` in both Redis (cache) and DynamoDB (persistent storage).
   - Returns the complete shortened URL (e.g., `https://enok.tech/shortener/<shortId>`).

2. **getLongUrl(String shortUrl)**:
   - Checks **Redis** first to see if the `shortUrl` exists in the cache.
   - If not found, queries **DynamoDB** to retrieve the `longUrl`.
   - Updates Redis with the retrieved URL for future lookups.

3. **storeMapping(String shortUrl, String longUrl)**:
   - Adds the short-to-long URL mapping to:
     - **Redis** for fast access.
     - **DynamoDB** for persistent storage.

4. **fetchFromDynamoDB(String shortUrl)**:
   - Queries DynamoDB to fetch the `longUrl` corresponding to the given `shortUrl`.
   - Updates Redis with the result to optimize future lookups.

#### Key Features:
- **Caching**: Reduces latency with Redis for frequent lookups.
- **Persistence**: DynamoDB ensures the mappings are durable and highly available.
- **Efficiency**: Implements a **cache-first strategy** to optimize performance.
- **Scalability**: Works efficiently under high traffic with serverless architecture.

---

### 4.5 Redis Configuration (`RedisConfig.java`)

The Redis configuration integrates with AWS ElastiCache Redis for caching frequently accessed URL mappings. The connection details are managed dynamically through `application.yml` and AWS Secrets Manager.

````java
package tech.enok.shortener.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisStandaloneConfiguration;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;

@Configuration
public class RedisConfig {

    @Value("${spring.redis.host}")
    private String redisHost;

    @Value("${spring.redis.port}")
    private int redisPort;

    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration(redisHost, redisPort);
        return new LettuceConnectionFactory(config);
    }

    @Bean
    public RedisTemplate<String, String> redisTemplate() {
        RedisTemplate<String, String> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        return template;
    }
}
````

---

#### Explanation:

1. **`@Value("${spring.redis.host}")` and `@Value("${spring.redis.port}")`**:  
   - Dynamically retrieves the Redis host and port from the `application.yml` configuration.  
   - Host: `redis.shortener-enok.tech`  
   - Port: `6379`

2. **`RedisStandaloneConfiguration`**:  
   - Configures a standalone Redis instance connection using the provided host and port.

3. **`LettuceConnectionFactory`**:  
   - Provides a Redis connection factory using the **Lettuce** client for Spring Data Redis.  
   - This factory is lightweight and thread-safe, ensuring efficient connections.

4. **`RedisTemplate<String, String>`**:  
   - A utility class that simplifies Redis operations.  
   - Handles common tasks such as storing and retrieving string-based keys and values.

5. **Dynamic Configuration**:  
   - The Redis host and port are dynamically loaded from `application.yml`, which makes it easy to switch environments (e.g., local, staging, production).

6. **Custom Redis Endpoint**:  
   - The host `redis.shortener-enok.tech` is a **CNAME** pointing to the AWS ElastiCache endpoint.  
   - This approach abstracts the AWS-specific hostname and provides a clean, human-readable DNS name.

---

#### Benefits:

1. **Low-Latency Caching**:  
   - Redis significantly reduces response times by caching frequently accessed URL mappings, improving performance.

2. **Dynamic Configuration**:  
   - The host and port are configurable through `application.yml` and AWS Secrets Manager, making it easy to adapt across environments.

3. **Scalable Architecture**:  
   - AWS ElastiCache can scale horizontally and vertically, ensuring high availability under heavy traffic loads.

4. **Clean Abstraction**:  
   - Using a custom DNS (`redis.shortener-enok.tech`) abstracts the backend infrastructure, allowing seamless upgrades or migrations.

5. **Maintainability**:  
   - `RedisTemplate` simplifies interactions with Redis, making it easier to implement caching logic without boilerplate code.

6. **Thread-Safe Connections**:  
   - The **LettuceConnectionFactory** ensures efficient and thread-safe connections to Redis.

7. **Improved Reliability**:  
   - By using AWS-managed Redis (ElastiCache), the system benefits from automatic backups, failovers, and monitoring.

---

By leveraging **Spring Data Redis** and AWS ElastiCache with a custom DNS endpoint, the application achieves improved **performance**, **scalability**, and **maintainability** for URL caching.

---

### 4.6 Controller Class (`UrlShortenerController.java`)

The **Controller Class** manages HTTP requests to shorten URLs and retrieve original URLs. It maps the endpoints to ensure they align with the **API Gateway configuration** and the **application.yml** properties.

```java
package tech.enok.shortener.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import tech.enok.shortener.service.UrlShortenerService;

@RestController
@RequestMapping("/shortener")
public class UrlShortenerController {

    @Autowired
    private UrlShortenerService urlShortenerService;

    /**
     * Endpoint to shorten a given long URL.
     * Matches API Gateway: POST /shortener/shorten
     * 
     * @param longUrl - Original long URL sent in the request body.
     * @return Shortened URL.
     */
    @PostMapping("/shorten")
    public ResponseEntity<String> shortenUrl(@RequestBody String longUrl) {
        String shortUrl = urlShortenerService.shortenUrl(longUrl);
        return ResponseEntity.ok(shortUrl);
    }

    /**
     * Endpoint to redirect to the original long URL.
     * Matches API Gateway: GET /shortener/{shortUrlId}
     * 
     * @param shortUrlId - Unique identifier for the shortened URL.
     * @return Redirect to the long URL with HTTP status 302 (Found).
     */
    @GetMapping("/{shortUrlId}")
    public ResponseEntity<Void> redirectToLongUrl(@PathVariable String shortUrlId) {
        String longUrl = urlShortenerService.getLongUrl(shortUrlId);
        if (longUrl != null) {
            return ResponseEntity.status(302).header("Location", longUrl).build();
        } else {
            return ResponseEntity.notFound().build();
        }
    }
}

```

#### Key Highlights:

1. **Class-level Mapping**:  
   - **`@RequestMapping("/shortener")`**: Ensures that all endpoints under this controller are prefixed with `/shortener`.

2. **POST Endpoint: `/shortener/shorten`**  
   - **`@PostMapping("/shorten")`**:  
     - Matches the API Gateway route **`POST /shortener/shorten`**.
     - Accepts a long URL in the request body.
     - Returns a shortened URL generated by the service.

3. **GET Endpoint: `/shortener/{shortUrlId}`**  
   - **`@GetMapping("/{shortUrlId}")`**:  
     - Matches the API Gateway route **`GET /shortener/{shortUrlId}`**.
     - Extracts the short URL ID as a path variable.
     - Returns a **302 Redirect** with the long URL if it exists, or **404 Not Found** otherwise.

4. **Integration with Service**:  
   - Uses the `UrlShortenerService` to:
     - **shortenUrl(longUrl)**: Generate a shortened URL.
     - **getLongUrl(shortUrlId)**: Retrieve the original long URL from DynamoDB or Redis.

5. **HTTP Status Codes**:
   - **200 OK**: For successful short URL generation.
   - **302 Found**: Redirects users to the original long URL.
   - **404 Not Found**: If the short URL ID does not exist.

#### Benefits:
1. **Consistency**: Aligns perfectly with API Gateway, Lambda integration, and the base URL in `application.yml`.
2. **Clean Code**: Separates HTTP concerns into the controller and business logic into the service layer.
3. **RESTful Design**: Adheres to REST conventions for POST and GET methods.
4. **Extensibility**: Easily extendable for additional features (e.g., analytics or deletion of short URLs).

---

#### Example Requests:

1. **Shorten URL**:
   - **Request**:  
     ```http
     POST /shortener/shorten
     Content-Type: application/json

     https://www.youtube.com/watch?v=dQw4w9WgXcQ
     ```
   - **Response**:  
     ```json
     "https://enok.tech/shortener/abcd1234"
     ```

2. **Redirect to Original URL**:
   - **Request**:  
     ```http
     GET /shortener/abcd1234
     ```
   - **Response**:  
     HTTP Status **302** with Header:  
     ```http
     Location: https://www.youtube.com/watch?v=dQw4w9WgXcQ
     ```

3. **Invalid Short URL**:
   - **Request**:  
     ```http
     GET /shortener/invalid123
     ```
   - **Response**:  
     HTTP Status **404 Not Found**.

---
<br>

## **5. Infrastructure as Code (Terraform)**

The following sections outline each resource created for the **URL Shortener Application** using **Terraform**. By defining infrastructure as code, we ensure consistency, automation, and ease of deployment for all AWS resources.

---

### **5.1 Terraform Folder Structure**

To ensure clarity, maintainability, and modularity, the Terraform resources for the **URL Shortener Application** are organized into the following structure:

```
terraform/
│
├── main.tf                 # Core Terraform resources for AWS Lambda, DynamoDB, API Gateway
├── vpc.tf                  # VPC Configuration + Endpoints
├── network.tf              # Network Configuration (subnets, security groups, route tables)
├── ssl.tf                  # SSL Certificates with ACM
├── alb.tf                  # Application Load Balancer (ALB) configuration
├── domain.tf               # Domain Configuration (Route53 setup)
├── waf.tf                  # AWS WAF Configuration for rate limiting
├── apigateway.tf           # API Gateway resource configuration
├── lambda.tf               # AWS Lambda Function definition
├── dynamodb.tf             # DynamoDB Table setup
├── redis.tf                # Redis (ElastiCache) configuration
├── secrets_manager.tf      # AWS Secrets Manager for secure credentials
├── cloudfront.tf           # AWS CloudFront Configuration for global caching
├── outputs.tf              # Outputs for resource endpoints and ARNs
├── variables.tf            # Configuration variables for reusability
├── provider.tf             # AWS provider and Terraform backend configuration
└── backend.tf              # Optional: Remote backend configuration (S3 bucket for state)
```

---

### **Explanation**

1. **`main.tf`**:  
   - Defines core resources such as AWS Lambda, DynamoDB, and API Gateway.

2. **`vpc.tf`**:  
   - Creates the Virtual Private Cloud (VPC) and VPC Endpoints for DynamoDB, Secrets Manager, and S3.

3. **`network.tf`**:  
   - Configures public and private subnets, security groups, route tables, and network ACLs.

4. **`ssl.tf`**:  
   - Requests and manages SSL certificates using AWS Certificate Manager (ACM).

5. **`alb.tf`**:  
   - Configures the Application Load Balancer (ALB) to manage incoming traffic to the application.

6. **`domain.tf`**:  
   - Manages custom domain configurations using AWS Route 53.

7. **`waf.tf`**:  
   - Sets up AWS WAF to protect APIs with rate limiting and security rules.

8. **`apigateway.tf`**:  
   - Defines API Gateway resources, including routes and integrations with Lambda.

9. **`lambda.tf`**:  
   - Configures AWS Lambda function, including environment variables, permissions, and deployment settings.

10. **`dynamodb.tf`**:  
    - Creates and configures the DynamoDB table for persistent storage of URL mappings.

11. **`redis.tf`**:  
    - Sets up ElastiCache Redis in private subnets for low-latency caching.

12. **`secrets_manager.tf`**:  
    - Manages secure credentials using AWS Secrets Manager, including secret rotation policies.

13. **`cloudfront.tf`**:  
    - Configures AWS CloudFront to provide global caching for API Gateway responses.

14. **`outputs.tf`**:  
    - Defines Terraform outputs to display important endpoints, such as API Gateway URLs and resource ARNs.

15. **`variables.tf`**:  
    - Stores reusable configuration variables, reducing redundancy across files.

16. **`provider.tf`**:  
    - Configures the AWS provider and credentials to enable Terraform to interact with AWS.

17. **`backend.tf`** *(Optional)*:  
    - Sets up a remote backend (e.g., S3 bucket) for Terraform state files to ensure consistency and collaboration.

---

### **Benefits**

1. **Modularity**:  
   - Each Terraform file focuses on a specific resource, improving readability and maintainability.

2. **Reusability**:  
   - Common configurations, like variables and outputs, are centralized to reduce duplication.

3. **Automation**:  
   - Terraform automates resource provisioning, ensuring consistent and repeatable deployments.

4. **Scalability**:  
   - The modular structure allows for the addition of new AWS resources without affecting existing configurations.

5. **Collaboration**:  
   - Version-controlled Terraform files enable teams to collaborate effectively.

6. **Consistency**:  
   - The structure aligns with AWS best practices and provides clear organization for infrastructure resources.

---

### **Next Steps**

Proceed to **Step 5.2 (VPC Configuration + Endpoints)** to define the Virtual Private Cloud (VPC) and integrate endpoints for DynamoDB, S3, and Secrets Manager.

---









---



















### 5.1 DynamoDB Table

The **DynamoDB Table** is used to store the mappings of short URLs to their corresponding long URLs. This ensures persistent and highly available storage.

```hcl
resource "aws_dynamodb_table" "url_mapping" {
  name         = "URLMappings"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "shortUrl"

  attribute {
    name = "shortUrl"
    type = "S"
  }

  server_side_encryption {
    enabled = true
  }

  tags = {
    Name        = "URLMappingsTable"
    Environment = "Production"
  }
}
```

#### Explanation:

1. **name**: Sets the table name to `URLMappings`.
2. **billing_mode**: Uses `PAY_PER_REQUEST` mode, ensuring cost-efficiency with on-demand capacity instead of pre-provisioned capacity.
3. **hash_key**: Defines `shortUrl` as the primary key for the table.
4. **attribute**:
   - Specifies the `shortUrl` attribute with the data type `S` (String).
5. **server_side_encryption**:
   - **enabled = true**: Encrypts all data stored in DynamoDB for security.
6. **tags**:
   - Adds metadata for resource management, such as `Name` and `Environment`.

#### Benefits:
- **High Availability**: DynamoDB automatically replicates data across multiple availability zones.
- **Cost-Effective**: On-demand billing ensures you only pay for what you use.
- **Scalability**: DynamoDB can scale seamlessly to handle growing traffic.
- **Security**: Server-side encryption ensures data confidentiality.

---

### 5.2 AWS Lambda Function

The **AWS Lambda Function** hosts and executes the core logic of the **Shortener Application**. The code is packaged as a JAR file and deployed to AWS Lambda.

```hcl
resource "aws_lambda_function" "url_shortener_lambda" {
  function_name = "url-shortener-lambda"
  runtime       = "java21"
  handler       = "tech.enok.shortener.LambdaHandler::handleRequest"
  role          = aws_iam_role.lambda_exec_role.arn

  s3_bucket = "shortener-app-bucket"
  s3_key    = "lambda/shortener.jar"

  environment {
    variables = {
      DYNAMODB_TABLE_NAME = aws_dynamodb_table.url_mapping.name
      APP_BASE_URL        = "https://enok.tech/shortener"
    }
  }

  tags = {
    Name        = "ShortenerLambda"
    Environment = "Production"
  }
}
```

#### Explanation:

1. **function_name**: The name of the Lambda function (`url-shortener-lambda`).
2. **runtime**: Specifies `java21` as the runtime environment for executing the JAR file.
3. **handler**:
   - Entry point of the Lambda function in the packaged code (`tech.enok.shortener.LambdaHandler::handleRequest`).
   - Root package is `tech.enok.shortener` to maintain project organization.
4. **role**:
   - References an **IAM role** (`lambda_exec_role`) that grants permissions for AWS resources like DynamoDB and Secrets Manager.
5. **s3_bucket** and **s3_key**:
   - Code is stored in an S3 bucket (`shortener-app-bucket`).
   - JAR file location is updated to `lambda/shortener.jar`.
6. **environment.variables**:
   - `DYNAMODB_TABLE_NAME`: References the DynamoDB table name (`URLMappings`).
   - `APP_BASE_URL`: Base URL for the shortened links (`https://enok.tech/shortener`).
7. **tags**:
   - Updates the resource metadata to reflect the new application name: **"ShortenerLambda"**.

#### Updates:
- **Repository**: The code will be hosted at `https://github.com/enok/shortener`.
- **Naming Consistency**: Updated naming for Lambda and S3 path to match "shortener".
- **Base URL**: Adjusted to `https://enok.tech/shortener` for consistency.

#### Benefits:
- **Clear Organization**: Updated paths, S3 buckets, and handler names align with the repository structure.
- **Consistency**: Unified naming (`shortener`) across code, infrastructure, and URLs.
- **Flexibility**: Environment variables can be updated without modifying the code.

---

### 5.3 API Gateway

The **API Gateway** serves as the entry point for the application, routing HTTP requests to the AWS Lambda function. It exposes RESTful endpoints for shortening URLs and handling redirection.

```hcl
resource "aws_apigatewayv2_api" "api_gateway" {
  name          = "url-shortener-api"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_stage" "api_stage" {
  api_id      = aws_apigatewayv2_api.api_gateway.id
  name        = "prod"
  auto_deploy = true
}

resource "aws_apigatewayv2_integration" "lambda_integration" {
  api_id           = aws_apigatewayv2_api.api_gateway.id
  integration_type = "AWS_PROXY"
  integration_uri  = aws_lambda_function.url_shortener_lambda.invoke_arn
}

resource "aws_apigatewayv2_route" "shorten_route" {
  api_id    = aws_apigatewayv2_api.api_gateway.id
  route_key = "POST /shortener/shorten"
  target    = "integrations/${aws_apigatewayv2_integration.lambda_integration.id}"
}

resource "aws_apigatewayv2_route" "redirect_route" {
  api_id    = aws_apigatewayv2_api.api_gateway.id
  route_key = "GET /shortener/{shortUrlId}"
  target    = "integrations/${aws_apigatewayv2_integration.lambda_integration.id}"
}
```

#### Explanation:

1. **API Definition**:
   - **aws_apigatewayv2_api**: Creates an HTTP API Gateway named `url-shortener-api`.
   - **protocol_type = "HTTP"**: Specifies that this is an HTTP-based API.

2. **Stage**:
   - **aws_apigatewayv2_stage**: Deploys the API to the `prod` stage and enables automatic deployment.

3. **Integration**:
   - **aws_apigatewayv2_integration**:
     - Integrates the API Gateway with the AWS Lambda function (`url-shortener-lambda`).
     - Uses **AWS_PROXY** mode to forward the full request to Lambda.

4. **Routes**:
   - **shorten_route** (`POST /shortener/shorten`):
     - Handles the API endpoint for shortening URLs.
     - Calls the Lambda function to generate a shortened URL.
   - **redirect_route** (`GET /shortener/{shortUrlId}`):
     - Handles the redirection logic.
     - Extracts `shortUrlId` from the path and forwards the request to Lambda.

5. **Route Key**:
   - Defines the HTTP method (`POST` or `GET`) and path (`/shortener/...`) for each route.

#### Benefits:
- **Centralized Entry Point**: API Gateway simplifies request handling and integrates seamlessly with AWS Lambda.
- **RESTful API**: Exposes clean, standardized endpoints (`POST /shortener/shorten` and `GET /shortener/{shortUrlId}`).
- **Scalability**: API Gateway automatically scales with traffic.
- **Flexibility**: Routes can be extended to include additional endpoints in the future.
- **Repository**: Code integration aligns with `https://github.com/enok/shortener`.

#### Exposed Endpoints:
1. **POST** → `https://<api-gateway-id>.execute-api.<region>.amazonaws.com/prod/shortener/shorten`
2. **GET** → `https://<api-gateway-id>.execute-api.<region>.amazonaws.com/prod/shortener/{shortUrlId}`

---

### 5.4 Redis (ElastiCache)

The **Redis Cluster** (AWS ElastiCache) is used as a caching layer to speed up lookups for shortened URLs, reducing latency and DynamoDB reads.

```hcl
resource "aws_elasticache_cluster" "redis_cluster" {
  cluster_id           = "shortener-redis-cluster"
  engine               = "redis"
  node_type            = "cache.t2.micro"
  num_cache_nodes      = 1
  port                 = 6379
  parameter_group_name = "default.redis6.x"

  tags = {
    Name        = "ShortenerRedis"
    Environment = "Production"
  }
}
```

#### Explanation:

1. **cluster_id**:  
   - The name of the Redis cluster is `shortener-redis-cluster`.

2. **engine**:  
   - Specifies `redis` as the caching engine.

3. **node_type**:  
   - Uses `cache.t2.micro`, a cost-effective instance type suitable for testing and light workloads.  
   - Can be upgraded to higher instance types as the application scales.

4. **num_cache_nodes**:  
   - Sets up a single Redis node. For production, multiple nodes can be configured for redundancy.

5. **port**:  
   - The default Redis port is `6379`.

6. **parameter_group_name**:  
   - Uses the default Redis 6.x parameter group for configuration.

7. **tags**:  
   - Adds metadata for resource identification:  
     - `Name`: **ShortenerRedis**  
     - `Environment`: **Production**

#### Benefits:
- **Performance**: Reduces latency by caching frequently accessed short URL lookups.
- **Cost Efficiency**: Decreases DynamoDB read costs by serving cached results.
- **Scalability**: Redis can scale vertically or horizontally to accommodate growing traffic.
- **Persistence**: Configurable to support Redis persistence if required in the future.

#### Use in Application:
- **Cache-first Strategy**:  
   - Lookup short URLs in Redis first.  
   - If not found, fetch from DynamoDB and update the Redis cache.

#### Connection:
- Connect to Redis using the **host** and **port** of the ElastiCache cluster.
- In production, use security groups to restrict access to the Redis endpoint.

---

### 5.5 AWS Secrets Manager

The **AWS Secrets Manager** securely stores and retrieves sensitive credentials, such as Redis host, Redis port, and DynamoDB table name. This eliminates the need to hardcode sensitive values in the application.

````hcl
resource "aws_secretsmanager_secret" "url_shortener_secrets" {
  name = "shortener/secrets"

  tags = {
    Name        = "ShortenerSecrets"
    Environment = "Production"
  }
}

resource "aws_secretsmanager_secret_version" "url_shortener_secrets_version" {
  secret_id     = aws_secretsmanager_secret.url_shortener_secrets.id
  secret_string = jsonencode({
    redisHost = "redis.shortener-enok.tech",
    redisPort = "6379",
    dbTable   = "URLMappings"
  })
}
````

---

#### Explanation:

1. **`aws_secretsmanager_secret`**:  
   - Creates a new secret in AWS Secrets Manager under the name **`shortener/secrets`**.

2. **`aws_secretsmanager_secret_version`**:  
   - Stores the actual secret as a JSON string in AWS Secrets Manager.  
   - Example JSON:
     ```json
     {
       "redisHost": "redis.shortener-enok.tech",
       "redisPort": "6379",
       "dbTable": "URLMappings"
     }
     ```

3. **Tags**:  
   - Adds metadata for the secret:
     - **Name**: ShortenerSecrets
     - **Environment**: Production.

4. **Custom Redis Endpoint**:  
   - Uses the DNS name `redis.shortener-enok.tech`, which is a CNAME pointing to the AWS ElastiCache Redis instance.

---

#### Integration in the Application:

The secrets are dynamically fetched at runtime using the `SecretsService`.

````java
package tech.enok.shortener.service;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.secretsmanager.SecretsManagerClient;
import software.amazon.awssdk.services.secretsmanager.model.GetSecretValueRequest;

import java.util.Map;

@Service
public class SecretsService {

    private final SecretsManagerClient secretsClient;

    public SecretsService() {
        this.secretsClient = SecretsManagerClient.builder().build();
    }

    public Map<String, String> getSecrets(String secretName) {
        GetSecretValueRequest request = GetSecretValueRequest.builder()
                .secretId(secretName)
                .build();

        String secretString = secretsClient.getSecretValue(request).secretString();

        try {
            return new ObjectMapper().readValue(secretString, Map.class);
        } catch (Exception e) {
            throw new RuntimeException("Error parsing secrets from AWS Secrets Manager", e);
        }
    }
}
````

---

#### application.yml Configuration:

````yaml
aws:
  secrets:
    name: shortener/secrets

spring:
  redis:
    host: ${REDIS_HOST:redis.shortener-enok.tech}
    port: ${REDIS_PORT:6379}
````

---

#### Example Secrets JSON:

This is how the secrets are stored in AWS Secrets Manager:

````json
{
  "redisHost": "redis.shortener-enok.tech",
  "redisPort": "6379",
  "dbTable": "URLMappings"
}
````

---

#### Benefits:

1. **Secure Credential Storage**:  
   - Secrets are encrypted and stored securely in AWS Secrets Manager.

2. **Dynamic Retrieval**:  
   - Secrets are fetched dynamically at runtime, avoiding hardcoded values in code or configuration files.

3. **Automatic Rotation**:  
   - AWS Secrets Manager supports automatic rotation of credentials, enhancing security.

4. **Environment-Specific Configurations**:  
   - Use different secrets for development, staging, and production environments.

5. **Simplified Management**:  
   - Updates to secrets in AWS do not require changes in application code.

6. **Custom Redis Endpoint**:  
   - Integrates with the user-friendly DNS name `redis.shortener-enok.tech`, abstracting the underlying AWS ElastiCache endpoint.

7. **Auditing**:  
   - AWS Secrets Manager logs all access to secrets, making it easier to monitor usage and comply with security requirements.

8. **Scalability and Reliability**:  
   - AWS Secrets Manager is fully managed, ensuring high availability and secure access.

---

By using **AWS Secrets Manager**, the application achieves **secure, scalable, and manageable handling of sensitive credentials**, ensuring minimal risk and easier maintenance.

---

### 5.6 AWS CloudFront Configuration

AWS CloudFront is configured as a Content Delivery Network (CDN) to globally cache API Gateway responses. This improves response times and reduces latency for users accessing the API.

````hcl
resource "aws_cloudfront_distribution" "shortener_cdn" {
  enabled = true
  comment = "CloudFront distribution for URL Shortener API"

  origin {
    domain_name = aws_apigatewayv2_api.shortener_api.api_endpoint
    origin_id   = "apiGatewayOrigin"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  default_cache_behavior {
    target_origin_id       = "apiGatewayOrigin"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD"]
    compress               = true
    default_ttl            = 60
    min_ttl                = 0
    max_ttl                = 3600

    forwarded_values {
      query_string = true
      cookies {
        forward = "none"
      }
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  tags = {
    Name        = "ShortenerCloudFront"
    Environment = "Production"
  }
}
````

---

#### Explanation:

1. **`aws_cloudfront_distribution`**:  
   - Creates a CloudFront distribution to cache and serve content globally.

2. **`origin`**:  
   - Configures **API Gateway** as the origin of CloudFront.  
   - **`domain_name`**: The API Gateway endpoint (automatically fetched from Terraform).  
   - **`origin_protocol_policy`**: Enforces HTTPS communication between CloudFront and the origin.

3. **`default_cache_behavior`**:  
   - Defines the caching behavior for CloudFront:  
     - **`viewer_protocol_policy`**: Redirects HTTP requests to HTTPS.  
     - **Allowed Methods**: Allows GET, HEAD, and OPTIONS requests.  
     - **Caching**:  
       - **`default_ttl`**: Default Time-to-Live for cached content (60 seconds).  
       - **`min_ttl`**: Minimum Time-to-Live for cache (0 seconds).  
       - **`max_ttl`**: Maximum cache duration (3600 seconds).

4. **`viewer_certificate`**:  
   - Uses the default CloudFront SSL certificate to secure connections.

5. **`restrictions`**:  
   - Configures global access with no geographic restrictions.

6. **Tags**:  
   - Adds metadata to identify and manage the CloudFront resource.

---

#### Benefits:

1. **Global Content Delivery**:  
   - CloudFront caches API Gateway responses at edge locations worldwide, reducing latency for users.

2. **Improved Performance**:  
   - Caching frequently accessed content ensures faster responses and lower load on API Gateway and Lambda.

3. **Security**:  
   - Enforces HTTPS communication between clients, CloudFront, and the API Gateway.

4. **Cost Efficiency**:  
   - By reducing the number of direct API Gateway and Lambda requests, CloudFront lowers the operational costs of the application.

5. **Automatic Compression**:  
   - CloudFront automatically compresses content (e.g., JSON responses), reducing data transfer costs and improving client performance.

6. **Scalability**:  
   - CloudFront can handle massive amounts of requests with low latency, ensuring the application scales seamlessly under high traffic.

7. **Flexible TTLs**:  
   - Configurable cache behavior allows fine-tuning of content freshness.

8. **Resiliency**:  
   - CloudFront serves cached content even if the API Gateway or Lambda backend experiences downtime.

---

By integrating **CloudFront** with API Gateway, the application achieves **global performance improvements**, lower latency, and reduced operational costs, ensuring a fast and reliable user experience.

---

### 5.7 AWS WAF Configuration

AWS Web Application Firewall (WAF) is configured to protect the API Gateway from abuse, including rate limiting and basic security threats. WAF ensures that the application remains resilient to malicious traffic while enforcing request limits.

````hcl
resource "aws_wafv2_web_acl" "shortener_waf" {
  name  = "shortener-waf"
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  rule {
    name     = "RateLimitRule"
    priority = 1

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitRuleMetrics"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "ShortenerWAFMetrics"
    sampled_requests_enabled   = true
  }

  tags = {
    Name        = "ShortenerWAF"
    Environment = "Production"
  }
}

resource "aws_wafv2_web_acl_association" "shortener_waf_association" {
  resource_arn = aws_apigatewayv2_api.shortener_api.arn
  web_acl_arn  = aws_wafv2_web_acl.shortener_waf.arn
}
````

---

#### Explanation:

1. **`aws_wafv2_web_acl`**:  
   - Creates a **Web ACL** for the application with rules to protect API Gateway.

2. **Default Action**:  
   - If no rules match, allow all requests by default. Additional rules can be added for stricter security.

3. **`RateLimitRule`**:  
   - Enforces rate limiting with the following parameters:
     - **Limit**: Allows a maximum of **2000 requests per 5 minutes** per IP.  
     - **Action**: Blocks the IP when the limit is exceeded.  

4. **`visibility_config`**:  
   - Enables CloudWatch metrics and sampled request logs for monitoring the WAF activity.

5. **`aws_wafv2_web_acl_association`**:  
   - Associates the WAF Web ACL with the API Gateway to enforce the defined rules.

6. **Tags**:  
   - Metadata for identification and management of the WAF resource.

---

#### Benefits:

1. **Abuse Protection**:  
   - Rate limiting prevents abuse by limiting the number of requests per IP, mitigating risks such as DDoS attacks.

2. **Security Rules**:  
   - WAF rules can be extended to block malicious patterns, SQL injections, and other common web vulnerabilities.

3. **Integration with CloudWatch**:  
   - Logs and metrics provide visibility into traffic patterns and potential threats.

4. **Scalable Protection**:  
   - AWS WAF automatically scales to protect API Gateway under high traffic loads.

5. **Custom Rules**:  
   - You can define additional security rules to match the specific needs of the application.

6. **Cost Optimization**:  
   - By blocking unnecessary or malicious requests early, WAF reduces API Gateway and Lambda invocation costs.

7. **Resilience**:  
   - WAF ensures the application remains available and performant even during traffic spikes or malicious attacks.

8. **Centralized Security Management**:  
   - WAF can be managed centrally, providing a uniform security layer across multiple AWS resources.

---

By integrating **AWS WAF** with API Gateway, the application achieves enhanced **security** and **resilience** through rate limiting and traffic filtering, protecting against abuse and ensuring reliable operations.


---

### 5.8 Network Configuration

This section defines a **VPC**, its subnets, route tables, security groups, and network ACLs to ensure secure and isolated network communication.

````hcl
# VPC Configuration
resource "aws_vpc" "shortener_vpc" {
  cidr_block = "10.0.0.0/16"
  enable_dns_support = true
  enable_dns_hostnames = true

  tags = {
    Name        = "ShortenerVPC"
    Environment = "Production"
  }
}

# Subnets (Public and Private)
resource "aws_subnet" "public_subnet" {
  vpc_id                  = aws_vpc.shortener_vpc.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true

  tags = {
    Name        = "PublicSubnet"
    Environment = "Production"
  }
}

resource "aws_subnet" "private_subnet" {
  vpc_id                  = aws_vpc.shortener_vpc.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = "us-east-1a"

  tags = {
    Name        = "PrivateSubnet"
    Environment = "Production"
  }
}

# Internet Gateway for Public Subnet
resource "aws_internet_gateway" "shortener_igw" {
  vpc_id = aws_vpc.shortener_vpc.id

  tags = {
    Name        = "ShortenerIGW"
    Environment = "Production"
  }
}

# Route Table and Association for Public Subnet
resource "aws_route_table" "public_route_table" {
  vpc_id = aws_vpc.shortener_vpc.id

  tags = {
    Name        = "PublicRouteTable"
    Environment = "Production"
  }
}

resource "aws_route" "public_internet_route" {
  route_table_id         = aws_route_table.public_route_table.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.shortener_igw.id
}

resource "aws_route_table_association" "public_subnet_association" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.public_route_table.id
}

# Security Group for Lambda and ElastiCache
resource "aws_security_group" "shortener_sg" {
  vpc_id = aws_vpc.shortener_vpc.id
  name   = "ShortenerSecurityGroup"

  ingress {
    description = "Allow HTTPS inbound traffic"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Allow Redis connections"
    from_port   = 6379
    to_port     = 6379
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }

  egress {
    description = "Allow all outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "ShortenerSecurityGroup"
    Environment = "Production"
  }
}

# Network ACLs for Public and Private Subnets
resource "aws_network_acl" "shortener_nacl" {
  vpc_id = aws_vpc.shortener_vpc.id
  tags = {
    Name        = "ShortenerNACL"
    Environment = "Production"
  }
}

resource "aws_network_acl_rule" "allow_https_inbound" {
  network_acl_id = aws_network_acl.shortener_nacl.id
  rule_number    = 100
  protocol       = "tcp"
  rule_action    = "allow"
  egress         = false
  cidr_block     = "0.0.0.0/0"
  from_port      = 443
  to_port        = 443
}

resource "aws_network_acl_rule" "allow_redis_inbound" {
  network_acl_id = aws_network_acl.shortener_nacl.id
  rule_number    = 110
  protocol       = "tcp"
  rule_action    = "allow"
  egress         = false
  cidr_block     = "10.0.0.0/16"
  from_port      = 6379
  to_port        = 6379
}

resource "aws_network_acl_rule" "allow_all_outbound" {
  network_acl_id = aws_network_acl.shortener_nacl.id
  rule_number    = 200
  protocol       = "-1"
  rule_action    = "allow"
  egress         = true
  cidr_block     = "0.0.0.0/0"
  from_port      = 0
  to_port        = 0
}
````

---

### Explanation:

1. **VPC**:  
   - Defines a Virtual Private Cloud with a **`10.0.0.0/16`** CIDR block to isolate resources securely.

2. **Public and Private Subnets**:  
   - Public Subnet: For resources requiring internet access, such as API Gateway.  
   - Private Subnet: For resources like Lambda and ElastiCache.

3. **Internet Gateway**:  
   - Connects the public subnet to the internet.

4. **Route Table and Route Association**:  
   - Routes external traffic to the internet gateway for public subnets.

5. **Security Group**:  
   - Allows inbound HTTPS (443) for API Gateway and Redis access (6379) within the VPC.  
   - Allows all outbound traffic for communication with external services.

6. **Network ACLs (NACL)**:  
   - Adds an additional layer of security to control inbound and outbound traffic for subnets.

---

### Benefits:

1. **Isolation and Security**:  
   - Resources are isolated within a VPC, providing enhanced security.

2. **Granular Access Control**:  
   - Security Groups and Network ACLs allow fine-grained control over traffic.

3. **Public and Private Subnets**:  
   - Public subnet exposes API Gateway while private subnets keep Lambda and ElastiCache secure.

4. **Scalable and Flexible**:  
   - AWS VPC supports scaling, enabling the addition of more resources or subnets as needed.

5. **Defense in Depth**:  
   - Combining Security Groups and Network ACLs provides multiple layers of security.

6. **Cost Efficiency**:  
   - Resources like ElastiCache can reside in private subnets, minimizing unnecessary exposure.

---

This **network configuration** ensures secure communication and access control for all AWS resources, aligning with best practices for production-ready infrastructure.

