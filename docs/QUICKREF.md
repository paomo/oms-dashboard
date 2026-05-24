# OMS 到飞书同步系统 - 快速参考

## 🎯 系统状态

✅ **已完成集成**
- ✓ 出库明细 (outbound)
- ✓ 库存快照 (inventory_snapshot)  
- ✓ 待入库单 (inbound_pending)

✅ **完整性测试**: 全部通过

## 📊 数据集概览

| 数据集 | 表 ID | 需要日期 | 去重键 |
|--------|-------|---------|--------|
| 出库明细 | tblQFkMJYrCXYWV6 | ✓ | 仓库_单号_SKU_批次 |
| 库存快照 | tbl3CjadBA7muybB | ✗ | 仓库_SKU_时间 |
| 待入库单 | tbl5r19CR5nNIRlU | ✗ | 仓库_单号_SKU_批次 |

## 🏭 仓库配置

5 个仓库已配置：TEST, ZA1009, OGR1808, 92005, YL888

## 🚀 快速命令

### 测试单个数据集
```bash
cd ~/.hermes/projects/oms-feishu-sync
source .venv/bin/activate

# 增量同步（推荐）
python run_daily_sync_v2.py --dataset outbound
python run_daily_sync_v2.py --dataset inventory_snapshot
python run_daily_sync_v2.py --dataset inbound_pending

# 强制全量
python run_daily_sync_v2.py --dataset outbound --force-full
```

### 完整同步（所有数据集）
```bash
cd ~/.hermes/projects/oms-feishu-sync
source .venv/bin/activate
python run_daily_sync_v2.py
```

### 测试单个仓库导出
```bash
# 库存快照（推荐先测试这个）
python src/oms_feishu_sync/oms_export_playwright.py \
  --warehouse TEST \
  --dataset inventory_snapshot

# 待入库单
python src/oms_feishu_sync/oms_export_playwright.py \
  --warehouse TEST \
  --dataset inbound_pending
```

## ⏰ 定时任务

- **Job ID**: b2a140204854
- **频率**: 每 6 小时
- **命令**: `./run_daily_sync.sh`

```bash
# 查看任务
hermes cronjob list

# 手动触发
hermes cronjob run b2a140204854
```

## 🔍 故障排查

### 查看日志
```bash
# 最新日志
ls -lt logs/sync_*.log | head -1

# 实时查看
tail -f logs/sync_*.log
```

### 常见问题

**未找到下载按钮**
→ 检查 `config/export_rules.yaml` 中的 `download_texts`

**缺少环境变量**
→ 在 `.env` 中添加 `OMS_ACCOUNT_{仓库}_USERNAME/PASSWORD`

**类型错误**
→ 检查数据转换脚本，确保字段类型正确

## 📁 关键文件

```
~/.hermes/projects/oms-feishu-sync/
├── run_daily_sync_v2.py       # 主脚本（增量同步+重试）
├── run_daily_sync.py          # [已废弃] 旧版编排层
├── run_daily_sync.sh          # Shell 包装
├── config/
│   ├── warehouses.yaml        # 仓库配置
│   └── export_rules.yaml      # 导出规则
├── .env                       # 环境变量
└── logs/                      # 日志目录
```

## 🎓 下一步

1. **测试库存快照导出**
   ```bash
   python src/oms_feishu_sync/oms_export_playwright.py \
     --warehouse TEST --dataset inventory_snapshot
   ```

2. **测试待入库单导出**
   ```bash
   python src/oms_feishu_sync/oms_export_playwright.py \
     --warehouse TEST --dataset inbound_pending
   ```

3. **完整同步测试**
   ```bash
   ./run_daily_sync.sh
   ```

4. **修复已知问题**
   - TEST: 下载按钮选择器
   - ZA1009: 环境变量
   - 92005: 类型转换

## 📚 详细文档

- 完整集成文档: `INTEGRATION.md`
- 项目说明: `README.md`
- 配置说明: `config/README.md`
