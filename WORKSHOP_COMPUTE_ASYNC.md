# Workshop: Game Services — Progression, Loot/Reward

**Mục tiêu:** Trình bày 2 Lambda service xử lý chỉ số nhân vật, Farm, Leaderboard, Forum, Save/Load.

> **Ghi chú:** Cơ chế async (SQS FIFO, DLQ, batch write) sẽ được trình bày trong workshop riêng.

---

## 1. Giới thiệu

Hai service này hiện hoạt động đồng bộ như Compute Sync, nhưng có đặc thù riêng:

- **lambda-progression-world:** Quản lý chỉ số nhân vật (stats, level, exp) + Farm (trồng, thu hoạch)
- **lambda-loot-reward:** Leaderboard, Forum, GameData Save/Load

---

## 2. Các Module

### lambda-progression-world — Chỉ số nhân vật & Farm

**Routes:**
```typescript
router.get('/PlayerStats/profile', authMiddleware, PlayerStatsController.getProfile);
router.post('/PlayerStats/distribute', authMiddleware, PlayerStatsController.distributePoints);
router.post('/PlayerStats/reset', authMiddleware, PlayerStatsController.resetStats);
router.post('/PlayerStats/add-exp', authMiddleware, PlayerStatsController.addExperience);
router.get('/Farm/sync', authMiddleware, FarmController.syncFarm);
router.post('/Farm/plant', authMiddleware, FarmController.plantSeed);
router.post('/Farm/harvest', authMiddleware, FarmController.harvestCrop);
```
→ 7 endpoint cho Progression: profile stats, phân bổ điểm, reset stats, cộng exp, sync farm, trồng, thu hoạch.

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
→ Kiểm tra đủ điểm tiềm năng + anti-cheat tổng điểm không vượt mức cho phép theo cấp độ, sau đó cập nhật stats và save.

**Business code — Plant seed (ghi ngay vào DB):**
```typescript
public static async plantSeed(req: Request, res: Response): Promise<void> {
    const accountId = (req as any).user.accountId;
    const data = req.body as PlantSeedRequestDTO;
    const repo = ApplicationDbContext.getRepository(FarmPlot);
    let plot = await repo.findOne({ where: { accountId, plotId: data.plotId } });
    if (plot) {
        plot.seedItemId = data.seedItemId;
        plot.plantedAt = TimeHelper.getVietnamTime();
    } else {
        plot = new FarmPlot();
        plot.accountId = accountId;
        plot.plotId = data.plotId;
        plot.seedItemId = data.seedItemId;
        plot.plantedAt = TimeHelper.getVietnamTime();
    }
    await repo.save(plot);
    res.status(200).json({ success: true, plantedAt: plot.plantedAt });
}
```
→ Nếu plot đã tồn tại thì cập nhật seedItemId + plantedAt, nếu chưa thì tạo mới. Ghi trực tiếp vào DB.

**Business code — Harvest crop (gom nhóm, xử lý hàng loạt):**
```typescript
public static async harvestCrop(req: Request, res: Response): Promise<void> {
    const accountId = (req as any).user.accountId;
    const data = req.body as BulkHarvestRequestDTO;
    const plots = await farmRepo.find({
        where: { accountId, plotId: In(data.plotIds) }
    });
    // Gom nhóm vật phẩm thu hoạch, cộng exp
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
    // Thêm item thu hoạch vào inventory
    for (const [itemId, qty] of Object.entries(harvestedItems)) {
        await callService(API_GATEWAY_URL, '/Inventory/add', { itemId, quantity: qty }, token);
    }
    res.status(200).json({ harvestedItems, totalExp });
}
```
→ Gom nhóm plots đã đủ thời gian thu hoạch, tính harvestedItems + exp, xóa plots cũ, thêm item vào inventory qua cross-service call.

---

### lambda-loot-reward — Leaderboard, Forum, Save/Load

**Routes:**
```typescript
router.get('/Leaderboard', authMiddleware, LeaderboardController.getLeaderboard);
router.post('/Forum/Create', authMiddleware, ForumController.createThread);
router.post('/Forum/Reply', authMiddleware, ForumController.replyThread);
router.get('/GameData/get-save', authMiddleware, GameDataController.getSaveData);
router.post('/GameData/save-data', authMiddleware, GameDataController.saveData);
```
→ 5 endpoint cho Loot/Reward: leaderboard, forum CRUD, game data save/load.

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
→ Nếu đã có save data thì cập nhật, nếu chưa thì tạo mới. Dữ liệu được lưu dưới dạng JSON string.

---

## 3. Database Access Pattern

### Progression — UPDATE level + exp
```typescript
// Cộng exp, tự động level up
stats.exp += amount;
while (true) {
    const expToNext = GameLogicValidator.getExpToNextLevel(stats.level);
    if (stats.exp >= expToNext) {
        stats.exp -= expToNext;
        stats.level++;
        stats.potentialPoints += 5; // 5 điểm tiềm năng mỗi level
    } else break;
}
await ApplicationDbContext.getRepository(UserStat).save(stats);
```
→ Cộng exp, kiểm tra và tự động level up (cộng 5 điểm tiềm năng mỗi level), save stats.

### Farm — INSERT/REMOVE plots hàng loạt
```typescript
// Gom nhóm plots thu hoạch, xóa hàng loạt
const plotsToRemove = plots.filter(p => elapsed >= config.growthTime);
await farmRepo.remove(plotsToRemove);
// Thêm item vào inventory qua cross-service call
for (const [itemId, qty] of Object.entries(harvestedItems)) {
    await callService(API_GATEWAY_URL, '/Inventory/add', { itemId, quantity: qty }, token);
}
```
→ Lọc các plots đã đủ thời gian thu hoạch, xóa hàng loạt, gọi Inventory/add qua cross-service API.

### Leaderboard — SELECT + tính toán
```typescript
// Sắp xếp người chơi theo level giảm dần
const allStats = await repo.find({ order: { level: 'DESC' }, take: 100 });
const leaderboard = allStats.map((stat, index) => ({
    rank: index + 1,
    accountId: stat.accountId,
    level: stat.level,
    exp: stat.exp,
}));
```
→ Sắp xếp top 100 người chơi theo level giảm dần, trả về kèm rank.

---

## 4. API Gateway Routing

Các route đã được định nghĩa trong `api-gateway/serverless.yml`:

| Path pattern | Lambda đích |
|---|---|
| `ANY /PlayerStats/{proxy+}` | `gameapi-progression-world-dev-api` |
| `ANY /Farm/{proxy+}` | `gameapi-progression-world-dev-api` |
| `ANY /Leaderboard/{proxy+}` | `gameapi-loot-reward-dev-api` |
| `ANY /Forum/{proxy+}` | `gameapi-loot-reward-dev-api` |
| `ANY /GameData/{proxy+}` | `gameapi-loot-reward-dev-api` |
| `ANY /SaveData/{proxy+}` | `gameapi-loot-reward-dev-api` |

---

## 5. Deploy

```bash
# Deploy song song cả 2 service
deploy_service "progression-world" &
deploy_service "loot-reward" &
wait
```
→ Deploy cả 2 service song song. API Gateway route đã được định nghĩa sẵn trong `api-gateway/serverless.yml`.

---

## 6. Tổng Kết

- **lambda-progression-world:** Chỉ số nhân vật (stats, level, exp), Farm (plant, harvest)
- **lambda-loot-reward:** Leaderboard, Forum, GameData Save/Load
- Cả 2 hiện hoạt động đồng bộ, gọi DB trực tiếp
- Cross-service call qua API Gateway (vd: harvest → Inventory/add)

**Key Takeaways:**
- Progression + Loot là 2 service độc lập, deploy riêng
- Farm harvest gọi Inventory qua API Gateway để thêm vật phẩm
- Database query đơn giản, chủ yếu SELECT/UPDATE theo accountId
- Anti-cheat tích hợp sẵn trong distribute points

---

👉 Workshop riêng về async infrastructure (SQS FIFO, DLQ, batch write) sẽ được viết sau
