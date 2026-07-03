---
title : "Compute Async — Xử Lý Bất Đồng Bộ với SQS"
date : 2024-01-01
weight : 7
chapter : false
pre : " <b> 5.7. </b> "
---
#### 5.7.1 Các Module trong Compute Async

**lambda-progression-world** — Chỉ số nhân vật & Farm (Core Service):

- `GET /PlayerStats/profile`, `POST /PlayerStats/distribute`, `POST /PlayerStats/reset`, `POST /PlayerStats/add-exp`
- `GET /Farm/sync`, `POST /Farm/plant`, `POST /Farm/harvest`

**lambda-loot-reward** — Leaderboard, Forum, Save/Load (Core Service):

- `GET /Leaderboard`, `POST /Forum/Create`, `POST /Forum/Reply`
- `GET /GameData/get-save`, `POST /GameData/save-data`

**Business code — Distribute points (validation chặt):**

```typescript
public static async distributePoints(req: Request, res: Response): Promise<void> {
    const accountId = (req as any).user.accountId;
    const { str = 0, dex = 0, intStat = 0, con = 0 } = req.body;
    const totalCost = str + dex + intStat + con;
    const stats = await getOrCreateStats(accountId);
    if (stats.potentialPoints < totalCost) {
        res.status(400).json("Không đủ điểm tiềm năng."); return;
    }
    // Anti-cheat: kiểm tra tổng điểm không vượt mức cho phép
    const maxAllowedPoints = 5 + (stats.level - 1) * 5;
    const currentTotalPoints = stats.str + stats.dex + stats.int + stats.con + stats.potentialPoints;
    if (currentTotalPoints > maxAllowedPoints) {
        res.status(400).json("Phát hiện bất thường trong dữ liệu nhân vật!"); return;
    }
    stats.str += str; stats.dex += dex;
    stats.int += intStat; stats.con += con;
    stats.potentialPoints -= totalCost;
    await ApplicationDbContext.getRepository(UserStat).save(stats);
    res.status(200).json(stats);
}
```

Kiểm tra đủ điểm tiềm năng + anti-cheat tổng điểm không vượt mức cho phép theo cấp độ, sau đó cập nhật stats và save.

**Business code — Harvest crop (gom nhóm, xử lý hàng loạt):**

```typescript
public static async harvestCrop(req: Request, res: Response): Promise<void> {
    const accountId = (req as any).user.accountId;
    const data = req.body as BulkHarvestRequestDTO;
    const plots = await farmRepo.find({
        where: { accountId, plotId: In(data.plotIds) }
    });
    const harvestedItems: Record<number, number> = {};
    let totalExp = 0;
    for (const plot of plots) {
        const config = CropConfig[plot.seedItemId];
        const elapsed = (now.getTime() - plot.plantedAt.getTime()) / 1000;
        if (elapsed >= config.growthTime) {
            harvestedItems[config.harvestItemId] =
                (harvestedItems[config.harvestItemId] || 0) + config.harvestQuantity;
            totalExp += config.expReward;
            plotsToRemove.push(plot);
        }
    }
    await farmRepo.remove(plotsToRemove);
    for (const [itemId, qty] of Object.entries(harvestedItems)) {
        await callService(API_GATEWAY_URL, '/Inventory/add', { itemId, quantity: qty }, token);
    }
    res.status(200).json({ harvestedItems, totalExp });
}
```

Gom nhóm plots đã đủ thời gian thu hoạch, tính harvestedItems + exp, xóa plots cũ, thêm item vào inventory qua cross-service call.

**Business code — Save game data:**

```typescript
public static async saveData(req: Request, res: Response): Promise<void> {
    const accountId = (req as any).user.accountId;
    const { dataSave, timestamp } = req.body;
    const repo = ApplicationDbContext.getRepository(SaveData);
    let saveData = await repo.findOne({ where: { accountId } });
    if (saveData) {
        saveData.dataSave = JSON.stringify(dataSave);
        saveData.updatedAt = new Date(timestamp);
    } else {
        saveData = new SaveData();
        saveData.accountId = accountId;
        saveData.dataSave = JSON.stringify(dataSave);
    }
    await repo.save(saveData);
    res.status(200).json({ success: true, message: 'Game data saved.' });
}
```

Nếu đã có save data thì cập nhật, nếu chưa thì tạo mới. Dữ liệu được lưu dưới dạng JSON string.

#### 5.7.2 Database Access Pattern

##### Progression — UPDATE level + exp

```typescript
stats.exp += amount;
while (true) {
    const expToNext = GameLogicValidator.getExpToNextLevel(stats.level);
    if (stats.exp >= expToNext) {
        stats.exp -= expToNext;
        stats.level++;
        stats.potentialPoints += 5;
    } else break;
}
await ApplicationDbContext.getRepository(UserStat).save(stats);
```

Cộng exp, kiểm tra và tự động level up (cộng 5 điểm tiềm năng mỗi level), save stats.

##### Farm — INSERT/REMOVE plots hàng loạt

```typescript
const plotsToRemove = plots.filter(p => elapsed >= config.growthTime);
await farmRepo.remove(plotsToRemove);
for (const [itemId, qty] of Object.entries(harvestedItems)) {
    await callService(API_GATEWAY_URL, '/Inventory/add', { itemId, quantity: qty }, token);
}
```

Lọc các plots đã đủ thời gian thu hoạch, xóa hàng loạt, gọi Inventory/add qua cross-service API.

##### Leaderboard — SELECT + tính toán

```typescript
const allStats = await repo.find({ order: { level: 'DESC' }, take: 100 });
const leaderboard = allStats.map((stat, index) => ({
    rank: index + 1,
    accountId: stat.accountId,
    level: stat.level,
    exp: stat.exp,
}));
```

Sắp xếp top 100 người chơi theo level giảm dần, trả về kèm rank.

#### 5.7.3 API Gateway Routing

Các route đã được định nghĩa trong `api-gateway/serverless.yml`:

| Path pattern                  | Lambda đích                         |
| ----------------------------- | ------------------------------------- |
| `ANY /PlayerStats/{proxy+}` | `gameapi-progression-world-dev-api` |
| `ANY /Farm/{proxy+}`        | `gameapi-progression-world-dev-api` |
| `ANY /Leaderboard/{proxy+}` | `gameapi-loot-reward-dev-api`       |
| `ANY /Forum/{proxy+}`       | `gameapi-loot-reward-dev-api`       |
| `ANY /GameData/{proxy+}`    | `gameapi-loot-reward-dev-api`       |
| `ANY /SaveData/{proxy+}`    | `gameapi-loot-reward-dev-api`       |

---

#### 5.7.4 Deploy

```bash
# Deploy song song cả 2 service
deploy_service "progression-world" &
deploy_service "loot-reward" &
wait
```

Deploy cả 2 service song song. API Gateway route đã được định nghĩa sẵn trong `api-gateway/serverless.yml`.

---
