# PancakeSwap V2 BSC Complete Data Lineage

## Overview
Using PancakeSwap V2 as an example, this document demonstrates the complete processing flow of BSC DEX trading data from blockchain events to the final `dex.trades` table.
dex.trades doc: https://docs.dune.com/data-catalog/curated/dex-trades/evm/dex-trades#dex-trades

## Data Flow Overview

```mermaid
graph LR
    BSC["🔗 BSC Blockchain<br/>PancakeSwap V2 Contract Events"] 
    --> EXTRACT["📥 Data Extraction<br/>Parse Smart Contract Events"]
    --> PROCESS["⚙️ Data Processing<br/>Macro Processing + Aggregation"]
    --> ENRICH["💎 Data Enrichment<br/>Token Info + Price Data"]
    --> TRADES["📊 dex.trades<br/>Final Trading Table"]
    
    DEFINEDFI["🤖 DefinedFi<br/>Automated Token Data"]
    MANUAL["✍️ Manual Supplement<br/>tokens_bnb_bep20"]
    
    DEFINEDFI --> ENRICH
    MANUAL --> ENRICH
    
    style BSC fill:#e3f2fd
    style EXTRACT fill:#f3e5f5
    style PROCESS fill:#fff3e0
    style ENRICH fill:#e8f5e8
    style TRADES fill:#ffebee
    style DEFINEDFI fill:#fce4ec
    style MANUAL fill:#f1f8e9
```

## Complete Data Flow Diagram

```mermaid
graph TD
    subgraph "BSC Blockchain Layer"
        SWAP_EVENTS["PancakeSwap V2 Contract Events<br/>- PancakePair_evt_Swap<br/>- PancakeFactory_evt_PairCreated<br/>- PancakeSwapMMPool_evt_Swap<br/>- PancakeStableSwap_evt_TokenExchange"]
    end

    subgraph "Data Source Layer"
        PSV2_BNB["pancakeswap_v2_bnb source<br/>- PancakePair_evt_Swap<br/>- PancakeFactory_evt_PairCreated<br/>- PancakeSwapMMPool_evt_Swap<br/>- PancakeStableSwap_evt_TokenExchange"]
    end

    subgraph "Macro Processing Layer"
        UNISWAP_MACRO["uniswap_compatible_v2_trades macro<br/>- Process standard V2 AMM trades<br/>- Parse Swap events<br/>- Associate Factory pairing info"]
        
        MMPOOL_LOGIC["MMPool Logic<br/>- Process market maker pool trades<br/>- User vs Market Maker<br/>- Token address conversion"]
        
        STABLESWAP_LOGIC["StableSwap Logic<br/>- Process stablecoin swaps<br/>- TokenExchange events<br/>- Factory pairing association"]
    end

    subgraph "Platform-level Aggregation"
        PSV2_BASE["pancakeswap_v2_bnb_base_trades<br/>- Standard V2 trades<br/>- MMPool trades<br/>- StableSwap trades<br/>UNION ALL merge"]
    end

    subgraph "Chain-level Aggregation"
        DEX_BNB_BASE["dex_bnb_base_trades<br/>Contains all BSC DEXs:<br/>- PancakeSwap V2/V3<br/>- ApeSwap, Biswap<br/>- SushiSwap, MDEX<br/>30+ DEXs"]
    end

    subgraph "Cross-chain Aggregation"
        DEX_BASE["dex_base_trades<br/>All blockchain DEX trades<br/>UNION ALL merge<br/>47 blockchains"]
    end

    subgraph "Token Information Enrichment"
        TOKENS["tokens.erc20<br/>BSC token metadata<br/>- DefinedFi automated data<br/>- tokens_bnb_bep20 supplement"]
        
        PRICES["prices.usd_with_native<br/>BSC token price data<br/>minute-level granularity"]
        
        TRUSTED["prices.trusted_tokens<br/>trusted token validation"]
    end

    subgraph "Final Processing"
        ENRICH_MACRO["enrich_dex_trades macro<br/>- Add token symbols and decimals<br/>- Calculate actual token amounts<br/>- Add USD values<br/>- Generate token pair names"]
        
        DEX_TRADES["dex.trades final table<br/>WHERE blockchain = 'bnb'<br/>AND project = 'pancakeswap'<br/>AND version = '2'"]
    end

    %% Data flow
    SWAP_EVENTS --> PSV2_BNB
    PSV2_BNB --> UNISWAP_MACRO
    PSV2_BNB --> MMPOOL_LOGIC
    PSV2_BNB --> STABLESWAP_LOGIC
    
    UNISWAP_MACRO --> PSV2_BASE
    MMPOOL_LOGIC --> PSV2_BASE
    STABLESWAP_LOGIC --> PSV2_BASE
    
    PSV2_BASE --> DEX_BNB_BASE
    DEX_BNB_BASE --> DEX_BASE
    DEX_BASE --> ENRICH_MACRO
    
    TOKENS --> ENRICH_MACRO
    PRICES --> ENRICH_MACRO
    TRUSTED --> ENRICH_MACRO
    
    ENRICH_MACRO --> DEX_TRADES

    %% Styling
    classDef blockchain fill:#e1f5fe
    classDef source fill:#f3e5f5
    classDef macro fill:#fff3e0
    classDef aggregation fill:#e8f5e8
    classDef enrichment fill:#fce4ec
    classDef final fill:#ffebee
    
    class SWAP_EVENTS blockchain
    class PSV2_BNB source
    class UNISWAP_MACRO,MMPOOL_LOGIC,STABLESWAP_LOGIC,ENRICH_MACRO macro
    class PSV2_BASE,DEX_BNB_BASE,DEX_BASE aggregation
    class TOKENS,PRICES,TRUSTED enrichment
    class DEX_TRADES final
```

## Detailed Lineage Analysis

### 1. Blockchain Event Layer
**Raw Data Source**: Smart contract events on BSC blockchain
- `PancakePair_evt_Swap`: Standard V2 AMM swap events
- `PancakeFactory_evt_PairCreated`: Trading pair creation events  
- `PancakeSwapMMPool_evt_Swap`: Market maker pool swap events
- `PancakeStableSwap_evt_TokenExchange`: Stablecoin swap events

### 2. Data Source Definition Layer
**File**: `sources/pancakeswap/bnb/pancakeswap_v2_bnb_sources.yml`
```yaml
sources:
  - name: pancakeswap_v2_bnb
    tables:
      - name: PancakePair_evt_Swap
      - name: PancakeFactory_evt_PairCreated
      - name: PancakeSwapMMPool_evt_Swap
      - name: PancakeStableSwap_evt_TokenExchange
```

### 3. Macro Processing Layer
**Location**: `dbt_subprojects/dex/models/trades/bnb/platforms/pancakeswap_v2_bnb_base_trades.sql`

#### Three Processing Logic Types:
1. **Standard V2 Trades**: `uniswap_compatible_v2_trades` macro
   - Process `PancakePair_evt_Swap` events
   - Get token pair info through `PancakeFactory_evt_PairCreated`
   - Parse `amount0In/Out`, `amount1In/Out` to determine buy/sell direction

2. **MMPool Trades**: Direct SQL logic
   - Process `PancakeSwapMMPool_evt_Swap` events
   - User vs Market Maker trading model
   - ETH address conversion to WBNB

3. **StableSwap Trades**: Direct SQL logic
   - Process stablecoin swap events
   - Get token info through factory contract association

### 4. Platform-level Aggregation
**Output**: `pancakeswap_v2_bnb_base_trades`
- Merge three trade types via `UNION ALL`
- Standardize field formats: `token_bought_amount_raw`, `token_sold_address`, etc.
- Version identification: `version = '2'/'mmpool'/'stableswap'`

### 5. Chain-level Aggregation  
**File**: `dbt_subprojects/dex/models/trades/bnb/dex_bnb_base_trades.sql`
- Contains all 30+ DEX protocols on BSC
- PancakeSwap V2 is one `ref('pancakeswap_v2_bnb_base_trades')`

### 6. Cross-chain Aggregation
**File**: `dbt_subprojects/dex/models/trades/dex_base_trades.sql`  
- Merge trading data from all 47 blockchains
- BSC data comes from `ref('dex_bnb_base_trades')`

### 7. Data Enrichment
**Macro**: `enrich_dex_trades()`
**Process**:
```sql
-- 1. Add token metadata
LEFT JOIN tokens.erc20 ta ON ta.blockchain='bnb' AND ta.contract_address=token_bought_address
LEFT JOIN tokens.erc20 tb ON tb.blockchain='bnb' AND tb.contract_address=token_sold_address

-- 2. Calculate actual amounts  
token_bought_amount = token_bought_amount_raw / pow(10, ta.decimals)
token_sold_amount = token_sold_amount_raw / pow(10, tb.decimals)

-- 3. Add USD values
LEFT JOIN prices.usd_with_native p ON p.blockchain='bnb' AND p.contract_address=token_bought_address
```

### 8. Token Data Sources
**BSC Token Metadata** (`tokens.erc20` WHERE `blockchain='bnb'`):
- **Primary Source**: `dune.definedfi.dataset_tokens` (automated)
- **Supplementary Source**: `tokens_bnb_bep20.sql` (manually maintained 62 tokens)
- **Merge Logic**: Automated data takes priority, supplementary data fills gaps

### 9. Final Output
**Table**: `dex.trades`
**PancakeSwap V2 Data Filter**:
```sql
WHERE blockchain = 'bnb' 
  AND project = 'pancakeswap' 
  AND version IN ('2', 'mmpool', 'stableswap')
```

## Key Technical Features

### Incremental Processing
- All layers support incremental updates
- Use `incremental_predicate('block_time')` to process only new data

### Data Quality
- Unique key constraint: `['tx_hash', 'evt_index']`
- Token address standardization (ETH → WBNB conversion)
- Price data validation through `trusted_tokens`

### Performance Optimization  
- Partitioned by `block_month`
- Delta file format
- Optimized merge strategy

---

This complete lineage demonstrates how PancakeSwap V2 trading data flows from BSC blockchain events through multiple layers of processing and enrichment to form standardized trading records, with token information primarily relying on DefinedFi's automated data source.
