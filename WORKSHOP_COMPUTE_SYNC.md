# Workshop: Compute Sync — Xử Lý Real-Time với Lambda

**Mục tiêu:** Trình bày nhóm Compute Sync — các Lambda xử lý đồng bộ, phản hồi trực tiếp cho Client trong thời gian thực.

---

## 1. Compute Sync là gì?

**Compute Sync** là nhóm các Lambda service xử lý request đồng bộ (synchronous): Client gửi request và **chờ response ngay lập tức**. Thời gian phản hồi mục tiêu < 500ms.

**Đặc điểm:**

- Request → Response trong cùng một luồng xử lý
- Client bị block cho đến khi nhận được kết quả
- Phù hợp cho các thao tác cần xác nhận tức thì
- DB query ngắn, chỉ đọc/ghi 1-3 dòng

---

## 2. Các Module trong Compute Sync

### lambda-auth — Xác thực người dùng (Core Service)

**Routes:**

```typescript
const router = Router();

router.post('/Accounts/Create', AccountsController.register);
router.post('/Accounts/Login', AccountsController.login);
router.get('/Accounts/Dashboard', authMiddleware, AccountsController.dashboard);
```

→ Định nghĩa 3 endpoint chính cho authentication: đăng ký, đăng nhập, dashboard.

**Business code — Register:**

```typescript
public static async register(req: Request, res: Response): Promise<void> {
    const { username, email, password } = req.body;
    const accountRepo = ApplicationDbContext.getRepository(Account);
    const existingUser = await accountRepo.findOne({
        where: [{ username }, { email }]
    });
    if (existingUser) {
        res.status(400).json({ error: 'Tên người dùng hoặc Email đã tồn tại!' });
        return;
    }
    // Tạo account + saveData + userStat + userCurrency trong 1 transaction
    await ApplicationDbContext.manager.transaction(async (manager) => {
        const account = new Account();
        account.username = username;
        account.passwordHash = PasswordHasher.hash(password);
        // ... tạo saveData, userStat, userCurrency mặc định
        await manager.save(account);
    });
    res.status(201).json({ message: "Đăng ký thành công", accountId });
}
```

→ Kiểm tra username/email đã tồn tại, nếu chưa thì tạo account + saveData + userStat + userCurrency trong 1 transaction.

**Business code — Login:**

```typescript
public static async login(req: Request, res: Response): Promise<void> {
    const { username, password } = req.body;
    const account = await accountRepo.findOne({
        where: [{ username }, { email: username }],
        relations: ['role']
    });
    if (!account || !PasswordHasher.verify(password, account.passwordHash)) {
        res.status(401).json({ error: 'Sai thông tin đăng nhập!' });
        return;
    }
    const token = JwtHelper.generateToken(
        account.id!, account.username, account.role.name,
        process.env.JWT_SECRET!, 'your_issuer', 'your_audience');
    res.status(200).json({ token, accountId: account.id, username: account.username });
}
```

→ Tìm account theo username hoặc email, verify password hash, trả về JWT token nếu hợp lệ.

---

### lambda-inventory — Kho đồ, Rương, Trang bị (Core Service)

**Routes:**

```typescript
router.get('/Inventory/sync', authMiddleware, InventoryController.getInventory);
router.post('/Inventory/equip', authMiddleware, InventoryController.setEquipState);
router.post('/Inventory/move', authMiddleware, InventoryController.moveItem);
router.post('/Inventory/add', authMiddleware, InventoryController.addItem);
router.post('/Inventory/remove', authMiddleware, InventoryController.removeItem);
router.post('/Storage/deposit', authMiddleware, StorageController.depositItem);
router.post('/Storage/withdraw', authMiddleware, StorageController.withdrawItem);
```

→ 7 endpoint cho inventory: sync, equip, move, add, remove, deposit, withdraw.

**Business code — Thêm item (có anti-cheat):**

```typescript
public static async addItem(req: Request, res: Response): Promise<void> {
    const accountId = (req as any).user.accountId;
    const data = req.body as AddItemRequestDTO;

    if (data.isStackable) {
        const existing = await repo.findOne({
            where: { accountId, itemId, chestId: IsNull(), slotIndex: LessThan(2000) }
        });
        if (existing) {
            existing.quantity += quantity;
            await repo.save(existing);
            res.status(200).json({ success: true, action: "stacked" });
            return;
        }
    }
    // Anti-cheat: xác thực seed ngẫu nhiên
    if (data.validationSeed && data.validationSeed > 0) {
        const rng = new SeededRandom(data.validationSeed);
        const expectedRarity = ItemGenerationHelper.getRandomRarity(rng);
        if (expectedRarity !== finalRarity) {
            res.status(403).json({ success: false, message: "Dữ liệu không hợp lệ!" });
            return;
        }
    }
    await repo.save(newItem);
    res.status(200).json({ success: true, dbId: newItem.id, action: "created" });
}
```

→ Kiểm tra item stackable (cộng dồn số lượng), anti-cheat bằng validationSeed, tạo item mới nếu chưa có.

**Business code — Move/Swap item (dùng transaction):**

```typescript
public static async moveItem(req: Request, res: Response): Promise<void> {
    await ApplicationDbContext.transaction(async (manager) => {
        const sourceItem = await manager.findOne(UserItem, { where: { id, accountId } });
        const targetItem = await manager.findOne(UserItem, { where: { accountId, slotIndex } });

        if (!targetItem) {
            sourceItem.slotIndex = newSlotIndex;
            await manager.save(sourceItem);
        } else if (isStackable && targetItem.itemId === sourceItem.itemId) {
            targetItem.quantity += sourceItem.quantity;
            await manager.save(targetItem);
            await manager.remove(sourceItem);
        } else {
            // Hoán đổi slot
            const oldSlot = sourceItem.slotIndex;
            targetItem.slotIndex = oldSlot;
            sourceItem.slotIndex = newSlotIndex;
            await manager.save([sourceItem, targetItem]);
        }
    });
    res.status(200).json({ success: true });
}
```

→ Dùng transaction để move/swap slot: nếu ô trống thì chuyển vào, nếu trùng item stackable thì gộp, nếu không thì hoán đổi slot.

---

### lambda-economy — Quản lý tiền tệ Coin/Gem (Core Service)

**Routes:**

```typescript
router.get('/Economy/balance', authMiddleware, EconomyController.getBalance);
router.post('/Economy/spend', authMiddleware, EconomyController.spendCurrency);
router.post('/Economy/earn', authMiddleware, EconomyController.earnCurrency);
```

→ 3 endpoint cho economy: xem balance, tiêu tiền, nhận tiền.

**Business code — Spend (dùng transaction):**

```typescript
public static async spendCurrency(req: Request, res: Response): Promise<void> {
    const accountId = (req as any).user.accountId;
    const amount = Number(req.body.amount);
    const type = req.body.currencyType?.toLowerCase();
    const isCoin = type === "coin";
    const isGem = type === "gem";

    await ApplicationDbContext.manager.transaction(async (manager) => {
        const wallet = await getOrCreateWallet(accountId);
        let success = false;
        if (isCoin && wallet.coin >= amount) { wallet.coin -= amount; success = true; }
        else if (isGem && wallet.gem >= amount) { wallet.gem -= amount; success = true; }
        if (!success) {
            res.status(200).json({ success: false, message: "Không đủ tiền." });
            return;
        }
        wallet.updatedAt = TimeHelper.getVietnamTime();
        await manager.save(wallet);
        res.status(200).json({ success: true, newBalance: isCoin ? wallet.coin : wallet.gem });
    });
}
```

→ Trong transaction: kiểm tra số dư, trừ coin hoặc gem, cập nhật thời gian, trả về số dư mới.

---

### lambda-transaction — Shop & Gift Code (Dependent Service)

**Routes:**

```typescript
router.get('/Shop/items', authMiddleware, ShopController.getShopItems);
router.post('/Shop/buy', authMiddleware, ShopController.buyItem);
router.get('/GiftCodes/live', authMiddleware, GiftCodeController.getLiveGiftCodes);
router.post('/GiftCodes/redeem', authMiddleware, GiftCodeController.redeem);
```

→ 4 endpoint cho shop và gift code: xem danh sách item, mua item, xem gift code đang live, redeem gift code.

**Business code — Mua item (trừ tiền + thêm item trong 1 transaction):**

```typescript
public static async buyItem(req: Request, res: Response): Promise<void> {
    const accountId = (req as any).user.accountId;
    const { itemId, quantity } = req.body;

    const itemDef = await manager.findOne(ItemDef, { where: { id: itemId } });
    const totalCost = itemDef.buyPrice * quantity;
    const userCurrency = await manager.findOne(UserCurrency, { where: { accountId } });

    if (currency === 'Coin' && userCurrency.coin < totalCost) {
        res.status(400).json({ message: 'Not enough Coin' }); return;
    }

    await ApplicationDbContext.manager.transaction(async (manager) => {
        await manager.save(userCurrency); // trừ tiền

        if (itemDef.isStackable) {
            const stackItem = existingItems.find(i => i.itemId === itemId);
            if (stackItem) { stackItem.quantity += quantity; await manager.save(stackItem); }
            else { await createNewItem(manager, accountId, itemDef, quantity, occupiedSlots); }
        } else {
            for (let i = 0; i < quantity; i++)
                await createNewItem(manager, accountId, itemDef, 1, occupiedSlots);
        }
    });
    // Ghi log giao dịch
    const log = new ShopLog();
    log.accountId = accountId; log.itemId = itemId;
    log.totalCost = totalCost;
    await ApplicationDbContext.manager.save(log);

    res.status(200).json(inventory);
}
```

→ Trong transaction: kiểm tra số dư, trừ tiền, thêm item (stackable gộp, non-stackable tạo riêng), ghi log giao dịch.

---

### Tổng quan phụ thuộc

- **lambda-auth**, **lambda-inventory**, **lambda-economy** là *Core Services* — không phụ thuộc service nào khác, deploy song song.
- **lambda-transaction** là *Dependent Service* — cần gọi Economy (trừ tiền) và Inventory (thêm item), deploy sau.

---

## 3. Kiến Trúc Xử Lý Đồng Bộ

### 3.1. Setup Lambda service (serverless.yml)

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
    # KHÔNG có events — API Gateway tập trung gọi qua Lambda Integration

plugins:
  - serverless-esbuild

custom:
  esbuild:
    bundle: true
    minify: false
    target: node20
```

→ Cấu hình serverless.yml cho mỗi Lambda service: runtime node20, IAM role cho RDS IAM Auth, dùng serverless-esbuild để bundle TypeScript thành 1 file. **Không có `events`** vì API Gateway tập trung gọi qua Lambda Integration.

### 3.2. Setup API Gateway tập trung

### 3.2. Setup API Gateway tập trung

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

    # --- Auth Integration ---
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

    # --- Inventory Integration ---
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

    # --- Economy Integration ---
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

    # --- Transaction Integration ---
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

### 3.3. API Gateway Routing

API Gateway tập trung định tuyến dựa trên path pattern:

- `ANY /Accounts/{proxy+}` → `gameapi-auth-dev-api`
- `ANY /Inventory/{proxy+}` → `gameapi-inventory-dev-api`
- `ANY /Storage/{proxy+}` → `gameapi-inventory-dev-api`
- `ANY /Economy/{proxy+}` → `gameapi-economy-dev-api`
- `ANY /Shop/{proxy+}` → `gameapi-transaction-dev-api`
- `ANY /GiftCodes/{proxy+}` → `gameapi-transaction-dev-api`

Mỗi Lambda được deploy **không kèm API Gateway riêng** — chỉ là function + handler. API Gateway là điểm vào duy nhất.

---

## 4. Cross-Service Communication (Đồng Bộ)

Khi một Lambda cần gọi service khác, nó gọi qua API Gateway (không gọi trực tiếp Lambda). Dependent service cần được setup thêm `API_GATEWAY_URL` trong env.

### Setup env cho Dependent Service

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
    API_GATEWAY_URL: 'https://{api-id}.execute-api.ap-southeast-1.amazonaws.com'  # << Cần setup
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

→ Dependent service cần thêm `API_GATEWAY_URL` trong `environment` để biết đường gọi cross-service. Các service khác giữ nguyên config, chỉ thêm biến này.

### Code cross-service call (ApiClient)

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

### Saga Pattern — Rollback khi fail

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

## 5. Database Access Pattern

Mỗi controller dùng TypeORM repository trực tiếp, query nhỏ gọn trong transaction:

### Auth — SELECT + INSERT trong transaction

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

### Inventory — SELECT + UPDATE/SWAP (hoán đổi slot)

```typescript
const repo = ApplicationDbContext.getRepository(UserItem);
const item = await repo.findOne({ where: { id: data.itemDbId, accountId } });
item.isEquipped = data.isEquipped;
await repo.save(item);
```

→ Tìm item theo id và accountId, cập nhật trạng thái equip, save lại. Query đơn giản, chỉ 1-2 dòng.

### Economy — SELECT balance → UPDATE trong transaction

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

→ Trong transaction: kiểm tra số dư, trừ coin/gem, cập nhật thời gian. Trả về số dư mới sau khi spend.

### Transaction — SELECT + UPDATE + INSERT trong 1 transaction

```typescript
await ApplicationDbContext.manager.transaction(async (manager) => {
    await manager.save(userCurrency);       // UPDATE: trừ tiền
    await manager.save(newItem);            // INSERT: thêm item vào inventory
});
```

→ Trong 1 transaction: UPDATE số dư (trừ tiền) + INSERT item mới. Đảm bảo cả 2 cùng thành công hoặc cùng rollback.

**Đặc điểm:** Mỗi request chỉ đọc/ghi 1-3 dòng. Index đầy đủ trên `accountId`, `itemId`. Query ngắn, không JOIN phức tạp. Response time target < 500ms. Không batch write.

---

## 6. Deploy Script

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

### Lambda handler entry point

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

---

## 7. Tổng Kết

- **Response:** Đồng bộ, < 500ms
- **Các service:** Auth, Inventory, Economy, Transaction
- **DB Query:** Nhỏ, 1-3 dòng, có index
- **Cross-service:** Qua API Gateway, dùng Saga rollback
- **Scale:** Mỗi service scale độc lập
- **Deploy thứ tự:** Core trước → Dependent sau

**Key Takeaways:**

- Client cần response ngay → không thể dùng async
- Mỗi service chỉ lo 1 việc, dễ maintain
- Saga pattern đảm bảo consistency giữa các service
- API Gateway tập trung giúp routing đơn giản, cross-service gọi qua HTTP
