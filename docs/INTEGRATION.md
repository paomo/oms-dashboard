# OMS 到飞书自动同步系统 - 完整集成文档

## 系统概述

已完成 OMS 系统到飞书多维表格的自动同步功能，支持三个数据集：
1. **出库明细** (outbound)
2. **库存快照** (inventory_snapshot)
3. **待入库单** (inbound_pending)

## 项目结构

```
~/.hermes/projects/oms-feishu-sync/
├── src/oms_feishu_sync/
│   ├── transform/                    # 数据转换模块
│   │   ├── __init__.py
│   │   ├── outbound.py              # 出库明细转换
│   │   ├── inventory.py             # 库存快照转换
│   │   └── inbound.py               # 待入库单转换
│   ├── oms_export_playwright.py     # Playwright 自动导出
│   ├── sync_outbound_once.py        # 出库明细同步
│   ├── sync_inventory_once.py       # 库存快照同步
│   └── sync_inbound_once.py         # 待入库单同步
├── config/
│   ├── warehouses.yaml              # 仓库配置
│   └── export_rules.yaml            # 导出规则
├── run_daily_sync.py                # 主同步脚本
├── run_daily_sync.sh                # Shell 包装脚本
├── test_system_integration.py       # 完整性测试
└── .env                             # 环境变量
```

## 数据集配置

### 1. 出库明细 (outbound)
- **飞书表 ID**: `tblQFkMJYrCXYWV6`
- **回溯天数**: 3 天
- **需要日期范围**: 是
- **去重键**: `{仓库代码}_{出库单号}_{SKU}_{批次号}`

### 2. 库存快照 (inventory_snapshot)
- **飞书表 ID**: `tbl3CjadBA7muybB`
- **回溯天数**: 1 天（仅用于日志）
- **需要日期范围**: 否（导出全量数据）
- **去重键**: `{仓库代码}_{SKU}_{快照时间}`

### 3. 待入库单 (inbound_pending)
- **飞书表 ID**: `tbl5r19CR5nNIRlU`
- **回溯天数**: 7 天（仅用于日志）
- **需要日期范围**: 否（导出全量数据）
- **去重键**: `{仓库代码}_{入库单号}_{SKU}_{批次号}`

## 仓库配置

当前配置了 5 个仓库：

| 仓库代码 | 名称 | 账号 | 状态 |
|---------|------|------|------|
| TEST | Dmile | Robot | ✓ 启用 |
| ZA1009 | ZA1009仓库 | ZA1009 | ✓ 启用 |
| OGR1808 | OGR1808仓库 | OGR1808 | ✓ 启用 |
| 92005 | 92005仓库 | 92005 | ✓ 启用 |
| YL888 | YL888仓库 | YL888 | ✓ 启用 |

## 使用方法

### 1. 测试单个数据集

```bash
# 测试出库明细
python run_daily_sync.py --dataset outbound

# 测试库存快照
python run_daily_sync.py --dataset inventory_snapshot

# 测试待入库单
python run_daily_sync.py --dataset inbound_pending
```

### 2. 测试单个仓库的导出

```bash
# 出库明细（需要日期范围）
python src/oms_feishu_sync/oms_export_playwright.py \
  --warehouse TEST \
  --dataset outbound \
  --query-start 2026-04-29 \
  --query-end 2026-05-02

# 库存快照（不需要日期范围）
python src/oms_feishu_sync/oms_export_playwright.py \
  --warehouse TEST \
  --dataset inventory_snapshot

# 待入库单（不需要日期范围）
python src/oms_feishu_sync/oms_export_playwright.py \
  --warehouse TEST \
  --dataset inbound_pending
```

### 3. 完整同步流程

```bash
# 使用 Shell 脚本（同步所有三个数据集）
./run_daily_sync.sh
```

### 4. 定时任务

当前定时任务配置：
- **Job ID**: `b2a140204854`
- **频率**: 每 6 小时 (0:00, 6:00, 12:00, 18:00)
- **工作目录**: `/Users/macpro/.hermes/projects/oms-feishu-sync`
- **同步数据集**: 所有三个（outbound, inventory_snapshot, inbound_pending）

查看定时任务：
```bash
hermes cronjob list
```

手动触发：
```bash
hermes cronjob run b2a140204854
```

## 数据流程

### 出库明细流程
1. **导出**: Playwright 自动登录 OMS → 筛选日期范围 → 导出 Excel
2. **转换**: `transform/outbound.py` 将 Excel 转换为飞书格式
3. **去重**: 查询飞书表中已有的 `dedupe_key`
4. **上传**: 批量上传新记录（200 条/批）

### 库存快照流程
1. **导出**: Playwright 自动登录 OMS → 点击"产品库存" → 导出全部数据
2. **转换**: `transform/inventory.py` 将 Excel 转换为飞书格式，添加快照时间
3. **去重**: 查询飞书表中已有的 `dedupe_key`
4. **上传**: 批量上传新记录（200 条/批）

### 待入库单流程
1. **导出**: Playwright 自动登录 OMS → 点击"待入库"标签 → 导出全部数据
2. **转换**: `transform/inbound.py` 将 Excel 转换为飞书格式
3. **去重**: 查询飞书表中已有的 `dedupe_key`
4. **上传**: 批量上传新记录（200 条/批）

## 关键特性

### 1. 智能去重
- 基于 `dedupe_key` 字段自动去重
- 避免重复上传相同数据
- 支持增量同步

### 2. 多仓库支持
- 从 `config/warehouses.yaml` 读取仓库配置
- 自动遍历所有启用的仓库
- 每个仓库独立同步

### 3. 灵活的日期范围
- 出库明细：支持自定义日期范围
- 库存快照：导出全量数据（快照模式）
- 待入库单：导出全量数据（状态筛选）

### 4. 错误处理
- 每个仓库独立处理，失败不影响其他仓库
- 详细的日志记录
- 失败重试机制

### 5. 批量上传
- 每批 200 条记录
- 避免 API 限制
- 提高上传效率

## 测试验证

运行完整性测试：
```bash
python test_system_integration.py
```

测试结果：
- ✓ 模块导入
- ✓ 同步脚本
- ✓ 配置文件
- ✓ 主脚本

## 下一步计划

### 1. 修复已知问题
- [ ] TEST 仓库：未找到下载按钮
- [ ] ZA1009 仓库：缺少环境变量
- [ ] 92005 仓库：字符串拼接类型错误

### 2. 测试库存快照和待入库单
- [ ] 测试 Playwright 导出功能
- [ ] 验证数据转换正确性
- [ ] 测试飞书上传功能

### 3. 优化和增强
- [ ] 添加数据验证规则
- [ ] 实现增量更新（而非仅追加）
- [ ] 添加数据统计和报表
- [ ] 实现异常告警机制

## 故障排查

### 问题 1: 导出失败
**症状**: `RuntimeError: 未找到下载按钮`
**原因**: 页面结构变化或选择器不匹配
**解决**: 检查 `config/export_rules.yaml` 中的 `download_texts` 配置

### 问题 2: 环境变量缺失
**症状**: `RuntimeError: 仓库 XXX 缺少凭据环境变量`
**原因**: `.env` 文件中缺少对应仓库的账号密码
**解决**: 添加 `OMS_ACCOUNT_{仓库代码}_USERNAME` 和 `OMS_ACCOUNT_{仓库代码}_PASSWORD`

### 问题 3: 类型错误
**症状**: `sequence item 3: expected str instance, int found`
**原因**: 字符串拼接时混入了整数类型
**解决**: 检查数据转换脚本，确保所有字段都转换为字符串

## 维护指南

### 添加新仓库
1. 在 `.env` 中添加账号密码
2. 在 `config/warehouses.yaml` 中添加仓库配置
3. 测试导出和同步功能

### 添加新数据集
1. 在 `config/export_rules.yaml` 中添加导出规则
2. 创建数据转换脚本 `src/oms_feishu_sync/transform/{dataset}.py`
3. 创建同步脚本 `src/oms_feishu_sync/sync_{dataset}_once.py`
4. 更新 `run_daily_sync.py` 中的 `DATASETS` 配置
5. 更新 `run_daily_sync.sh` 添加新数据集的同步命令

### 日志管理
- 日志位置: `logs/sync_*.log`
- 自动清理: 保留最近 30 天
- 手动查看: `tail -f logs/sync_*.log`

## 联系方式

如有问题，请查看：
- 项目文档: `~/.hermes/projects/oms-feishu-sync/README.md`
- 配置文件: `~/.hermes/projects/oms-feishu-sync/config/`
- 日志文件: `~/.hermes/projects/oms-feishu-sync/logs/`
