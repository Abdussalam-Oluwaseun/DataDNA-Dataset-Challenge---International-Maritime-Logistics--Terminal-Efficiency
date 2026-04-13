# Maritime Logistics Analysis - Semantic Model Data Dictionary

**Model Name:** Maritime Logistics Analysis  
**Last Updated:** April 13, 2026  
**Description:** Power BI semantic model for analyzing international maritime logistics terminal efficiency, vessel performance, and cargo movements.

---

## Table of Contents
1. [Overview](#overview)
2. [Table Summaries](#table-summaries)
3. [Dimension Tables](#dimension-tables)
4. [Fact Tables](#fact-tables)
5. [Measures Table](#measures-table)
6. [Parameter Table](#parameter-table)
7. [All Measures Reference](#all-measures-reference)

---

## Overview

The Maritime Logistics Analysis semantic model contains **6 tables** with **18 calculated measures**, organized as:
- **3 Dimension tables** (dim_terminal, dim_time, dim_vessel)
- **1 Fact table** (fact_cargo_movements)
- **1 Measures table** (_measures)
- **1 Parameter table** (Parameter)

### Relationships
```
fact_cargo_movements --[terminal_id]--> dim_terminal
fact_cargo_movements --[vessel_key]--> dim_vessel
fact_cargo_movements --[date]--> dim_time
```

---

## Table Summaries

| Table Name | Type | Columns | Purpose |
|------------|------|---------|---------|
| dim_terminal | Dimension | 8 | Terminal locations, capacities, and operational efficiency metrics |
| dim_time | Dimension | 8 | Time hierarchy with fiscal periods, quarters, months, weeks, and shifts |
| dim_vessel | Dimension | 9 | Vessel characteristics, categories, status, and capacity benchmarks |
| fact_cargo_movements | Fact | 11 | Core facts: individual cargo movement records with duration, capacity, and performance flags |
| _measures | Measures | 1 + 18 measures | Repository of 18 calculated measures for reporting |
| Parameter | Utility | 3 | Field parameters for dynamic metric selection in reports |

---

## Dimension Tables

### **1. dim_terminal**
**Type:** Dimension  
**Description:** Contains information about maritime terminals, their geographic locations, and operational efficiency metrics.

| Column Name | Data Type | Column Type | Description | Sample Values / Range |
|------------|-----------|-------------|-------------|----------------------|
| RowNumber-* | Int64 | Calculated (Hidden) | Auto-generated unique identifier | Generated row numbers |
| **terminal_id** | Int64 | Source | Unique identifier for each terminal | 1, 2, 3, 4, 5... |
| **terminal_name** | String | Source | Name of the maritime terminal | e.g., "Port of Singapore", "Port of Dubai" |
| **regional_hub** | String | Source | Geographic region or hub classification | e.g., "Southeast Asia", "Middle East", "Europe" |
| **longitude** | Double | Source | Geographic longitude coordinate | -180.0 to 180.0 |
| **latitude** | Double | Source | Geographic latitude coordinate | -90.0 to 90.0 |
| **Terminal Capacity** | Double | Calculated | Composite efficiency metric combining density, volume, and efficiency ratio | Formula: Density × Volume × Efficiency; Range: typically 0-1000+ |
| **operational efficiency** | Double | Calculated | Ratio of global average duration to terminal's average duration; >1 indicates faster than global average | Format: Percentage (0%-200%+); Values range 0.50 to 2.50 |

**Calculated Column Details:**

**Terminal Capacity:**
```
VAR _movements = RELATEDTABLE(fact_cargo_movements)
VAR Density = DIVIDE(COUNTROWS(_movements), 
              COUNTROWS(DISTINCT(SELECTCOLUMNS(_movements, "date", fact_cargo_movements[date]))))
VAR Volume = DIVIDE(SUMX(_movements, fact_cargo_movements[container_count]), COUNTROWS(_movements))
VAR TerminalAvgDuration = AVERAGEX(_movements, fact_cargo_movements[move_duration (days)])
VAR GlobalAvgDuration = CALCULATE(AVERAGEX(fact_cargo_movements, fact_cargo_movements[move_duration (days)]), ALL(dim_terminal))
VAR Efficiency = DIVIDE(GlobalAvgDuration, TerminalAvgDuration)
RETURN Density * Volume * Efficiency
```

**operational efficiency:**
```
VAR _movements = RELATEDTABLE(fact_cargo_movements)
VAR Efficiency = DIVIDE(
    CALCULATE(AVERAGEX(fact_cargo_movements, fact_cargo_movements[move_duration (days)]), ALL(dim_terminal)),
    AVERAGEX(_movements, fact_cargo_movements[move_duration (days)]))
RETURN Efficiency
```

---

### **2. dim_time**
**Type:** Dimension  
**Description:** Time dimension providing hierarchical breakdown of dates into fiscal years, quarters, months, weeks, and operational shifts.

| Column Name | Data Type | Column Type | Description | Sample Values / Range |
|------------|-----------|-------------|-------------|----------------------|
| RowNumber-* | Int64 | Calculated (Hidden) | Auto-generated unique identifier | Generated row numbers |
| **date** | DateTime | Source | Date of the cargo movement | 2024-01-01 to 2026-12-31 (or current range) |
| **fiscal_year** | Int64 | Source | Fiscal year classification | 2024, 2025, 2026 |
| **quarter** | Int64 | Source | Quarter of the fiscal year | 1, 2, 3, 4 |
| **month** | Int64 | Source | Month number | 1-12 |
| **week** | Int64 | Source | Week number in the year | 1-52/53 |
| **shift** | String | Source | Operational shift designation | e.g., "Day", "Night", "Evening" |
| **month name** | String | Calculated | Month name abbreviation | "Jan", "Feb", "Mar"... "Dec" |
| **month year** | String | Calculated | Combined month and year label | "Jan-2024", "Feb-2024"... "Dec-2026" |

**Calculated Column Details:**

**month name:**
```
FORMAT(dim_time[date], "MMM")
```

**month year:**
```
FORMAT(dim_time[date], "MMM-YYYY")
```

---

### **3. dim_vessel**
**Type:** Dimension  
**Description:** Contains vessel master data including characteristics, categorization, age, status, and performance benchmarks.

| Column Name | Data Type | Column Type | Description | Sample Values / Range |
|------------|-----------|-------------|-------------|----------------------|
| RowNumber-* | Int64 | Calculated (Hidden) | Auto-generated unique identifier | Generated row numbers |
| **vessel_key** | String | Source | Unique identifier for vessel (surrogate key) | e.g., "V001", "V002", "V_ABC123" |
| **vessel_id** | Int64 | Source | Operational vessel ID | 1001, 1002, 1003... |
| **vessel_name** | String | Source | Commercial name of the vessel | e.g., "MV Pacific", "Ever Given", "MSC Oscar" |
| **vessel_category** | String | Source | Type/class of vessel | "Container Ship", "Bulk Carrier", "Tanker", "General Cargo" |
| **build_year** | Int64 | Source | Year the vessel was constructed | 1990-2024 |
| **Avg Move Duration (Category Benchmark)** | Double | Calculated | Average move duration for all vessels in same category (benchmark) | Typically 2-10 days |
| **vessel age** | Int64 | Calculated | Age of vessel in years (current year - build year) | 2-34 years |
| **Vessel Status** | String | Calculated | Operational status based on recent activity | "Active", "Inactive", "Dormant" |
| **vessel capacity** | Double | Calculated | Average container capacity for this vessel category | Typically 1000-20000 containers |

**Calculated Column Details:**

**Avg Move Duration (Category Benchmark):**
```
CALCULATE(
    AVERAGE(fact_cargo_movements[move_duration (days)]),
    FILTER(ALL(fact_cargo_movements), 
           RELATED(dim_vessel[vessel_category]) = dim_vessel[vessel_category]))
```

**vessel age:**
```
[crt_year] - dim_vessel[build_year]
```

**Vessel Status:**
```
VAR LastGlobalDate = CALCULATE(MAX(fact_cargo_movements[date]), ALL(fact_cargo_movements))
VAR LastVesselDate = CALCULATE(MAX(fact_cargo_movements[date]), RELATEDTABLE(fact_cargo_movements))
VAR DaysSinceActive = LastGlobalDate - LastVesselDate
RETURN SWITCH(TRUE(),
    DaysSinceActive <= 180,  "Active",
    DaysSinceActive <= 270,  "Inactive",
    "Dormant")
```

**vessel capacity:**
```
CALCULATE(
    AVERAGE(fact_cargo_movements[container_count]),
    FILTER(ALL(fact_cargo_movements), 
           RELATED(dim_vessel[vessel_category]) = dim_vessel[vessel_category]))
```

---

## Fact Tables

### **4. fact_cargo_movements**
**Type:** Fact Table  
**Description:** Core fact table containing individual cargo movement records with operational metrics, performance flags, and efficiency indicators. Each row represents one cargo movement event.

| Column Name | Data Type | Column Type | Description | Sample Values / Range | Notes |
|------------|-----------|-------------|-------------|----------------------|-------|
| RowNumber-* | Int64 | Calculated (Hidden) | Auto-generated unique identifier | Generated row numbers | |
| **date** | DateTime | Source | Date of the cargo movement | 2024-01-01 to current | FK to dim_time |
| **vessel_key** | String | Source | Reference to vessel dimension | "V001", "V002", etc. | FK to dim_vessel |
| **terminal_id** | Int64 | Source | Reference to terminal dimension | 1, 2, 3... | FK to dim_terminal |
| **container_count** | Int64 | Source | Number of containers moved | 100-20,000 | Summarize: Sum |
| **move_duration (hrs)** | Double | Source | Duration of movement in hours | 2-500 hours | Raw source value |
| **move_duration (days)** | Double | Calculated | Duration of movement in days | 0.08-20.83 days | Formula: hrs ÷ 24 |
| **movement_id** | Int64 | Source | Unique identifier for this movement record | 1, 2, 3... | Primary identifier |
| **Duration Flag** | String | Calculated | Performance classification based on variance from category benchmark | "Invalid", "Fast", "On Time", "Delayed", "Late", "Critical" | Based on benchmark comparison |
| **capacity flag** | String | Calculated | Capacity utilization status relative to vessel capacity | "Empty", "Light", "Optimal", "Strained", "Over", "Critical" | Based on container count vs capacity |
| **Performance Flag** | String | Calculated | Combined performance indicator from duration and capacity flags | "Peak", "Efficient", "Repositioning", "Lagging", "At Risk", "Critical", "Failed", "Overloaded", "Unverified" | 2D matrix: Duration × Capacity |
| **Performance Flag SVG** | String | Calculated (Hidden) | Visual badge as SVG image URL | data:image/svg+xml... | Image URL for visualization |

**Key Calculated Columns:**

**move_duration (days):**
```
DIVIDE(fact_cargo_movements[move_duration (hrs)], 24, 0)
```

**Duration Flag:**
```
VAR ActualDuration = fact_cargo_movements[move_duration (days)]
VAR Benchmark = RELATED(dim_vessel[Avg Move Duration (Category Benchmark)])
VAR VariancePct = DIVIDE(ActualDuration - Benchmark, Benchmark)
RETURN SWITCH(TRUE(),
    VariancePct <= -0.90,   "Invalid",
    VariancePct <= -0.50,   "Fast",
    VariancePct <= 0.0,     "On Time",
    VariancePct <= 0.20,    "Delayed",
    VariancePct <= 0.50,    "Late",
    "Critical")
```

**capacity flag:**
```
VAR container_count = fact_cargo_movements[container_count]
VAR vessel_capacity = RELATED(dim_vessel[vessel capacity])
VAR VariancePct = DIVIDE(container_count - vessel_capacity, vessel_capacity)
RETURN SWITCH(TRUE(),
    VariancePct <= -1.00,   "Empty",
    VariancePct <= -0.50,   "Light",
    VariancePct <= 0.00,    "Optimal",
    VariancePct <= 0.20,    "Strained",
    VariancePct <= 0.50,    "Over",
    "Critical")
```

**Performance Flag:**
```
VAR CapacityFlag = fact_cargo_movements[capacity flag]
VAR DurationFlag = fact_cargo_movements[Duration Flag]
RETURN SWITCH(TRUE(),
    -- Data quality rows (Invalid Duration - any capacity)
    DurationFlag = "Invalid",    "Unverified",
    
    -- Empty vessel rows
    CapacityFlag = "Empty" && DurationFlag = "Fast",     "Repositioning",
    CapacityFlag = "Empty" && DurationFlag = "On Time",  "Repositioning",
    CapacityFlag = "Empty" && DurationFlag = "Delayed",  "Lagging",
    CapacityFlag = "Empty" && DurationFlag = "Late",     "Lagging",
    CapacityFlag = "Empty" && DurationFlag = "Critical", "Critical",
    
    -- Light capacity rows
    CapacityFlag = "Light" && DurationFlag = "Fast",     "Peak",
    CapacityFlag = "Light" && DurationFlag = "On Time",  "Efficient",
    CapacityFlag = "Light" && DurationFlag = "Delayed",  "Lagging",
    CapacityFlag = "Light" && DurationFlag = "Late",     "At Risk",
    CapacityFlag = "Light" && DurationFlag = "Critical", "Critical",
    
    -- Optimal capacity
    CapacityFlag = "Optimal" && DurationFlag = "Fast",       "Peak",
    CapacityFlag = "Optimal" && DurationFlag = "On Time",    "Peak",
    CapacityFlag = "Optimal" && DurationFlag = "Delayed",    "Efficient",
    CapacityFlag = "Optimal" && DurationFlag = "Late",       "Lagging",
    CapacityFlag = "Optimal" && DurationFlag = "Critical",   "Critical",
    
    -- Strained capacity
    CapacityFlag = "Strained" && DurationFlag = "Fast",      "Efficient",
    CapacityFlag = "Strained" && DurationFlag = "On Time",   "Efficient",
    CapacityFlag = "Strained" && DurationFlag = "Delayed",   "Lagging",
    CapacityFlag = "Strained" && DurationFlag = "Late",      "At Risk",
    CapacityFlag = "Strained" && DurationFlag = "Critical",  "Critical",
    
    -- Overloaded capacity
    CapacityFlag = "Over" && DurationFlag = "Fast",     "At Risk",
    CapacityFlag = "Over" && DurationFlag = "On Time",  "Overloaded",
    CapacityFlag = "Over" && DurationFlag = "Delayed",  "At Risk",
    CapacityFlag = "Over" && DurationFlag = "Late",     "Critical",
    CapacityFlag = "Over" && DurationFlag = "Critical", "Failed",
    
    -- Critical overload
    CapacityFlag = "Critical" && DurationFlag = "Fast",       "At Risk",
    CapacityFlag = "Critical" && DurationFlag = "On Time",    "Critical",
    CapacityFlag = "Critical" && DurationFlag = "Delayed",    "Critical",
    CapacityFlag = "Critical" && DurationFlag = "Late",       "Failed",
    CapacityFlag = "Critical" && DurationFlag = "Critical",   "Failed",
    
    "Unverified")
```

---

## Measures Table

### **5. _measures**
**Type:** Measures Repository  
**Description:** Dedicated measures table containing 18 calculated measures for reporting and analysis. Follows Power BI best practice of centralizing all measures in a dedicated table.

| Column Name | Data Type | Column Type | Description |
|------------|-----------|-------------|-------------|
| RowNumber-* | Int64 | Calculated (Hidden) | Auto-generated unique identifier |
| ** | String | Source (Hidden) | Placeholder column (hidden) |

---

## All Measures Reference

### Core Movement Metrics

#### **1. Total Movements**
- **Table:** _measures
- **Data Type:** Int64
- **Format:** #,0
- **Description:** Count of all cargo movements in the selected context
- **DAX Formula:**
```
COUNTROWS(fact_cargo_movements)
```
- **Usage:** Total volume of movements, port activity analysis

---

#### **2. Total Containers**
- **Table:** _measures
- **Data Type:** Int64
- **Format:** #,0
- **Description:** Aggregate count of all containers moved across all movements
- **DAX Formula:**
```
SUM(fact_cargo_movements[container_count])
```
- **Usage:** Total cargo volume, throughput analysis

---

#### **3. Avg Move Duration (days)**
- **Table:** _measures
- **Data Type:** Double
- **Format:** 0.0
- **Description:** Average duration of a single cargo movement in days
- **DAX Formula:**
```
AVERAGE(fact_cargo_movements[move_duration (days)])
```
- **Usage:** Performance benchmarking, efficiency comparison

---

#### **4. Total Move_duration**
- **Table:** _measures
- **Data Type:** Double
- **Format:** (Default)
- **Description:** Sum of all movement durations in days (cumulative)
- **DAX Formula:**
```
SUM(fact_cargo_movements[move_duration (days)])
```
- **Usage:** Operational capacity analysis

---

#### **5. Throughput**
- **Table:** _measures
- **Data Type:** Double
- **Format:** 0
- **Description:** Ratio of total containers moved to total movement duration; containers per day efficiency metric
- **DAX Formula:**
```
DIVIDE(
    SUMX(fact_cargo_movements, fact_cargo_movements[container_count]),
    SUMX(fact_cargo_movements, fact_cargo_movements[move_duration (days)]))
```
- **Usage:** Operational efficiency, capacity utilization

---

#### **6. Movements per Terminal**
- **Table:** _measures
- **Data Type:** Int64
- **Format:** 0
- **Description:** Total movements for a specific terminal in the filter context
- **DAX Formula:**
```
CALCULATE(
    [Total Movements],
    ALLEXCEPT(dim_terminal, dim_terminal[terminal_id]))
```
- **Usage:** Terminal-level activity analysis

---

### Vessel Metrics

#### **7. Total Vessel**
- **Table:** _measures
- **Data Type:** Int64
- **Format:** #,0
- **Description:** Count of unique vessels
- **DAX Formula:**
```
COUNTA(dim_vessel[vessel_key])
```
- **Usage:** Fleet size tracking

---

#### **8. active vessels**
- **Table:** _measures
- **Data Type:** Int64
- **Format:** #,0
- **Description:** Count of vessels with "Active" status in current filter context
- **DAX Formula:**
```
CALCULATE(
    COUNTROWS(dim_vessel),
    dim_vessel[Vessel Status] = "Active")
```
- **Usage:** Fleet availability, operational capacity

---

#### **9. Inactive vessels**
- **Table:** _measures
- **Data Type:** Int64
- **Format:** (Default)
- **Description:** Count of vessels that are not actively operating (Inactive or Dormant)
- **DAX Formula:**
```
CALCULATE(
    [Total Vessel],
    dim_vessel[Vessel Status] <> "Active")
```
- **Usage:** Idle fleet identification

---

### Performance Metrics

#### **10. Efficiency Score**
- **Table:** _measures
- **Data Type:** Double
- **Format:** 0.00%;-0.00%;0.00%
- **Description:** Percentage of movements with Peak, Efficient, or Repositioning performance flags; indicator of optimal operations
- **DAX Formula:**
```
VAR EfficientMovements =
    CALCULATE(
        COUNTROWS(fact_cargo_movements),
        fact_cargo_movements[Performance Flag] IN {"Peak", "Efficient", "Repositioning"})
VAR TotalMovements =
    CALCULATE(
        COUNTROWS(fact_cargo_movements),
        fact_cargo_movements[Performance Flag] <> "Unverified")
RETURN DIVIDE(EfficientMovements, TotalMovements)
```
- **Usage:** Overall performance KPI, quality metrics

---

#### **11. Count of Bottlenecks**
- **Table:** _measures
- **Data Type:** Int64
- **Format:** #,0
- **Description:** Count of movements with suboptimal performance (not Peak, Efficient, Repositioning, or Unverified)
- **DAX Formula:**
```
CALCULATE(
    [Total Movements],
    NOT fact_cargo_movements[Performance Flag] IN {"Peak", "Efficient", "Repositioning", "Unverified"})
```
- **Usage:** Problem identification, operational issues

---

### Time & Calendar Metrics

#### **12. crt_year**
- **Table:** _measures
- **Data Type:** Int64
- **Format:** 0
- **Description:** Current/latest fiscal year in the dataset
- **DAX Formula:**
```
MAX(dim_time[fiscal_year])
```
- **Usage:** Year-to-date calculations, current period metrics

---

### Month-over-Month (MoM) Change Metrics

#### **13. Total Containers MoM%**
- **Table:** _measures
- **Data Type:** Double
- **Format:** 0.00%;-0.00%;0.00%
- **Description:** Month-over-month percentage change in total containers moved
- **DAX Formula:**
```
VAR __PREV_MONTH = CALCULATE([Total Containers], DATEADD('dim_time'[date], -1, MONTH))
RETURN DIVIDE([Total Containers] - __PREV_MONTH, __PREV_MONTH)
```
- **Usage:** Trend analysis, growth metrics

---

#### **14. Total Movements MoM%**
- **Table:** _measures
- **Data Type:** Double
- **Format:** 0.00%;-0.00%;0.00%
- **Description:** Month-over-month percentage change in movement count
- **DAX Formula:**
```
VAR __PREV_MONTH = CALCULATE([Total Movements], DATEADD('dim_time'[date], -1, MONTH))
RETURN DIVIDE([Total Movements] - __PREV_MONTH, __PREV_MONTH)
```
- **Usage:** Activity trend analysis

---

#### **15. Avg Move Duration (days) MoM%**
- **Table:** _measures
- **Data Type:** Double
- **Format:** 0.00%;-0.00%;0.00%
- **Description:** Month-over-month percentage change in average movement duration
- **DAX Formula:**
```
VAR __PREV_MONTH = CALCULATE([Avg Move Duration (days)], DATEADD('dim_time'[date], -1, MONTH))
RETURN DIVIDE([Avg Move Duration (days)] - __PREV_MONTH, __PREV_MONTH)
```
- **Usage:** Performance improvement tracking

---

### Field Parameter Measures

#### **16. Selected Metric Value**
- **Table:** _measures
- **Data Type:** Variant
- **Format:** (Default)
- **Description:** Dynamic measure that returns the selected metric from the Parameter field parameter; enables flexible metric selection
- **DAX Formula:**
```
SELECTEDMEASURE()
```
- **Usage:** Dynamic dashboards, multi-metric selection

---

#### **17. Highlight MAX**
- **Table:** _measures
- **Data Type:** Variant
- **Format:** (Default)
- **Description:** Returns the selected metric value only when it is the maximum value in the trend; used for conditional highlighting
- **DAX Formula:**
```
VAR _CurrentValue = [Selected Metric Value]
VAR _AllValuesOnTrend = 
    CALCULATETABLE(
        ADDCOLUMNS(
            ALLSELECTED('dim_time'[date]),
            "@Val", [Selected Metric Value]),
        ALLSELECTED('dim_time'))
VAR _MaxVal = MAXX(_AllValuesOnTrend, [@Val])
RETURN IF(_CurrentValue = _MaxVal, _CurrentValue, BLANK())
```
- **Usage:** Visual highlighting of peak performance

---

#### **18. Highlight MIN**
- **Table:** _measures
- **Data Type:** Variant
- **Format:** (Default)
- **Description:** Returns the selected metric value only when it is the minimum value in the trend; used for conditional highlighting
- **DAX Formula:**
```
VAR _CurrentValue = [Selected Metric Value]
VAR _AllValuesOnTrend = 
    CALCULATETABLE(
        ADDCOLUMNS(
            ALLSELECTED('dim_time'[date]),
            "@Val", [Selected Metric Value]),
        ALLSELECTED('dim_time'))
VAR _MinVal = MINX(_AllValuesOnTrend, [@Val])
RETURN IF(_CurrentValue = _MinVal, _CurrentValue, BLANK())
```
- **Usage:** Visual highlighting of lowest performance

---

## Parameter Table

### **6. Parameter**
**Type:** Utility Table  
**Description:** Field parameter table supporting dynamic metric selection in reports via the "Parameter Field"

| Column Name | Data Type | Column Type | Description | Sample Values |
|------------|-----------|-------------|-------------|----------------|
| RowNumber-* | Int64 | Calculated (Hidden) | Auto-generated unique identifier | Generated row numbers |
| **Parameter** | String | Source | Display value for parameter selection | Measure names from _measures table |
| **Parameter Fields** | String | Source (Hidden) | Technical identifier linked to fields | Field references (hidden from users) |
| **Parameter Order** | Int64 | Source (Hidden) | Sort order for parameter display | 1, 2, 3, 4... |

---

## Data Quality & Validation Notes

### Performance Flags Distribution
The **Performance Flag** column creates a 2-dimensional classification matrix:

**Capacity Dimension:**
| Flag | Meaning | Range |
|------|---------|-------|
| Empty | ≤100% below capacity | -100% variance |
| Light | 50-100% below capacity | -50% to -100% |
| Optimal | 0-100% of capacity | 0% to ±0% |
| Strained | 0-20% over capacity | 0% to +20% |
| Over | 20-50% over capacity | +20% to +50% |
| Critical | >50% over capacity | >+50% |

**Duration Dimension:**
| Flag | Meaning | Variance |
|------|---------|----------|
| Invalid | >90% faster than benchmark | ≤-90% |
| Fast | 50-90% faster than benchmark | -50% to -90% |
| On Time | At or faster than benchmark | 0% to ±0% |
| Delayed | 0-20% slower than benchmark | 0% to +20% |
| Late | 20-50% slower than benchmark | +20% to +50% |
| Critical | >50% slower than benchmark | >+50% |

### Vessel Status Determination
Status is determined by days since last activity:
- **Active:** ≤180 days since last movement
- **Inactive:** 181-270 days since last movement
- **Dormant:** >270 days since last movement

---

## Tips for Data Analysis

1. **Terminal Efficiency:** Use `operational efficiency` and `Terminal Capacity` columns from dim_terminal
2. **Vessel Performance:** Filter by `Vessel Status` and compare `Avg Move Duration (Category Benchmark)`
3. **Movement Quality:** Focus on `Performance Flag` and `Efficiency Score` measure
4. **Trend Analysis:** Use `*_MoM%` measures with dim_time dimensions
5. **Bottleneck Identification:** Use `Count of Bottlenecks` measure for problem areas
6. **Fleet Utilization:** Monitor `active vessels` vs `Inactive vessels` in dim_vessel

---

## Relationships & Filter Flow

```
dim_time ──────┐
               │
               ├─► fact_cargo_movements ◄──┬─ dim_terminal
               │                           │
               └──────────────────────────┴─ dim_vessel
```

- All filters flow **one-way** from dimensions to fact table
- Cross-filtering behavior: **OneDirection**
- Referential integrity: Not enforced (allowing some flexibility in data)
- Join behavior: DateAndTime for temporal filtering

---

*End of Data Dictionary*
