---
name: Factor Migration
description: Migrate quantitative factors from Python to C++ (blitzer) with complete implementation and testing
---

# Factor Migration Skill

Migrate quantitative factor calculations from Python to C++ for the blitzer project.

## Overview

This skill guides the migration of factor computation logic from Python (pandas/numpy based) to C++ (streaming/incremental processing). The blitzer framework uses a specific architecture with state management, parallel execution, and Parquet output.

## Interactive Workflow

### Step 0: Gather User Input

**Ask user for the following information:**

1. **Factor Name(s)**: Which factor(s) to migrate
   - Can be a single factor or comma-separated list
   - Example: `ValueAtRiskLt95`, `RESIDUAL_ALPHA`, `compute_volatility`

Use `AskUserQuestion` tool if factor names are not provided in the arguments.

### Step 1: Locate and Read Python Factor

Search for the factor implementation in these repositories:

```bash
# Primary locations
/home/lxb/github/factor-store
/home/lxb/github/intraday-alpha-research
/home/lxb/github/research-toolkit
```

Read the factor implementation and analyze:
- Input data fields (Close, Volume, etc.)
- Window size and parameters
- Computation logic
- Output field names

### Step 2: Resolve Dependencies

When encountering functions/modules with unclear definitions, trace them to their source:

| Library | Path | Key Functions |
|---------|------|---------------|
| `numpandas` | `/home/lxb/github/numpandas` | `ffillna`, `clean`, `shift`, `rolling_query_quantile` |
| `research-toolkit` | `/home/lxb/github/research-toolkit/toolkit/stats` | `regress_multi_dimensional_by_columns`, `standardize` |
| `bottleneck` | System package | `move_sum`, `move_std`, `move_median`, `nanargmax` |
| `factorai` | `/home/lxb/github/factor-store/factorai` | `align_backward`, `ComputationContext` |

**Important Functions to Understand:**

1. **`rolling_query_quantile`** - NaN-aware rolling quantile calculation
   - Collects valid (non-NaN) values in window
   - Sorts and indexes at quantile position
   - Returns NaN if insufficient valid data

2. **`regress_multi_dimensional_by_columns`** - Cross-sectional regression
   - Takes Y (T×N) and list of X factors (T×N each)
   - Returns betas, errors, T-values, F-values
   - Used for ResidualAlpha type factors

### Step 3: Check Blitzer Patterns

Refer to existing implementations for patterns:

| Factor Type | Reference File | Key Pattern |
|-------------|----------------|-------------|
| Rolling Sum | `feature_volatility.cc` | `MoveSumState` + running sum |
| Rolling Quantile | `feature_high_std_return_mean.cc` | `std::sort` + index at quantile |
| Cross-sectional | `feature_residual_alpha.cc` | Market return regression |

## Prerequisites

- Access to the blitzer codebase at `/home/lxb/github/blitzer`
- Python factor implementation with clear computation logic
- Understanding of the factor's input data and output schema

## Files to Modify

| File | Operation | Purpose |
|------|-----------|---------|
| `blitzer/features/feature_impls/feature_xxx.cc` | **Create** | Factor computation implementation |
| `blitzer/features/feature_impls/feature_common.h` | Modify | Add function declaration |
| `blitzer/features/slice_minbar_features.h` | Modify | Add FeatureResult fields + Parquet schema |
| `blitzer/features/slice_minbar_features.cc` | Modify | Register factor in BuildComputeFuncs |
| `blitzer/features/CMakeLists.txt` | Modify | Add source file |
| `config/slice_minbar_features_config.json` | Modify | Enable factor in runtime config |
| `blitzer/features/slice_minbar_features_test.cc` | Modify | Add unit tests |

## Implementation Patterns

### State Structure Template

```cpp
struct FeatureXxxState {
  static constexpr int kRollingWindow = 240;  // Match Python window
  static constexpr double kNearZeroThreshold = 1e-8;  // For filtering

  // Rolling window data
  std::deque<double> returns;

  // Forward fill state
  double last_valid_price = std::numeric_limits<double>::quiet_NaN();
};
```

### Validity Check Pattern

**IMPORTANT**: Use `valid_data_start_time` based on previous trading day:

```cpp
// Correct pattern - based on previous trading day 9:31
static const absl::CivilSecond valid_data_start_time = []() {
  absl::CivilDay today = prv::Take<ClockService>().Now<absl::CivilDay>();
  absl::CivilDay prev_trade_day = utils::GetTradingDateBefore(today, 1);
  return absl::CivilSecond(prev_trade_day.year(),
                           prev_trade_day.month(),
                           prev_trade_day.day(),
                           9, 31, 0);  // 9:31 not 9:30
}();

// In the computation loop
if (minbar.date_time < valid_data_start_time) {
  return;  // Skip data before valid start
}
```

### Quantile Calculation Pattern (from VaR)

```cpp
// Collect valid returns
std::vector<double> valid_returns;
valid_returns.reserve(kRollingWindow);
for (const double& ret : st.returns) {
  if (std::isfinite(ret)) {
    valid_returns.push_back(ret);
  }
}

if (valid_returns.empty()) {
  out.value_at_risk = NaN;
  return;
}

// Calculate quantile
std::sort(valid_returns.begin(), valid_returns.end());
size_t k = static_cast<size_t>(static_cast<double>(valid_returns.size()) * kQuantile);
if (k >= valid_returns.size()) {
  k = valid_returns.size() - 1;
}
double quantile_value = valid_returns[k];

// Near-zero filtering
if (std::abs(quantile_value) < kNearZeroThreshold) {
  out.value_at_risk = NaN;
} else {
  out.value_at_risk = quantile_value;
}
```

### Return Calculation Pattern

```cpp
static double SafeReturn(double current, double previous) {
  if (!std::isfinite(previous) || previous <= 1e-9) {
    return std::numeric_limits<double>::quiet_NaN();
  }
  double ret = (current - previous) / previous;
  // Clean extreme values (|ret| > 10.0 is unrealistic)
  if (!std::isfinite(ret) || std::abs(ret) > 10.0) {
    return std::numeric_limits<double>::quiet_NaN();
  }
  return ret;
}
```

### Forward Fill Pattern

```cpp
// Get current price with adjustment
double current_price = m.close * adj_factor;
if (!std::isfinite(current_price) || current_price <= 0) {
  current_price = st.last_valid_price;  // Forward fill
}

// Update for next iteration
if (std::isfinite(current_price) && current_price > 0) {
  st.last_valid_price = current_price;
}
```

### Rolling Window Maintenance

```cpp
// Add new value
st.returns.push_back(current_return);

// Maintain window size
while (st.returns.size() > kRollingWindow) {
  st.returns.pop_front();
}

// Check minimum data requirement
if (st.returns.size() < kRollingWindow) {
  out.factor_value = NaN;
  return;
}
```

### Batch Processing Template

```cpp
void ComputeFeatureXxx(const std::vector<SliceMinbar>& minbars,
                       std::vector<FeatureResult>& results,
                       const std::vector<double>& adj_factors,
                       const std::vector<SharedFeatureState>& /* shared_states */,
                       marl::Scheduler& scheduler) {
  static std::vector<FeatureXxxState> states;
  InitializeAndValidateStates(states, minbars, "FeatureXxx");

  // Valid data start time
  static const absl::CivilSecond valid_data_start_time = /* ... */;

  marl::WaitGroup wg(minbars.size());
  for (size_t i = 0; i < minbars.size(); ++i) {
    scheduler.enqueue(marl::Task{[&, i, wg]() {
      defer(wg.done());

      const auto& minbar = minbars[i];
      if (minbar.date_time < valid_data_start_time) {
        return;
      }

      auto& result = results[i];
      auto& state = states[i];
      double adj_factor = adj_factors[i];

      ComputeFeatureXxxSingle(minbar, state, result, adj_factor);
    }});
  }
  wg.wait();
}
```

## Validation Workflow

### Step 1: Build and Run Tests

```bash
cd /home/lxb/github/blitzer
nix develop
cd build
ninja slice_minbar_features
ninja slice_minbar_features_test
./test/slice_minbar_features_test --gtest_filter="*Xxx*"
```

### Step 2: Compare with Python Output

用户提供对比脚本位于 `/home/lxb/github/factor-store/notebook/` 目录下。

请用户提供或创建对比 notebook，包含以下逻辑：

```python
# 1. 加载 online (C++) 数据
from toolkit.data_loader import load_data, DateSpec, IsStock
online_data = load_data(
    path="/var/lib/wonder/warehouse/database/lxb/T0FactorData/factors/...",
    date=DateSpec(start="2026-01-30", end="2026-01-30"),
    filter=IsStock()
)

# 2. 加载 offline (Python) 数据 - 使用 factor-store 计算
# ... 运行 Python 因子计算逻辑 ...

# 3. 对比差异
diff = online_data["factor_name"] - offline_data["factor_name"]
print(f"Max diff: {diff.abs().max()}")
print(f"Mean diff: {diff.abs().mean()}")

# 4. 检查特定股票
problem_stocks = diff[diff.abs() > 1e-6].index
print(f"Problem stocks: {problem_stocks}")
```

### Common Issues to Check

1. **kMinPeriods 差异**: 前几根 bar 可能因最小周期设置不同而有差异
2. **NaN 处理差异**: 确保 NaN 传播逻辑一致
3. **Forward Fill 差异**: 确保价格填充逻辑完全匹配
4. **边界条件**: 第一分钟 (9:31) 的数据可能需要特殊处理

## Update Header Files

**In `feature_common.h`**, add function declaration:

```cpp
void ComputeFeatureXxx(const std::vector<SliceMinbar>& minbars,
                       std::vector<FeatureResult>& results,
                       const std::vector<double>& adj_factors,
                       const std::vector<SharedFeatureState>& shared_states,
                       marl::Scheduler& scheduler);
```

**In `slice_minbar_features.h`**, add to `FeatureResult`:

```cpp
struct FeatureResult {
  // ... existing fields ...

  // New factor fields
  double xxx_field = std::numeric_limits<double>::quiet_NaN();
};
```

**In Parquet codec section**, add mapping:

```cpp
codec.Add("XxxField", &blitzer::features::FeatureResult::xxx_field);
```

## Register Factor

**In `slice_minbar_features.cc`**, find `BuildComputeFuncs` and add:

```cpp
if (enabled_set.contains("xxx_field")) {
  funcs.push_back({"xxx_field", ComputeFeatureXxx});
}
```

## Update CMakeLists.txt

Add to `slice_minbar_features` target:

```cmake
target_sources(
  slice_minbar_features
  ...
  PRIVATE ...
          feature_impls/feature_xxx.cc
  ...)
```

## Update Runtime Config

**In `config/slice_minbar_features_config.json`**, add new factor fields:

```json
{
  "enable_features": [
    "prices_err",
    "upward_std_bias",
    "xxx_field"
  ]
}
```

## Add Unit Tests

**In `slice_minbar_features_test.cc`**:

1. Add to `all_features` in SetUp:
```cpp
std::unordered_set<std::string> all_features = {
  // ... existing ...
  "xxx_field"
};
```

2. Add test cases:
```cpp
TEST_F(SliceMinbarFeaturesTest, XxxInsufficientDataTest) {
  // Test with < window size data points → should return NaN
}

TEST_F(SliceMinbarFeaturesTest, XxxFullWindowTest) {
  // Test with full window → should return valid values
}

TEST_F(SliceMinbarFeaturesTest, XxxEdgeCasesTest) {
  // Test NaN, zero volume, extreme values
}
```

## Verification Checklist

- [ ] State structure with correct window size
- [ ] Validity check using `valid_data_start_time` (9:31 AM previous trading day)
- [ ] Price adjustment and forward fill
- [ ] Return calculation with extreme value cleaning
- [ ] Sliding window maintenance
- [ ] Near-zero value filtering (if applicable)
- [ ] Function declaration in header (`feature_common.h`)
- [ ] FeatureResult fields added (`slice_minbar_features.h`)
- [ ] Parquet codec mapping (`slice_minbar_features.h`)
- [ ] Factor registered in BuildComputeFuncs (`slice_minbar_features.cc`)
- [ ] CMakeLists.txt updated
- [ ] Runtime config updated (`config/slice_minbar_features_config.json`)
- [ ] Unit tests added (`slice_minbar_features_test.cc`)
- [ ] Compilation successful
- [ ] Tests pass
- [ ] Output matches Python implementation (using notebook comparison)

## Common Pitfalls

1. **Wrong validity check**: Don't use `shared_states[i].is_valid`, use `valid_data_start_time`
2. **Start time off-by-one**: Python lookback=240 aligns with 9:31, not 9:30
3. **Missing near-zero filter**: Some factors filter `|value| < 1e-8` to NaN
4. **NaN propagation**: Ensure `std::isfinite()` checks are consistent
5. **Window not full**: Return NaN until window has enough data
