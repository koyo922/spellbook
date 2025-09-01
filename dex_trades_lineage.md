# DEX Trades 表数据血缘关系

## 概述
`dex.trades` 表是一个汇总了所有去中心化交易所（DEX）交易数据的核心表，通过多层数据处理和丰富化流程，将原始的区块链交易数据转换为标准化的交易记录。

## 主要数据流

### 1. 数据源层级结构

```
dex.trades (最终表)
├── balancer_v3 (CTE) - Balancer V3 特殊处理
├── dexs (CTE) - 其他所有 DEX 项目
└── as_is_dexs (CTE) - 保持原有血缘关系的项目
```

### 2. 三个主要数据分支

#### 分支 1: Balancer V3 (特殊处理)
- **数据源**: `ref('dex_base_trades')`
- **过滤条件**: `project = 'balancer' AND version = '3'`
- **处理宏**: `enrich_balancer_v3_dex_trades()`
- **特殊性**: 处理 ERC4626 代币，需要特殊的价格计算逻辑

#### 分支 2: 其他 DEX 项目
- **数据源**: `ref('dex_base_trades')`
- **过滤条件**: `NOT (project = 'balancer' AND version = '3')`
- **处理宏**: `enrich_dex_trades()`
- **功能**: 标准的 DEX 交易数据丰富化

#### 分支 3: 保持原有血缘关系的项目
- **数据源**: 
  - `ref('oneinch_lop_own_trades')` - 1inch LOP 交易
  - `ref('zeroex_native_trades')` - 0x 原生交易
- **特殊性**: 这些项目有自己的完整数据处理流程

## 核心依赖组件

### 1. dex_base_trades (基础数据源)
- **位置**: `dbt_subprojects/dex/models/trades/dex_base_trades.sql`
- **功能**: 汇总所有区块链的基础交易数据
- **结构**: 通过 UNION ALL 合并多个区块链的数据

#### 支持的区块链 (47个):
```
arbitrum, avalanche_c, abstract, base, berachain, blast, bnb, boba, 
celo, corn, ethereum, fantom, flare, gnosis, hemi, ink, linea, kaia, 
katana, mantle, nova, opbnb, optimism, plume, polygon, ronin, scroll, 
sei, shape, sonic, sophon, superseed, taiko, unichain, worldchain, 
zkevm, zksync, zora
```

#### 每个区块链的数据来源:
- **格式**: `dex_{blockchain}_base_trades`
- **示例**: `dex_ethereum_base_trades`, `dex_base_base_trades`
- **内容**: 每个区块链汇总了该链上所有 DEX 协议的交易

### 2. 数据丰富化宏

#### enrich_dex_trades()
- **位置**: `dbt_subprojects/dex/macros/models/enrich_dex_trades.sql`
- **功能**:
  - 从 `tokens.erc20` 获取代币元数据 (symbol, decimals)
  - 计算代币数量 (从 raw 转换为实际数量)
  - 生成代币对名称
  - 通过 `add_amount_usd()` 添加 USD 价值

#### enrich_balancer_v3_dex_trades()
- **位置**: `dbt_subprojects/dex/macros/models/enrich_balancer_v3_dex_trades.sql`
- **功能**: 
  - 类似 `enrich_dex_trades()` 但专门处理 ERC4626 代币
  - 使用 `balancer_v3.erc4626_token_prices` 获取特殊价格数据

#### add_amount_usd()
- **位置**: `dbt_subprojects/dex/macros/add_amount_usd.sql`
- **功能**:
  - 从 `prices.usd_with_native` 获取代币价格
  - 使用 `prices.trusted_tokens` 进行价格验证
  - 计算交易的 USD 价值

### 3. 外部数据源

#### tokens.erc20
- **用途**: 提供代币的基础信息
- **字段**: blockchain, contract_address, symbol, decimals

#### prices.usd_with_native
- **用途**: 提供代币的 USD 价格数据
- **字段**: blockchain, contract_address, minute, price

#### prices.trusted_tokens
- **用途**: 标识可信的代币，用于价格验证

#### balancer_v3.erc4626_token_prices
- **用途**: 专门为 Balancer V3 的 ERC4626 代币提供价格数据

## 数据处理流程

### 1. 基础数据收集
```
区块链原始事件 → 各协议的 base_trades → 区块链级别汇总 → dex_base_trades
```

### 2. 数据丰富化
```
dex_base_trades → 添加代币元数据 → 计算数量 → 添加价格 → 最终交易记录
```

### 3. 最终合并
```
balancer_v3 + dexs + as_is_dexs → UNION ALL → dex.trades
```

## 配置特性

### 表配置
- **分区**: `['block_month', 'blockchain', 'project']`
- **物化方式**: `incremental` (增量更新)
- **文件格式**: `delta`
- **合并策略**: `merge`
- **唯一键**: `['blockchain', 'project', 'version', 'tx_hash', 'evt_index']`

### 支持的区块链 (36个)
```
arbitrum, avalanche_c, base, blast, bnb, boba, celo, ethereum, 
fantom, gnosis, kaia, linea, mantle, nova, optimism, polygon, 
ronin, scroll, sei, sonic, sophon, taiko, zkevm, zksync, unichain, zora
```

## 特殊项目处理

### 1inch LOP 交易
- **数据源**: `oneinch_evm_swaps`
- **过滤**: LOP 协议，非 fusion，非 second_side，非跨链
- **特点**: 保持原有的完整数据处理流程

### 0x 原生交易
- **数据源**: `zeroex_native_fills`
- **特点**: 直接通过 0x 交易合约的原生交易（非 API）

## 数据质量保证

### 增量更新
- 使用 `incremental_predicate('block_time')` 确保只处理新数据
- 所有相关的价格数据也会同步进行增量更新

### 数据验证
- 通过 trusted_tokens 验证价格数据的可靠性
- 使用唯一键约束防止重复数据

## 维护者
- hosuke, 0xrob, jeff-dude, tomfutago, viniabussafi

---

这个数据血缘关系展示了 `dex.trades` 表如何从原始的区块链事件数据，通过多层处理和丰富化，最终形成标准化的 DEX 交易数据表。整个流程涵盖了 40+ 个区块链和数百个 DEX 协议的数据整合。
