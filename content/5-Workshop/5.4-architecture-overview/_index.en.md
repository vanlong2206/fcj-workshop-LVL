---
title : "Project Architecture"
date : 2026-06-18
weight : 4
chapter : false
pre : " <b> 5.4. </b> "
---

#### 5.4.1 Architecture Overview

System architecture before project deployment:

![1782935686221](image/_index.vi/1782935686221.png)

Expected system architecture after deploying with serverless (Lambda):

```
services/
├── lambda-auth/                    # gameapi-auth
│   ├── src/
│   │   ├── controllers/AccountsController.ts
│   │   ├── models/
│   │   │   ├── Account.ts
│   │   │   ├── Role.ts
│   │   │   ├── UserStat.ts
│   │   │   └── UserCurrency.ts
│   │   ├── routes.ts
│   │   ├── lambda.ts               # Lambda handler
│   │   └── index.ts
│   ├── package.json
│   ├── tsconfig.json
│   └── serverless.yml
│
├── lambda-economy/                 # gameapi-economy
│   ├── src/
│   │   ├── controllers/EconomyController.ts
│   │   ├── models/UserCurrency.ts
│   │   ├── routes.ts
│   │   ├── lambda.ts
│   │   └── index.ts
│   ├── package.json
│   ├── tsconfig.json
│   └── serverless.yml
│
├── lambda-player-stats/            # gameapi-player-stats
├── lambda-forum/                   # gameapi-forum
├── lambda-inventory/               # gameapi-inventory
├── lambda-giftcode/                # gameapi-giftcode
├── lambda-gamedata/                # gameapi-gamedata
├── lambda-leaderboard/             # gameapi-leaderboard
├── lambda-farm/                    # gameapi-farm (depends on inventory)
├── lambda-storage/                 # gameapi-storage (depends on inventory)
├── lambda-shop/                    # gameapi-shop (depends on economy)
├── lambda-admin/                   # gameapi-admin
│
├── shared/                         # Shared code
│   ├── models/                     # Shared models (copy or package)
│   ├── middlewares/                # auth.middleware, admin.middleware
│   ├── utils/                      # JwtHelper, PasswordHasher, TimeHelper
│   ├── config/                     # database.ts, CropConfig.ts
│   ├── types/                      # Shared DTOs and types
│   └── package.json                # Shared package
│
└── docker-compose.yml              # Local development
```

The system is built on a **Serverless Microservices** model on AWS Lambda. Instead of deploying the entire backend as a single monolithic application, the system is divided into multiple Lambda Functions, each responsible for a specific Business Domain.

Each Lambda is deployed as an independent service that can be developed, tested, and deployed separately. All services are accessed through Amazon API Gateway and share the same Amazon Aurora PostgreSQL database.

This architecture makes the system easy to scale, reduces operational costs, and improves maintainability.

#### 5.4.2 Deploying **Serverless Microservices** on AWS Lambda

##### * **Step 1: Domain Separation**

![1783013847165](image/_index.vi/1783013847165.png)

Each domain can be developed, deployed, and scaled independently. Service A calls Service B via HTTP.

##### * **Step 2: Setting Up Monorepo with npm Workspaces**

```
gameapi/
├── shared/                         # Shared package (@gameapi/shared)
│   └── src/
│       ├── models/                 # TypeORM entities (Account, UserItem, FarmPlot...)
│       ├── middlewares/            # authMiddleware, adminMiddleware
│       ├── utils/                  # JwtHelper, PasswordHasher...
│       └── DTO/                    # Request/Response types
├── services/
│   ├── lambda-auth/                # Each service is a separate workspace
│   ├── lambda-economy/
│   ├── lambda-inventory/
│   ├── lambda-transaction/
│   ├── lambda-progression-world/
│   └── lambda-loot-reward/
└── package.json                    # workspaces: ["shared", "services/*"]
```

Root `package.json`:

```json
{
  "workspaces": ["shared", "services/*", "frontend"],
  "scripts": {
    "build:shared": "npm run build --workspace shared",
    "build:lambdas": "npm run build --workspaces --if-present -- --exclude shared",
    "dev:auth": "npm run dev --workspace services/lambda-auth"
  }
}
```

Share entities, middlewares, and utilities between Lambdas without copy-pasting.

##### * Step 3: Building the Lambda Handler Pattern

Each service uses `@vendia/serverless-express` to wrap an Express app as a Lambda handler:

```typescript
// services/lambda-auth/src/lambda.ts
import serverlessExpress from '@vendia/serverless-express';
import app from './index';

let serverlessExpressInstance;

export const handler = async (event, context) => {
  if (!serverlessExpressInstance) {
    serverlessExpressInstance = serverlessExpress({ app });
  }
  return serverlessExpressInstance(event, context);
};
```

Keep the familiar Express controller code, hot-reload locally with nodemon, only change the entry point when deploying to Lambda.

##### * Step 4: Configuring Serverless Framework for Each Service

```yaml
# services/lambda-auth/serverless.yml
service: gameapi-auth

provider:
  name: aws
  runtime: nodejs20.x
  region: ap-southeast-1
  environment:
    DB_HOST: ${env:DB_HOST}
    DB_PORT: ${env:DB_PORT}
    JWT_SECRET: ${env:JWT_SECRET}
    USE_RDS_IAM_AUTH: 'true'
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
```

##### * Step 5: Database Connection (Aurora PostgreSQL + IAM Auth)

Each Lambda connects to the same Aurora PostgreSQL cluster via TypeORM:

```typescript
// shared/src/config/database.ts
async function getAuroraIamToken(): Promise<string> {
  if (process.env.USE_RDS_IAM_AUTH !== "true")
    return process.env.DB_PASSWORD || "";

  const signer = new Signer({
    hostname: process.env.DB_HOST!,
    port: parseInt(process.env.DB_PORT || "5432"),
    username: process.env.DB_USER!,
    region: process.env.AWS_REGION || "ap-southeast-1",
  });
  return signer.getAuthToken();
}

export const createApplicationDbContext = async () => {
  const dbConfig = await getDbConfigCached();
  return new DataSource({
    ...dbConfig,
    synchronize: true,   // Auto sync schema during dev
    entities: [Account, UserCurrency, ..., FarmPlot]
  });
};
```

**IAM Auth:** Uses `@aws-sdk/rds-signer` to generate tokens automatically, no need to store passwords in env. The Lambda IAM Role needs `rds-db:connect` permission.

##### * Step 6: Cross-Service Communication (HTTP)

When a Lambda needs to call another service (e.g., `Shop.buyItem` needs to deduct currency from Economy + add item to Inventory), it makes HTTP calls to API Gateway:

```typescript
// lambda-transaction calls lambda-economy
await axios.post(`${API_GATEWAY_URL}/Economy/spend`, {
  accountId, amount, currency
});

// lambda-transaction calls lambda-inventory
await axios.post(`${API_GATEWAY_URL}/Inventory/add`, {
  accountId, itemId, quantity
});
```

Services are independent and do not share the DB directly; they communicate only through APIs. `API_GATEWAY_URL` is injected via environment variable.

##### * Step 7: Deploy Script

```bash
#!/bin/bash
# scripts/deploy-lambdas.sh

# Batch 1: Core services (no dependencies)
deploy_service "auth" &
deploy_service "economy" &
deploy_service "inventory" &
wait

# Batch 2: Dependent services
deploy_service "transaction" &
deploy_service "progression-world" &
deploy_service "loot-reward" &
wait
```
