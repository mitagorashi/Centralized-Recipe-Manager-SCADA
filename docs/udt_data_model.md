# Recipe_Type UDT — Data Model Reference

## Overview

`Recipe_Type` is a User-Defined Type (UDT) created in TIA Portal V18 under **PLC data types**. It defines the structure of a single product recipe. The `Recipe_DB` data block holds an `Array[0..9] of Recipe_Type` — giving the system 10 recipe slots, each fully independent.

This architecture means switching recipes requires a single integer write from Ignition (the array index). The PLC handles the rest — reading the correct slot and applying all parameters in one scan cycle.

---

## UDT Definition

```pascal
TYPE Recipe_Type :
STRUCT
    Product_Name       : String;    // Product label — displayed in Ignition dropdown
    Conveyor_Speed_Pct : Int;       // Belt speed as percentage 0–100
    Cap_Hold_Time      : Int;       // Cap station hold duration in milliseconds
    Label_Hold_Time    : Int;       // Label station hold duration in milliseconds
    Batch_Count_Target : Int;       // Number of products to produce per batch
    Recipe_Active      : Bool;      // Activation flag — set by PLC logic
END_STRUCT
END_TYPE
```

---

## Member Reference

| Member | TIA Portal Type | Range | Default | Maps To | Written By |
|---|---|---|---|---|---|
| `Product_Name` | String | — | `'Product_A'` | Display only in Ignition | Ignition form |
| `Conveyor_Speed_Pct` | Int | 0–100 | `50` | Belt speed → QD100 (REAL 0.0–3.0 m/s) | Ignition form |
| `Cap_Hold_Time` | Int | 0–9999 | `500` | Cap pusher state machine timer (ms) | Ignition form |
| `Label_Hold_Time` | Int | 0–9999 | `300` | Label pusher state machine timer (ms) | Ignition form |
| `Batch_Count_Target` | Int | 1–9999 | `10` | Batch completion check in Control_DB | Ignition form |
| `Recipe_Active` | Bool | TRUE/FALSE | `FALSE` | Internal activation flag | PLC logic |

---

## Recipe_DB Structure

```
Recipe_DB (DB1)
│
├── recipes[0]  →  Recipe_Type
│   ├── Product_Name       = 'Water_500ml'
│   ├── Conveyor_Speed_Pct = 60
│   ├── Cap_Hold_Time      = 500
│   ├── Label_Hold_Time    = 300
│   ├── Batch_Count_Target = 10
│   └── Recipe_Active      = FALSE
│
├── recipes[1]  →  Recipe_Type
│   ├── Product_Name       = 'Juice_330ml'
│   ├── Conveyor_Speed_Pct = 45
│   ├── Cap_Hold_Time      = 400
│   ├── Label_Hold_Time    = 250
│   ├── Batch_Count_Target = 15
│   └── Recipe_Active      = FALSE
│
├── recipes[2]  →  Recipe_Type
│   ├── Product_Name       = 'Oil_1L'
│   ├── Conveyor_Speed_Pct = 25
│   ├── Cap_Hold_Time      = 700
│   ├── Label_Hold_Time    = 400
│   ├── Batch_Count_Target = 5
│   └── Recipe_Active      = FALSE
│
├── recipes[3]  →  Recipe_Type  (empty — available)
├── recipes[4]  →  Recipe_Type  (empty — available)
├── recipes[5]  →  Recipe_Type  (empty — available)
├── recipes[6]  →  Recipe_Type  (empty — available)
├── recipes[7]  →  Recipe_Type  (empty — available)
├── recipes[8]  →  Recipe_Type  (empty — available)
└── recipes[9]  →  Recipe_Type  (empty — available)
```

---

## Pre-loaded Test Recipes

| Index | Product | Speed | Belt (m/s) | Cap Hold | Label Hold | Batch |
|---|---|---|---|---|---|---|
| 0 | Water 500ml | 60% | 1.8 m/s | 500ms | 300ms | 10 |
| 1 | Juice 330ml | 45% | 1.35 m/s | 400ms | 250ms | 15 |
| 2 | Oil 1L | 25% | 0.75 m/s | 700ms | 400ms | 5 |
| 3–9 | Available | — | — | — | — | — |

---

## Speed Scaling Formula

The `Conveyor_Speed_Pct` integer (0–100) is converted to a REAL value for the Factory I/O analog input:

```pascal
// In Recipe_Control FB — belt speed output
"Conveyor_Speed" := REAL#3.0 *
    INT_TO_REAL(Control_DB.Conveyor_Speed_Pct) / REAL#100.0;
```

| Speed % | REAL Value | Factory I/O Speed |
|---|---|---|
| 0% | 0.0 | Stopped |
| 25% | 0.75 | Slow (Oil 1L) |
| 45% | 1.35 | Medium (Juice 330ml) |
| 60% | 1.80 | Normal (Water 500ml) |
| 100% | 3.00 | Maximum |

---

## Control_DB Buffer Variables

Because Modbus TCP cannot address nested UDT arrays with variable offsets, active recipe parameters are copied into flat buffer variables in Control_DB when the recipe index changes:

```pascal
// In OB1 — triggered when Active_Recipe_Index changes
Control_DB.Active_Cap_Time :=
    Recipe_DB.recipes[Control_DB.Active_Recipe_Index].Cap_Hold_Time;

Control_DB.Active_Label_Time :=
    Recipe_DB.recipes[Control_DB.Active_Recipe_Index].Label_Hold_Time;

Control_DB.Active_Batch_Target :=
    Recipe_DB.recipes[Control_DB.Active_Recipe_Index].Batch_Count_Target;

Control_DB.Conveyor_Speed_Pct :=
    Recipe_DB.recipes[Control_DB.Active_Recipe_Index].Conveyor_Speed_Pct;
```

These buffer variables are what Ignition reads via Modbus holding registers — simple flat integers with known absolute offsets.

---

## Design Decisions

**Why UDT array instead of 10 separate DBs?**
A single array with one index variable means recipe switching is atomic — one write changes everything. Separate DBs would require multiple writes and introduce timing issues between parameter updates.

**Why non-optimized block access?**
Optimized blocks use internal Siemens symbolic addressing. Non-optimized blocks expose absolute byte offsets, which is required for Modbus TCP addressing and any future snap7 or OPC-UA access.

**Why not expose Recipe_DB directly over Modbus?**
Modbus cannot address nested structures. The buffer variable approach flattens the active recipe into addressable holding registers, keeping Ignition tag configuration simple and reliable.
