---
title: "Amazon API Gateway Setup"
date: 2026-06-18
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

#### 5.5.1 Introduction to API Gateway

Amazon API Gateway helps create, publish, maintain, monitor, and secure APIs at any scale. In the system, API Gateway acts as the entry point that receives all requests from clients and routes them to the logic handler functions (Lambda).

#### 5.5.2 Workflow

![1783018858523](image/APIGW_diagram.png)

<div align="center"><i>Figure 5.5.1: API Gateway workflow</i></div>

* Client sends an HTTP request to Amazon API Gateway.
* API Gateway identifies the endpoint and routes the request to the corresponding Lambda.
* Lambda is triggered and initializes the execution environment.
* Lambda forwards the request to the appropriate Controller for business logic processing.
* Controller accesses Amazon Aurora PostgreSQL to read or write data.
* After processing, Lambda returns the result to API Gateway.
* API Gateway sends the final response back to the Client.

#### 5.5.3 AWS API Gateway Configuration

* Configure AWS credentials:

```env
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-southeast-1
DB_HOST=
DB_PORT=5432
DB_USER=<db-username>
DB_NAME=postgres
DB_SSL=true
JWT_SECRET=
```

- RDS cluster `<cluster-name>` (Aurora PostgreSQL 17) uses IAM authentication:

```sql
-- Create user and grant permissions
CREATE USER <db-username>;
GRANT rds_iam TO <db-username>;
GRANT ALL ON SCHEMA public TO <db-username>;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO <db-username>;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO <db-username>;
```

* Lambda IAM Role needs permission:

```json
{
    "Effect": "Allow",
    "Action": "rds-db:connect",
    "Resource": "arn:aws:rds-db:<region>:<aws-account-id>:dbuser:*/<db-username>"
}
```

* IAM token generation code (shared/src/config/database.ts):

```typescript
const signer = new Signer({
    hostname: process.env.DB_HOST,
    port: parseInt(process.env.DB_PORT || "5432"),
    username: process.env.DB_USER,
    region: process.env.AWS_REGION || "ap-southeast-1",
});
const password = await signer.getAuthToken();
```

* Initialize serverless.yml for Lambda functions, example configuration for the auth function:

```yaml
service: gameapi-auth

frameworkVersion: '3'

provider:
  name: aws
  runtime: nodejs20.x
  region: ap-southeast-1
  environment:
    DB_HOST: ${env:DB_HOST}
    DB_PORT: ${env:DB_PORT}
    DB_USER: ${env:DB_USER}
    DB_NAME: ${env:DB_NAME}
    DB_SSL: 'true'
    USE_RDS_IAM_AUTH: 'true'
    JWT_SECRET: ${env:JWT_SECRET}
    JWT_ISSUER: ${env:JWT_ISSUER, 'ChroniclesBackend'}
    JWT_AUDIENCE: ${env:JWT_AUDIENCE, 'ChroniclesClient'}
  iamRoleStatements:
    - Effect: Allow
      Action:
        - rds-db:connect
      Resource: 'arn:aws:rds-db:<region>:<aws-account-id>:dbuser:*/<db-username>'

functions:
  api:
    handler: src/lambda.handler
    timeout: 30
    memorySize: 512

plugins:
  - serverless-esbuild

custom:
  esbuild:
    bundle: true
    minify: false
    target: node20
    packager: npm
    sourcemap: false
```

* Deploy:

```shell
npm run build:shared

cd services/lambda-auth && npx serverless@3 deploy
cd services/lambda-economy && npx serverless@3 deploy
cd services/lambda-inventory && npx serverless@3 deploy
cd services/lambda-transaction && npx serverless@3 deploy
cd services/lambda-progression-world && npx serverless@3 deploy
cd services/lambda-loot-reward && npx serverless@3 deploy

cd api-gateway && npx serverless@3 deploy
```

* Lambda functions after deploy:

```
gameapi-auth-dev-api              512MB  nodejs20.x
gameapi-economy-dev-api           512MB  nodejs20.x
gameapi-inventory-dev-api         512MB  nodejs20.x
gameapi-loot-reward-dev-api       512MB  nodejs20.x
gameapi-progression-world-dev-api  512MB  nodejs20.x
gameapi-transaction-dev-api       512MB  nodejs20.x
```

![1783362971367](image/_index.vi/1783362971367.png)

<div align="center"><i>Figure 5.5.2: Project API Gateway</i></div>

![1783362882458](image/_index.vi/1783362882458.png)

<div align="center"><i>Figure 5.5.3: Initialized routes</i></div>

#### 5.5.4 Test AWS API Gateway

![1783018546606](image/_index.vi/1783018546606.png)

<div align="center"><i>Figure 5.5.4: Test user registration</i></div>

![1783018562951](image/_index.vi/1783018562951.png)

<div align="center"><i>Figure 5.5.5: User login.</i></div>

![1783349316662](image/_index.vi/1783349316662.png)

<div align="center"><i>Figure 5.5.6: View dashboard.</i></div>

![1783362731585](image/_index.vi/1783362731585.png)

<div align="center"><i>Figure 5.5.7: Lambda received request successfully</i></div>

![1783349378894](image/_index.vi/1783349378894.png)

<div align="center"><i>Figure 5.5.8: View wallet balance.</i></div>

![1783363100297](image/_index.vi/1783363100297.png)

<div align="center"><i>Figure 5.5.9: Lambda received request successfully</i></div>

![1783349432243](image/_index.vi/1783349432243.png)

<div align="center"><i>Figure 5.5.10: Get equipment list</i></div>

![1783018613506](image/_index.vi/1783018613506.png)

<div align="center"><i>Figure 5.5.11: Purchase equipment</i></div>

![1783364104671](image/_index.vi/1783364104671.png)

<div align="center"><i>Figure 5.5.12: Lambda received request successfully</i></div>

#### 5.5.5 Results

**Lambda Auth**

POST /Accounts/Create → 201 Created: User account created successfully.

POST /Accounts/Login → 200 OK: User login and authentication successful.

**Lambda Economy**

GET /Economy/balance → 200 OK: Player balance retrieved successfully.

**Lambda Inventory**

GET /Inventory/sync → 200 OK: Player inventory data synchronized successfully.

**Lambda Transaction**

GET /Shop/items → 200 OK: Shop item list retrieved successfully.

**Lambda Progression World**

GET /PlayerStats/profile → 200 OK: Player profile and progression info retrieved successfully.

**Lambda Loot Reward**

GET /Leaderboard → 200 OK: Leaderboard data retrieved successfully.

GET /GameData/ping → 200 OK: Service health check successful.
