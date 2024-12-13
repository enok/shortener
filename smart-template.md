# Project Diagram for URL Shortener Application

## Layers:
- **Client Layer**: Represents users interacting with the API.
- **API Layer**: AWS API Gateway acts as the entry point for all HTTP requests.
- **Application Layer**: AWS Lambda processes business logic, integrated with Java 21 and Spring Boot 3.
- **Data Layer**: AWS DynamoDB stores URL mappings, Redis (AWS ElastiCache) caches data for faster lookups.
- **Security Layer**: AWS WAF provides rate limiting and security rules, AWS Secrets Manager manages secure credentials.
- **Network Layer**: AWS VPC, with public and private subnets, isolates resources securely.
- **Caching Layer**: AWS CloudFront delivers content globally, reducing latency.

## Components:
1. **Client**
   - Role: Sends HTTP requests to the application.
   - Connection: Communicates with API Gateway.

2. **AWS API Gateway**
   - Role: Exposes RESTful API endpoints for the application.
   - Routes traffic to the AWS Lambda functions.
   - Connected to AWS WAF for security.

3. **AWS WAF**
   - Role: Implements rate limiting (e.g., 2000 requests/IP) and filters malicious traffic.
   - Placement: Integrated with API Gateway.

4. **AWS Lambda**
   - Role: Handles business logic for:
      - Shortening URLs (POST /shorten)
      - Redirecting URLs (GET /{shortUrl}).
   - Connection:
      - Fetches secrets from **AWS Secrets Manager**.
      - Queries Redis (ElastiCache) for cached results.
      - Stores/retrieves data from **AWS DynamoDB**.
      - Processes requests securely within a VPC.

5. **AWS ElastiCache (Redis)**
   - Role: Provides low-latency caching of URL mappings.
   - Endpoint: Configured as `redis.shortener-enok.tech` (custom DNS).
   - Placement: Deployed within a **private subnet** in VPC.

6. **AWS DynamoDB**
   - Role: Persistent storage for original and shortened URLs.
   - Schema: 
      - Primary Key: `shortUrl`
      - Attributes: `longUrl`, `createdAt`.
   - Connection: Accessed securely from AWS Lambda.

7. **AWS Secrets Manager**
   - Role: Securely stores sensitive credentials:
      - Redis endpoint
      - DynamoDB configuration.
   - Connection: Accessed by AWS Lambda at runtime.

8. **AWS CloudFront**
   - Role: Acts as a CDN for caching API Gateway responses globally.
   - Placement: Fronts the API Gateway to reduce latency for global users.
   - SSL: Uses default CloudFront HTTPS certificates.

9. **AWS VPC**
   - Role: Network isolation for application resources.
   - Components:
      - **Public Subnet**: API Gateway and Internet Gateway.
      - **Private Subnet**: AWS Lambda and ElastiCache Redis.
   - Security:
      - Security Groups control access (HTTPS, Redis).
      - Network ACLs filter inbound/outbound traffic.

10. **AWS Glue (ETL Pipeline)**
    - Role: Periodically extracts metadata (headers) from URL mappings.
    - Stores results in **Amazon S3** for analysis.

11. **AWS OpenSearch**
    - Role: Provides usage analytics and visualization for:
      - URL access metrics
      - Performance insights.

12. **GitHub Actions (CI/CD Pipeline)**
    - Role: Automates deployment of infrastructure and application code.
    - Steps:
      - Runs Terraform to provision AWS resources.
      - Deploys Lambda JAR files to S3.
      - Updates Lambda functions.

## Data Flow:
1. **Client** → **API Gateway**:  
   Client sends a POST or GET request.

2. **API Gateway** → **AWS Lambda**:  
   API Gateway forwards requests to Lambda.

3. **AWS Lambda**:  
   - **POST /shorten**:
      - Checks Redis cache.
      - If not present, generates a short URL.
      - Stores the mapping in DynamoDB.
      - Updates Redis cache.
   - **GET /{shortUrl}**:
      - Checks Redis for the short URL.
      - If cache misses, fetches from DynamoDB.
      - Updates Redis with the result.

4. **AWS WAF**:  
   Secures API Gateway by applying rate limiting and traffic filtering.

5. **AWS CloudFront**:  
   Caches API Gateway responses to improve global latency.

6. **AWS Secrets Manager**:  
   Provides Redis and DynamoDB credentials securely to Lambda.

7. **AWS Glue**:  
   Periodically extracts metadata (headers) from URLs and stores in S3.

8. **AWS OpenSearch**:  
   Processes S3 data for analytics and usage visualization.

9. **Monitoring**:  
   AWS CloudWatch monitors all resources and logs.

10. **CI/CD Pipeline**:  
    GitHub Actions automates deployment for both application and infrastructure.

---

### Notes for Smart Template:
- Use rectangles to represent each component.
- Arrows indicate data flow between components.
- Use layers or groupings for organization (Client, API, Security, Caching, Data, Network).
- Highlight key security features (e.g., WAF, Secrets Manager, VPC).
- Use icons: 
   - API Gateway, Lambda, DynamoDB, CloudFront, Redis, VPC, and Secrets Manager (AWS Architecture Icons).

---

This input will guide **draw.io** to create a structured, professional-looking diagram for your project. Let me know if you need further adjustments or refinements! 🚀

Created using eraser.io: https://app.eraser.io/workspace/Q1tMWrmK8je8C8qto7yp