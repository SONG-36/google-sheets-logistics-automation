# Requirements Analysis (Client Specification + Real CSV Structure)

## 1. Objective

Build a fully automated Google Sheets workbook that:

- Accepts a Shopify CSV export pasted into `Import!A1`
- Automatically updates all downstream sheets
- Uses formulas only (no scripts, no manual steps)
- Is robust to changing column order
- Uses rule-driven classification (Mapping sheet)
- Produces logistics and invoice-ready outputs

## 2. Client Requirements Breakdown

Below is a structured interpretation of the client’s request.

### R1. Single Data Entry Point

> CSV will be pasted only into Import!A1 and replaced each time.
>  No other manual steps allowed.

#### Interpretation

- Import sheet must act as raw input layer
- No helper columns allowed inside Import
- No manual sorting / filtering / refreshing
- Entire system must recalculate automatically

#### Implementation Direction

- All downstream sheets use header-based extraction
- ARRAYFORMULA-style dynamic column expansion
- No static range references (like A2:A1000)

### R2. Column Order May Change

> Column order in CSV may change.

#### Risk

Shopify exports may:

- Reorder columns
- Add new columns
- Remove optional fields

#### Requirement

Formulas must locate columns by header name.

#### Implementation Strategy

- Use MATCH("Header Name", Import!1:1, 0)
- Use INDEX with dynamic column index
- Never use fixed column letters

### R3. No Pivot Tables

> No pivot tables allowed.

#### Interpretation

Invoice summary must be computed via:

- UNIQUE
- COUNTIFS
- SUMIFS
- or QUERY (non-pivot)

### R4. Mapping-Driven Classification

> All accessory detection must use Mapping sheet
>  No hardcoding inside formulas

#### Interpretation

- Accessory detection cannot rely on:
  - Price = 0
  - FREE GIFT text
  - Hardcoded keywords in formulas

### Therefore

Mapping sheet becomes:

Single Source of Truth for classification rules.

## 3. Real CSV Structure Analysis

The provided CSV is a Shopify Order Export with line-item granularity.

Key structural properties:

### 3.1 Order-Level vs Line-Item-Level

Each order appears multiple times:

Example:

```
#2404  Greenhouse
#2404  Telescopic Roof Supports
#2404  Wooden Gardening Bed
#2404  Automatic Air Vent
```

#### Conclusion

System must operate at line-item level,
 not at order header level.

### 3.2 Core Fields Identified

From the CSV:

#### Order Identifier

- `Name` → Order Nr

#### Product Information

- `Lineitem name`
- `Lineitem sku`
- `Lineitem quantity`
- `Lineitem price`

These form the main calculation base.

#### Address Fields (Shipping)

- Shipping Name
- Shipping Street / Address1
- Shipping Address2
- Shipping City
- Shipping Zip
- Shipping Province
- Shipping Country
- Shipping Phone

Address must be combined from these fields.

#### Currency

- Currency column confirms USD
- All pricing based on Lineitem price

### 3.3 Fields Required by Client But Missing in CSV

Client requires:

- Pallet Size (mm)
- Pallet Weight (kg)
- Packages

These are NOT present in CSV.

#### Decision

- These columns must exist in General sheet
- They will initially be empty
- Validation rules must highlight missing values

This is intentional and demonstrates system robustness.

## 4. Business Logic Extraction from CSV

### 4.1 Greenhouse Detection

Greenhouse products follow naming pattern:

Examples:

- World's Most Popular Polycarbonate Greenhouse - 19.66'(6m)
- Classic Arch Greenhouse - 32.75(10m)
- Greenhouse - 13'(4m)

Observation:

- Length always appears in parentheses
- Format varies slightly
- Always includes “m”

Conclusion:

Length must be extracted via regex.

### 4.2 Base / Extension Logic

Client Rule:

- Base = 4m
- Extension = 2m
- 4m → 1 base
- 6m → 1 base + 1 extension
- 8m → 1 base + 2 extensions

General formula:

If length = L:

- Bases = Qty
- Extensions = ((L - 4) / 2) × Qty

### 4.3 Accessory Detection

Accessory items include:

- FREE GIFT items
- Vent products
- Doors
- Gardening Beds

But detection must NOT rely on:

- Price
- "FREE GIFT" text
- Product category guess

Instead:

Accessory must be determined via Mapping sheet.

## 5. Derived Sheet Requirements

### General Sheet (Core Data Model)

Must include:

- Order Nr
- Address
- Ordered Greenhouse
- SKU
- Qty
- Pallet Size
- Pallet Weight
- Packages
- Price USD
- Exchange Rate
- Price EUR
- Total EUR

General acts as transformation layer.

### Packing List

Derived from General:

- Only shipping-relevant rows
- Includes pallet data
- Includes automatic totals

### Invoice

Derived from General:

- Unique Greenhouse types
- Count of Bases
- Count of Extensions
- Amount of Accessories
- Total row

## 6. Validation Requirements

Two categories of validation:

1. Missing pallet information
2. Items not matched by Mapping rules

Both must be visually highlighted.

## 7. Final Acceptance Criteria

After replacing CSV in Import!A1:

- General updates automatically
- Packing List updates automatically
- Invoice updates automatically
- Totals update automatically
- Validation updates automatically
- No manual adjustments required

## 8. Architectural Direction

This workbook will be designed as:

Input Layer → Transformation Layer → Output Layers

Rather than raw/interim/out file-based pipeline.