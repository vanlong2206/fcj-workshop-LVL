---
title: "Compute Sync — Xử Lý Real-Time với Lambda"
date: 2024-01-01
weight: 6
chapter: false
pre: " <b> 5.6. </b> "
---
**Compute Sync** là nhóm các Lambda service xử lý request đồng bộ (synchronous): Client gửi request và **chờ response ngay lập tức**. Thời gian phản hồi mục tiêu < 500ms.

#### 5.6.1 Các Module trong Compute Sync

**lambda-auth**, **lambda-inventory**, **lambda-economy** là *Core Services* — không phụ thuộc service nào khác, có thể deploy song song.

**lambda-transaction** là *Dependent Service* — cần gọi Economy (trừ tiền) và Inventory (thêm item), phải deploy sau.

- **lambda-auth** (Xác thực người dùng): `POST /Accounts/Create`, `POST /Accounts/Login`, `GET /Accounts/Dashboard` — Phụ thuộc: None
- **lambda-inventory** (Kho đồ, Rương, Trang bị): `GET /Inventory/sync`, `POST /Inventory/add`, `POST /Inventory/equip`, `POST /Storage/deposit`, `POST /Storage/withdraw` — Phụ thuộc: None
- **lambda-economy** (Quản lý tiền tệ - Coin, Gem): `GET /Economy/balance`, `POST /Economy/spend`, `POST /Economy/earn` — Phụ thuộc: None
- **lambda-transaction** (Shop, Gift Code): `GET /Shop/items`, `POST /Shop/buy`, `GET /GiftCodes/live`, `POST /GiftCodes/redeem` — Phụ thuộc: Economy, Inventory

---

#### 5.6.2 Kiến Trúc Xử Lý Đồng Bộ

##### Setup Lambda service (serverless.yml)

Mỗi Compute Sync service được deploy **không kèm API Gateway riêng** — chỉ là function + handler. API Gateway tập trung là điểm vào duy nhất.

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
    DB_USER: ${env:DB_USER}
    DB_NAME: ${env:DB_NAME}
    DB_SSL: 'true'
    USE_RDS_IAM_AUTH: 'true'
    JWT_SECRET: ${env:JWT_SECRET}
  iamRoleStatements:
    - Effect: Allow
      Action: rds-db:connect
      Resource: 'arn:aws:rds-db:ap-southeast-1:xxx:dbuser:*/quan'

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

→ Cấu hình serverless.yml cho mỗi Lambda service: runtime node20, IAM role cho RDS IAM Auth, dùng serverless-esbuild để bundle TypeScript thành 1 file. **Không có `events`** vì API Gateway tập trung gọi qua Lambda Integration.

##### Setup API Gateway tập trung

File `api-gateway/serverless.yml` định nghĩa HTTP API V2 và route từng path đến Lambda tương ứng:

```yaml
# api-gateway/serverless.yml
service: gameapi-gateway

provider:
  name: aws
  runtime: nodejs20.x
  region: ap-southeast-1

resources:
  Resources:
    HttpApi:
      Type: AWS::ApiGatewayV2::Api
      Properties:
        Name: gameapi-gateway
        ProtocolType: HTTP
        CorsConfiguration:
          AllowOrigins: ['*']
          AllowMethods: ['*']
          AllowHeaders: ['*']

    AuthIntegration:
      Type: AWS::ApiGatewayV2::Integration
      Properties:
        ApiId: !Ref HttpApi
        IntegrationType: AWS_PROXY
        IntegrationUri:
          Fn::Sub:
            - arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${FunctionArn}/invocations
            - FunctionArn: !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:gameapi-auth-dev-api
        PayloadFormatVersion: '2.0'

    AuthRoute:
      Type: AWS::ApiGatewayV2::Route
      Properties:
        ApiId: !Ref HttpApi
        RouteKey: ANY /Accounts/{proxy+}
        Target: !Join ['/', ['integrations', !Ref AuthIntegration]]

    AuthLambdaPermission:
      Type: AWS::Lambda::Permission
      Properties:
        Action: lambda:InvokeFunction
        FunctionName: !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:gameapi-auth-dev-api
        Principal: apigateway.amazonaws.com

    InventoryIntegration:
      Type: AWS::ApiGatewayV2::Integration
      Properties:
        ApiId: !Ref HttpApi
        IntegrationType: AWS_PROXY
        IntegrationUri:
          Fn::Sub:
            - arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${FunctionArn}/invocations
            - FunctionArn: !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:gameapi-inventory-dev-api
        PayloadFormatVersion: '2.0'

    InventoryRoute:
      Type: AWS::ApiGatewayV2::Route
      Properties:
        ApiId: !Ref HttpApi
        RouteKey: ANY /Inventory/{proxy+}
        Target: !Join ['/', ['integrations', !Ref InventoryIntegration]]

    InventoryLambdaPermission:
      Type: AWS::Lambda::Permission
      Properties:
        Action: lambda:InvokeFunction
        FunctionName: !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:gameapi-inventory-dev-api
        Principal: apigateway.amazonaws.com

    EconomyIntegration:
      Type: AWS::ApiGatewayV2::Integration
      Properties:
        ApiId: !Ref HttpApi
        IntegrationType: AWS_PROXY
        IntegrationUri:
          Fn::Sub:
            - arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${FunctionArn}/invocations
            - FunctionArn: !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:gameapi-economy-dev-api
        PayloadFormatVersion: '2.0'

    EconomyRoute:
      Type: AWS::ApiGatewayV2::Route
      Properties:
        ApiId: !Ref HttpApi
        RouteKey: ANY /Economy/{proxy+}
        Target: !Join ['/', ['integrations', !Ref EconomyIntegration]]

    EconomyLambdaPermission:
      Type: AWS::Lambda::Permission
      Properties:
        Action: lambda:InvokeFunction
        FunctionName: !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:gameapi-economy-dev-api
        Principal: apigateway.amazonaws.com

    TransactionIntegration:
      Type: AWS::ApiGatewayV2::Integration
      Properties:
        ApiId: !Ref HttpApi
        IntegrationType: AWS_PROXY
        IntegrationUri:
          Fn::Sub:
            - arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${FunctionArn}/invocations
            - FunctionArn: !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:gameapi-transaction-dev-api
        PayloadFormatVersion: '2.0'

    TransactionRoute:
      Type: AWS::ApiGatewayV2::Route
      Properties:
        ApiId: !Ref HttpApi
        RouteKey: ANY /Shop/{proxy+}
        Target: !Join ['/', ['integrations', !Ref TransactionIntegration]]

    TransactionLambdaPermission:
      Type: AWS::Lambda::Permission
      Properties:
        Action: lambda:InvokeFunction
        FunctionName: !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:gameapi-transaction-dev-api
        Principal: apigateway.amazonaws.com

Outputs:
  ApiEndpoint:
    Value: !Sub https://${HttpApi}.execute-api.${AWS::Region}.amazonaws.com
```

→ Định nghĩa HTTP API V2 tập trung. Mỗi service có 3 resource: Integration (kết nối đến Lambda), Route (pattern path), LambdaPermission (cho phép API Gateway invoke). Output trả về URL endpoint.

##### API Gateway Routing

API Gateway tập trung định tuyến dựa trên path pattern:

- `ANY /Accounts/{proxy+}` → `gameapi-auth-dev-api`
- `ANY /Inventory/{proxy+}` → `gameapi-inventory-dev-api`
- `ANY /Storage/{proxy+}` → `gameapi-inventory-dev-api`
- `ANY /Economy/{proxy+}` → `gameapi-economy-dev-api`
- `ANY /Shop/{proxy+}` → `gameapi-transaction-dev-api`
- `ANY /GiftCodes/{proxy+}` → `gameapi-transaction-dev-api`

Mỗi Lambda được deploy **không kèm API Gateway riêng** — chỉ là function + handler. API Gateway là điểm vào duy nhất.

---

#### 5.6.3 Cross-Service Communication (Đồng Bộ)

Khi một Lambda cần gọi service khác, nó gọi qua API Gateway (không gọi trực tiếp Lambda). Dependent service cần được setup thêm `API_GATEWAY_URL` trong env.

##### Setup env cho Dependent Service

```yaml
# services/lambda-transaction/serverless.yml
service: gameapi-transaction

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
    API_GATEWAY_URL: 'https://{api-id}.execute-api.ap-southeast-1.amazonaws.com'
  iamRoleStatements:
    - Effect: Allow
      Action: rds-db:connect
      Resource: 'arn:aws:rds-db:ap-southeast-1:xxx:dbuser:*/quan'

functions:
  api:
    handler: src/lambda.handler
    timeout: 30
    memorySize: 512
```

→ Dependent service cần thêm `API_GATEWAY_URL` trong `environment` để biết đường gọi cross-service.

##### Code cross-service call (ApiClient)

```typescript
// shared/src/utils/ApiClient.ts
import axios from 'axios';

export async function callService(serviceUrl: string, path: string, data: any, authHeader?: string) {
  const response = await axios.post(`${serviceUrl}${path}`, data, {
    headers: authHeader ? { Authorization: authHeader } : {},
    timeout: 5000,
  });
  return response.data;
}
```

→ Hàm gọi cross-service: POST request đến Lambda khác qua API Gateway với Authorization header và timeout 5 giây.

##### Saga Pattern — Rollback khi fail

```typescript
// Mua item — đảm bảo tính nhất quán giữa các service
try {
  // Bước 1: Trừ tiền
  await callService(economyUrl, '/Economy/spend', { amount: price }, token);

  try {
    // Bước 2: Thêm item
    await callService(inventoryUrl, '/Inventory/add', { itemId, quantity }, token);
  } catch {
    // Rollback: Hoàn tiền nếu thêm item thất bại
    await callService(economyUrl, '/Economy/earn', { amount: price }, token);
    throw new Error('Add item failed — rolled back');
  }
} catch (error) {
  res.status(500).json({ error: error.message });
}
```

→ Saga pattern: trừ tiền trước, thêm item sau. Nếu thêm item thất bại → tự động hoàn tiền (rollback) để đảm bảo nhất quán dữ liệu giữa các service.

**Nguyên tắc:** Mỗi service độc lập, không share DB trực tiếp, chỉ giao tiếp qua API.

---

#### 5.6.4 Database Access Pattern

Mỗi controller dùng TypeORM repository trực tiếp, query nhỏ gọn trong transaction:

**Auth** — SELECT + INSERT trong transaction:

```typescript
await ApplicationDbContext.manager.transaction(async (manager) => {
    const account = new Account();
    account.id = generatedId;
    account.username = username;
    account.passwordHash = PasswordHasher.hash(password);
    await manager.save(account);
    await manager.save(saveData);
    await manager.save(userStat);
    await manager.save(userCurrency);
});
```

→ Tạo account + saveData + userStat + userCurrency trong 1 transaction, đảm bảo toàn vẹn dữ liệu khi đăng ký.

**Inventory** — SELECT + UPDATE/SWAP (hoán đổi slot):

```typescript
const repo = ApplicationDbContext.getRepository(UserItem);
const item = await repo.findOne({ where: { id: data.itemDbId, accountId } });
item.isEquipped = data.isEquipped;
await repo.save(item);
```

→ Tìm item theo id và accountId, cập nhật trạng thái equip, save lại.

**Economy** — SELECT balance → UPDATE trong transaction:

```typescript
await ApplicationDbContext.manager.transaction(async (manager) => {
    const wallet = await getOrCreateWallet(accountId);
    if (isCoin && wallet.coin >= amount) {
        wallet.coin -= amount;
        wallet.updatedAt = TimeHelper.getVietnamTime();
        await manager.save(wallet);
    }
});
```

→ Trong transaction: kiểm tra số dư, trừ coin/gem, cập nhật thời gian.

**Transaction** — SELECT + UPDATE + INSERT trong 1 transaction:

```typescript
await ApplicationDbContext.manager.transaction(async (manager) => {
    await manager.save(userCurrency);       // UPDATE: trừ tiền
    await manager.save(newItem);            // INSERT: thêm item vào inventory
});
```

→ Trong 1 transaction: UPDATE số dư (trừ tiền) + INSERT item mới. Đảm bảo cả 2 cùng thành công hoặc cùng rollback.

**Đặc điểm:** Mỗi request chỉ đọc/ghi 1-3 dòng. Index đầy đủ trên `accountId`, `itemId`. Query ngắn, không JOIN phức tạp. Response time target < 500ms.

---

#### 5.6.5 Deploy Script

Script deploy tuần tự theo thứ tự dependency: Core services trước, Dependent service sau, API Gateway cuối cùng.

```bash
#!/bin/bash
# scripts/deploy-compute-sync.sh

set -e

deploy_service() {
  local name=$1
  echo "Deploying lambda-$name..."
  cd "services/lambda-$name" && npx serverless@3 deploy --conceal
  cd ../..
}

echo "=== Batch 1: Deploy Core Services ==="
deploy_service "auth" &
deploy_service "inventory" &
deploy_service "economy" &
wait
echo "=== Core services deployed ==="

echo "=== Batch 2: Deploy Dependent Service ==="
deploy_service "transaction"
echo "=== Dependent service deployed ==="

echo "=== Batch 3: Deploy API Gateway ==="
cd api-gateway && npx serverless@3 deploy --conceal
cd ..
echo "=== API Gateway deployed ==="

echo "=== Compute Sync deployment complete ==="
echo "API Endpoint: $(cd api-gateway && npx serverless@3 info --conceal | grep 'ApiEndpoint' | awk '{print $2}')"
```

→ Deploy core services (auth, inventory, economy) song song ở batch 1, dependent service (transaction) ở batch 2, API Gateway ở batch 3. Đợi batch trước hoàn tất mới chạy batch sau.

##### Lambda handler entry point

Mỗi service dùng `@vendia/serverless-express` để wrap Express app:

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

→ Wrap Express app bằng `@vendia/serverless-express`. Instance được tạo 1 lần duy nhất và cache ở global scope (Lambda warm start), giúp giảm cold start đáng kể.
