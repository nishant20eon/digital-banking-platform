# User Service — Development, Testing & Troubleshooting Guide

This document contains the local development setup, configuration, API testing commands, security verification, database operations, Maven commands, and troubleshooting steps for the User Service.

**Module:** `services/user-service`  
**Application name:** `user-service`  
**Local port:** `8081`  
**Test database:** `banking_user_test_db`  
**Development database:** `banking_user_db`

---

## 1. Technology Stack

- Java 25
- Spring Boot 4.2.0
- Spring Security
- Spring Data JPA / Hibernate
- PostgreSQL 18.7
- JWT authentication (HMAC SHA-256)
- BCrypt password hashing
- Spring Cloud Netflix Eureka Client
- Jakarta Bean Validation
- JUnit 5 and Mockito
- Maven

## 2. Project Structure

```text
user-service/
├── src/
│   ├── main/
│   │   ├── java/com/example/banking/user/
│   │   │   ├── config/
│   │   │   ├── controller/
│   │   │   ├── dto/
│   │   │   ├── entity/
│   │   │   ├── enums/
│   │   │   ├── exception/
│   │   │   ├── repository/
│   │   │   ├── security/
│   │   │   ├── service/
│   │   │   └── UserServiceApplication.java
│   │   └── resources/
│   │       └── application.yml
│   └── test/
│       ├── java/com/example/banking/user/
│       │   ├── integration/
│       │   │   ├── AuthControllerIntegrationTest.java
│       │   │   └── UserControllerIntegrationTest.java
│       │   └── service/
│       │       └── UserServiceTest.java
│       └── resources/
│           └── application-test.yml
├── pom.xml
└── README.md
```

## 3. Prerequisites

```powershell
java -version
mvn -version
docker --version
docker ps
```

Expected Java version: Java 25.

Required local infrastructure:

| Component | Container | Port |
|---|---|---:|
| PostgreSQL | banking-postgres | 5432 |
| Redis | banking-redis | 6379 |
| Kafka | banking-kafka | 9092 |

The current User Service requires PostgreSQL. Redis and Kafka are not required for its current APIs or tests.

## 4. Start PostgreSQL

```powershell
docker ps
docker exec banking-postgres pg_isready -U banking_local
```

Expected:

```text
localhost:5432 - accepting connections
```

If the existing container is stopped:

```powershell
docker start banking-postgres
```

## 5. Create and Verify Databases

| Database | Purpose |
|---|---|
| `banking_user_db` | Local application development |
| `banking_user_test_db` | Automated integration tests |

Connect:

```powershell
docker exec -it banking-postgres psql -U banking_local -d postgres
```

List databases:

```sql
\l
```

Create each database only if it does not already exist:

```sql
CREATE DATABASE banking_user_db;
CREATE DATABASE banking_user_test_db;
```

Verify:

```sql
SELECT datname
FROM pg_database
WHERE datname IN ('banking_user_db', 'banking_user_test_db');
```

Exit:

```sql
\q
```

### Verify the users table

After starting the application at least once:

```powershell
docker exec -it banking-postgres psql -U banking_local -d banking_user_db
```

```sql
\dt
\d users

SELECT
    id, first_name, last_name, email,
    mobile_number, role, status, created_at
FROM users;

SELECT
    email,
    CASE
        WHEN password_hash IS NOT NULL AND password_hash <> ''
        THEN 'HASH PRESENT'
        ELSE 'HASH MISSING'
    END AS password_status
FROM users;

\q
```

The password verification query deliberately does not display the actual hash.

## 6. Configure Local Environment Variables

Run from the project root:

```powershell
cd G:\Workspace\digital-banking-platform
```

Set the local PostgreSQL password:

```powershell
$env:DB_PASSWORD = "local_only_change_me"
```

Never commit actual passwords or JWT secrets to Git.

### 6.1 Generate a Secure JWT Secret

The application uses a Base64-encoded, cryptographically generated 32-byte secret for HMAC SHA-256.

```powershell
$bytes = New-Object byte[] 32

$rng = [System.Security.Cryptography.RandomNumberGenerator]::Create()
$rng.GetBytes($bytes)
$rng.Dispose()

$env:JWT_SECRET = [Convert]::ToBase64String($bytes)
```

### 6.2 Verify JWT Secret Length

```powershell
if ([string]::IsNullOrWhiteSpace($env:JWT_SECRET)) {
    Write-Host "JWT_SECRET is missing or empty"
}
else {
    Write-Host "JWT_SECRET length: $($env:JWT_SECRET.Length)"
}
```

Expected:

```text
JWT_SECRET length: 44
```

A 32-byte secret encoded in Base64 normally contains 44 characters.

### 6.3 Check Whether the Secret Needs Regeneration

```powershell
if ([string]::IsNullOrWhiteSpace($env:JWT_SECRET)) {
    Write-Host "JWT_SECRET is empty. Generate a new secret."
}
elseif ($env:JWT_SECRET.Length -ne 44) {
    Write-Host "JWT_SECRET length is unexpected. Verify the configuration."
}
else {
    Write-Host "JWT_SECRET is configured."
}
```

If empty, generate a new secret using section 6.1.

**Important:**

- Do not generate a new secret every time you run the application.
- Keep using the same secret during local development.
- Replacing the secret invalidates previously issued JWTs.
- Do not print the secret in logs, screenshots, or Git commits.
- `$env:` variables are available only in that PowerShell session and its child processes.

## 7. Configuration Files

### 7.1 Development Configuration

Location:

```text
services/user-service/src/main/resources/application.yml
```

Important settings:

```yaml
server:
  port: 8081

spring:
  application:
    name: user-service

  datasource:
    url: jdbc:postgresql://localhost:5432/banking_user_db?options=-c%20TimeZone=Asia/Kolkata
    username: banking_local
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver

  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true

app:
  security:
    jwt:
      secret: ${JWT_SECRET}
      expiration: 900000
```

The complete configuration also contains JPA formatting, Eureka registration, and Actuator settings. Do not replace the complete file with this shortened reference.

### 7.2 Test Configuration

Location:

```text
services/user-service/src/test/resources/application-test.yml
```

Important settings:

```yaml
server:
  port: 0

spring:
  application:
    name: user-service-test

  datasource:
    url: jdbc:postgresql://localhost:5432/banking_user_test_db
    username: banking_local
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver

  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: false

eureka:
  client:
    enabled: false

app:
  security:
    jwt:
      secret: ${JWT_SECRET}
      expiration: 900000
```

Notes:

- Integration tests use `banking_user_test_db`, not the development database.
- `ddl-auto: create-drop` creates the test schema and drops it when the test context closes.
- The test profile disables Eureka.
- Do not add the development timezone query parameter to the test JDBC URL.

## 8. Start the User Service

Ensure `DB_PASSWORD` and `JWT_SECRET` are configured in the current PowerShell session.

Check without printing their values:

```powershell
Write-Host "DB_PASSWORD configured: $(-not [string]::IsNullOrWhiteSpace($env:DB_PASSWORD))"
Write-Host "JWT_SECRET length: $($env:JWT_SECRET.Length)"
```

Start from the project root:

```powershell
mvn -pl services/user-service spring-boot:run "-Dspring-boot.run.jvmArguments=-Duser.timezone=UTC"
```

Expected:

- Spring Boot starts successfully.
- PostgreSQL connection succeeds.
- Hibernate initializes JPA entities.
- User Service starts on port 8081.
- Eureka registration succeeds when the Discovery Server is running.

Check health:

```powershell
Invoke-RestMethod http://localhost:8081/actuator/health
```

Expected:

```json
{
  "status": "UP"
}
```

Stop the service with `Ctrl + C`.

## 9. API Endpoints

Base URL:

```text
http://localhost:8081
```

| Method | Endpoint | Authentication | Purpose |
|---|---|---|---|
| POST | `/users/register` | Public | Register a user |
| POST | `/auth/login` | Public | Authenticate and generate JWT |
| GET | `/users/me` | JWT required | Get logged-in user's profile |
| GET | `/actuator/health` | Public | Application health |
| GET | `/actuator/info` | Public | Application information |

### 9.1 Register User

```powershell
$registerBody = @{
    firstName = "John"
    lastName = "Doe"
    email = "john.doe@example.com"
    mobileNumber = "9876543210"
    password = "Password@123"
} | ConvertTo-Json

$registerResponse = Invoke-RestMethod `
    -Method Post `
    -Uri "http://localhost:8081/users/register" `
    -ContentType "application/json" `
    -Body $registerBody

$registerResponse | Format-List
```

Expected: HTTP 201 Created, user ID and safe user details. Password and password hash must not appear in the response.

### 9.2 Test Duplicate Email

Register again with the same email.

Expected:

```text
HTTP 409 Conflict
Email is already registered
```

### 9.3 Test Duplicate Mobile Number

Register another user using the same mobile number.

Expected:

```text
HTTP 409 Conflict
Mobile number is already registered
```

### 9.4 Test Invalid Registration

```powershell
$invalidBody = @{
    firstName = ""
    lastName = "Doe"
    email = "invalid-email"
    mobileNumber = "123"
    password = "123"
} | ConvertTo-Json

try {
    Invoke-RestMethod `
        -Method Post `
        -Uri "http://localhost:8081/users/register" `
        -ContentType "application/json" `
        -Body $invalidBody
}
catch {
    Write-Host "HTTP Status:" ([int]$_.Exception.Response.StatusCode)
    Write-Host $_.ErrorDetails.Message
}
```

Expected: HTTP 400 Bad Request.

## 10. Login and Generate JWT

Endpoint:

```text
POST http://localhost:8081/auth/login
```

```powershell
$loginBody = @{
    email = "john.doe@example.com"
    password = "Password@123"
} | ConvertTo-Json

$loginResponse = Invoke-RestMethod `
    -Method Post `
    -Uri "http://localhost:8081/auth/login" `
    -ContentType "application/json" `
    -Body $loginBody

$token = $loginResponse.accessToken
```

Check response:

```powershell
$loginResponse.tokenType
$loginResponse.expiresIn
```

Expected:

```text
Bearer
900000
```

Token expiration is 900000 milliseconds (15 minutes).

### 10.1 Check Access Token Length

```powershell
if ([string]::IsNullOrWhiteSpace($token)) {
    Write-Host "Access token is missing."
}
else {
    Write-Host "Access token length: $($token.Length)"
}
```

If token length is zero, verify login succeeded and the response contains `accessToken`.

Check whether the JWT has three sections:

```powershell
if (-not [string]::IsNullOrWhiteSpace($token)) {
    $tokenParts = $token.Split('.')

    Write-Host "JWT sections: $($tokenParts.Length)"

    if ($tokenParts.Length -eq 3) {
        Write-Host "JWT structure looks valid."
    }
    else {
        Write-Host "JWT structure is invalid."
    }
}
```

A standard signed JWT has three dot-separated sections:

```text
header.payload.signature
```

Do not confuse access token length with JWT secret length. They are different values.

## 11. Test Authenticated User Profile

Endpoint:

```text
GET http://localhost:8081/users/me
```

```powershell
$headers = @{
    Authorization = "Bearer $token"
}

$profileResponse = Invoke-RestMethod `
    -Method Get `
    -Uri "http://localhost:8081/users/me" `
    -Headers $headers

$profileResponse | Format-List
```

Expected:

- HTTP 200 OK.
- Returns the authenticated user's profile.
- User ID matches the JWT subject.
- Password and password hash are not returned.

### 11.1 Test Without JWT

```powershell
try {
    Invoke-RestMethod `
        -Method Get `
        -Uri "http://localhost:8081/users/me"
}
catch {
    Write-Host "HTTP Status:" ([int]$_.Exception.Response.StatusCode)
}
```

Expected: HTTP 401 Unauthorized.

### 11.2 Test With Invalid JWT

```powershell
$invalidHeaders = @{
    Authorization = "Bearer invalid.token.value"
}

try {
    Invoke-RestMethod `
        -Method Get `
        -Uri "http://localhost:8081/users/me" `
        -Headers $invalidHeaders
}
catch {
    Write-Host "HTTP Status:" ([int]$_.Exception.Response.StatusCode)
}
```

Expected: HTTP 401 Unauthorized.

### 11.3 Test With Incorrect Password

Try login using an incorrect password.

Expected:

```text
401 Unauthorized
Invalid email or password
```

The application intentionally uses a generic authentication error rather than revealing whether an email exists.

## 12. Spring Security Configuration

Location:

```text
services/user-service/src/main/java/com/example/banking/user/config/SecurityConfig.java
```

Current security behavior:

| Endpoint | Access |
|---|---|
| POST `/users/register` | Permit all |
| POST `/auth/login` | Permit all |
| GET `/actuator/health` | Permit all |
| GET `/actuator/info` | Permit all |
| GET `/users/me` | Authenticated |
| Other endpoints | Authenticated by default |

Security configuration includes:

- CSRF disabled for this stateless API.
- Stateless session management.
- Form login disabled.
- HTTP Basic authentication disabled.
- JWT Resource Server authentication enabled.
- Unauthenticated requests receive 401 responses.

### Password Security

Passwords are encoded using BCrypt.

The database stores a password hash, not the original password.

The application verifies passwords using:

```java
passwordEncoder.matches(rawPassword, encodedPassword);
```

Never store raw passwords or return password hashes through API responses.

### JWT Configuration

Location:

```text
services/user-service/src/main/java/com/example/banking/user/config/JwtConfig.java
```

Current JWT settings:

| Property | Value |
|---|---|
| Algorithm | HS256 |
| Secret | Base64-encoded 32-byte key |
| Token expiration | 900000 ms |
| Expiration in minutes | 15 |
| Issuer | digital-banking-platform |
| Token type | Bearer |

JWT claims include:

- Subject: User UUID
- Email
- Role
- Issued-at timestamp
- Expiration timestamp

## 13. Maven Build and Test Commands

Run commands from:

```text
G:\Workspace\digital-banking-platform
```

### Compile User Service

```powershell
mvn -pl services/user-service compile
```

### Run Unit Tests Only

```powershell
mvn -pl services/user-service "-Dtest=UserServiceTest" test
```

### Run Integration Tests Together

```powershell
mvn -pl services/user-service "-Dtest=UserControllerIntegrationTest,AuthControllerIntegrationTest" "-DargLine=-Duser.timezone=UTC" test
```

### Run Complete User Service Test Suite

```powershell
mvn -pl services/user-service "-DargLine=-Duser.timezone=UTC" test
```

### Clean and Build

```powershell
mvn -pl services/user-service "-DargLine=-Duser.timezone=UTC" clean package
```

### Full Verification

```powershell
mvn -pl services/user-service "-DargLine=-Duser.timezone=UTC" clean verify
```

Keep the UTC JVM argument in the test commands because of the timezone compatibility issue described below.

## 14. Important Troubleshooting

### Issue 1: PostgreSQL TimeZone Error

Error:

```text
FATAL: invalid value for parameter "TimeZone": "Asia/Calcutta"
```

Cause:

The Java JVM default timezone resolves to `Asia/Calcutta`. The PostgreSQL instance rejects this timezone value when the JDBC connection is established.

Check JVM timezone:

```powershell
jshell
```

Inside JShell:

```java
System.out.println(java.util.TimeZone.getDefault().getID());
System.out.println(java.time.ZoneId.systemDefault());
```

Exit:

```java
/exit
```

Current verified workaround for Maven tests:

```powershell
mvn -pl services/user-service "-DargLine=-Duser.timezone=UTC" test
```

For Spring Boot application startup:

```powershell
mvn -pl services/user-service spring-boot:run "-Dspring-boot.run.jvmArguments=-Duser.timezone=UTC"
```

Keep the test datasource URL without a timezone parameter:

```yaml
url: jdbc:postgresql://localhost:5432/banking_user_test_db
```

Do not fix this by randomly changing the Hibernate dialect. The original problem is the database connection timezone.

### Issue 2: DB_PASSWORD Is Missing

Check:

```powershell
$env:DB_PASSWORD
```

If empty:

```powershell
$env:DB_PASSWORD = "local_only_change_me"
```

### Issue 3: JWT_SECRET Is Missing

```powershell
if ([string]::IsNullOrWhiteSpace($env:JWT_SECRET)) {
    Write-Host "JWT_SECRET missing"
}
else {
    Write-Host "JWT_SECRET length: $($env:JWT_SECRET.Length)"
}
```

If missing, generate a new secret using section 6.1.

### Issue 4: PostgreSQL Connection Refused

```powershell
docker ps
docker exec banking-postgres pg_isready -U banking_local
Test-NetConnection localhost -Port 5432
```

### Issue 5: Port 8081 Already in Use

Check:

```powershell
netstat -ano | findstr :8081
```

Identify the process:

```powershell
Get-Process -Id <PROCESS_ID>
```

Stop only the process identified as the old User Service instance:

```powershell
Stop-Process -Id <PROCESS_ID>
```

Replace `<PROCESS_ID>` with the actual PID.

### Issue 6: JWT Returns 401

Check:

1. Login request succeeded.
2. Access token is not empty.
3. Authorization header uses `Bearer`.
4. JWT secret is the same one used when the token was generated.
5. Token has not expired.
6. User status is `ACTIVE`.
7. Token was not issued using an older secret.

Correct header:

```text
Authorization: Bearer YOUR_ACCESS_TOKEN
```

### Issue 7: Integration Tests Fail

Check:

- PostgreSQL container is running.
- `banking_user_test_db` exists.
- `DB_PASSWORD` is configured.
- `JWT_SECRET` is configured.
- Test profile is active.
- Test JDBC URL has no timezone parameter.
- Maven test command includes the UTC JVM argument.

Run:

```powershell
mvn -pl services/user-service "-DargLine=-Duser.timezone=UTC" test
```

## 15. Current Automated Test Coverage

Last verified test results:

| Test class | Test count | Result |
|---|---:|---|
| UserControllerIntegrationTest | 6 | Passed |
| AuthControllerIntegrationTest | 4 | Passed |
| UserServiceTest | 16 | Passed |
| **Total** | **26** | **Passed** |

Last verified Maven result:

```text
Tests run: 26, Failures: 0, Errors: 0, Skipped: 0

BUILD SUCCESS
```

This is the current verified baseline. Update the numbers when the test suite changes.

## 16. Quick Start — Commands to Remember

Whenever you open a new PowerShell terminal:

**Step 1: Go to the project**

```powershell
cd G:\Workspace\digital-banking-platform
```

**Step 2: Set the local database password**

```powershell
$env:DB_PASSWORD = "local_only_change_me"
```

**Step 3: Check JWT secret**

```powershell
if ([string]::IsNullOrWhiteSpace($env:JWT_SECRET)) {
    Write-Host "Generate JWT_SECRET"
}
else {
    Write-Host "JWT_SECRET length: $($env:JWT_SECRET.Length)"
}
```

Generate a new secret only when it is missing or you intentionally need to rotate it.

**Step 4: Check PostgreSQL**

```powershell
docker exec banking-postgres pg_isready -U banking_local
```

**Step 5: Run all User Service tests**

```powershell
mvn -pl services/user-service "-DargLine=-Duser.timezone=UTC" test
```

**Step 6: Start User Service**

```powershell
mvn -pl services/user-service spring-boot:run "-Dspring-boot.run.jvmArguments=-Duser.timezone=UTC"
```

**Step 7: Check health**

```powershell
Invoke-RestMethod http://localhost:8081/actuator/health
```

## 17. Future Improvements / Pending Work

These are not claimed as completed features. They can be addressed during later hardening:

- Replace Hibernate `ddl-auto: update` with Flyway database migrations.
- Add robust handling for database unique-constraint race conditions.
- Add refresh-token support if required.
- Add role-based authorization for admin endpoints.
- Add login rate limiting and account lockout protection.
- Improve production exception handling and observability.
- Review password validation against BCrypt's 72-byte input limit.
- Externalize secrets through a proper secret-management mechanism.
- Add more integration and security tests.

---

**Development rule:** Keep development and test databases separate, never commit secrets, and run the complete test suite before committing User Service changes.
