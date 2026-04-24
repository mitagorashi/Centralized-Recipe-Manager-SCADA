# Modbus Register Map

## Overview

The PLC exposes `Control_DB` (DB2) to Ignition via the Siemens `MB_SERVER` instruction running in OB1. The server listens on **port 502** using the standard Modbus TCP protocol. Ignition connects as a Modbus TCP client using the built-in Modbus TCP driver.

The register map covers 9 holding registers — all mapped to absolute byte offsets in Control_DB. Non-optimized block access is enabled on DB2, which guarantees stable byte offsets that match this map exactly.

---

## MB_SERVER Configuration

```pascal
// Modbus_Connection DB uses TCON_IP_v4 datatype
// Port: 502
// Interface ID: hardware identifier of CPU PROFINET port
// IP: 0.0.0.0 (listen on all interfaces)

MB_SERVER(
    DISCONNECT    := FALSE,              // Keep connection open permanently
    MB_HOLD_REG   := P#DB2.DBX0.0 WORD 20,  // Expose first 20 words of DB2
    NDR           => MB_NDR,            // New data received flag
    DR            => MB_DR,             // Data read flag
    ERROR         => MB_ERROR,          // Error flag
    STATUS        => MB_STATUS,         // Status code
    CONNECT       := "Modbus_Connection"
);
```

---

## Holding Register Map

Modbus holding registers are 1-indexed. HR1 maps to the first word (bytes 0–1) of DB2.

| Register | HR Address | DB2 Byte Offset | Variable Name | TIA Type | Access | Description |
|---|---|---|---|---|---|---|
| HR1 | 40001 | 0 | `Active_Recipe_Index` | Int | **R/W** | Active recipe slot (0–9). Write from Ignition to switch recipe. |
| HR2 | 40002 | 2 | `Conveyor_Speed_Pct` | Int | **R/W** | Belt speed setpoint 0–100%. Write from Ignition slider. |
| HR3 | 40003 | 4 | `Drive_Enable` | Bool (bit 0) | **R/W** | Belt enable/disable. Write TRUE to enable belt movement. |
| HR4 | 40004 | 6 | `Actual_Speed_Pct` | Int | R | Actual belt speed readback 0–100%. Read only. |
| HR5 | 40005 | 8 | `Product_Count` | Int | R | Products counted this batch. Increments at label sensor. |
| HR6 | 40006 | 10 | `Line_Running` | Bool (bit 0) | **R/W** | Master line start/stop. Write TRUE to start, FALSE to stop. |
| HR7 | 40007 | 12 | `Active_Cap_Time` | Int | R | Current recipe cap hold time in ms. Read only. |
| HR8 | 40008 | 14 | `Active_Label_Time` | Int | R | Current recipe label hold time in ms. Read only. |
| HR9 | 40009 | 16 | `Active_Batch_Target` | Int | R | Current recipe batch target count. Read only. |

---

## Ignition Tag Configuration

Tags created in Ignition Designer using the Modbus TCP device named `Recipe_Modbus`:

| Ignition Tag Name | OPC Item Path | Data Type | Access |
|---|---|---|---|
| `Active_Recipe_Index` | `[Recipe_Modbus]HR1` | Int2 | Read/Write |
| `Conveyor_Speed_Pct` | `[Recipe_Modbus]HR2` | Int2 | Read/Write |
| `Drive_Enable` | `[Recipe_Modbus]HR3` | Int2 | Read/Write |
| `Actual_Speed_Pct` | `[Recipe_Modbus]HR4` | Int2 | Read |
| `Product_Count` | `[Recipe_Modbus]HR5` | Int2 | Read |
| `Line_Running` | `[Recipe_Modbus]HR6` | Int2 | Read/Write |
| `Active_Cap_Time` | `[Recipe_Modbus]HR7` | Int2 | Read |
| `Active_Label_Time` | `[Recipe_Modbus]HR8` | Int2 | Read |
| `Active_Batch_Target` | `[Recipe_Modbus]HR9` | Int2 | Read |

---

## Ignition Device Settings

```
Device Name:  Recipe_Modbus
Device Type:  Modbus TCP
Hostname:     192.168.0.1
Port:         502
Unit ID:      1
```

---

## Write Operation Flow

When Ignition writes to a holding register, this is the exact sequence:

```
1. Operator changes dropdown to Oil 1L (value = 2)
2. Ignition onActionPerformed script executes:
       system.tag.writeBlocking(['[default]Recipe_PLC/Active_Recipe_Index'], [2])
3. Ignition Modbus driver sends Function Code 06 (Write Single Register) to HR1
4. MB_SERVER receives write → updates Control_DB.Active_Recipe_Index = 2
5. MB_SERVER.NDR output goes TRUE for one scan
6. OB1 detects index change → executes MOVE blocks:
       Control_DB.Conveyor_Speed_Pct    := Recipe_DB.recipes[2].Conveyor_Speed_Pct    (= 25)
       Control_DB.Active_Cap_Time       := Recipe_DB.recipes[2].Cap_Hold_Time          (= 700)
       Control_DB.Active_Label_Time     := Recipe_DB.recipes[2].Label_Hold_Time        (= 400)
       Control_DB.Active_Batch_Target   := Recipe_DB.recipes[2].Batch_Count_Target     (= 5)
7. Recipe_Control FB reads Conveyor_Speed_Pct = 25 on next scan
8. Belt speed output changes: 25 * 3.0 / 100 = 0.75 m/s → QD100
9. Factory I/O belt slows to 0.75 m/s immediately
10. Next boxes processed with Cap_Hold_Time = 700ms and Label_Hold_Time = 400ms
```

---

## Read Operation Flow

Ignition reads holding registers on a configurable scan rate (default 1000ms):

```
1. Ignition Modbus driver sends Function Code 03 (Read Holding Registers) for HR1–HR9
2. MB_SERVER responds with current values of Control_DB bytes 0–17
3. Ignition updates tag values in the Tag Browser
4. Perspective dashboard components bound to these tags update automatically
5. Product_Count display updates every 1 second showing live count
```

---

## Control_DB Byte Layout Reference

| Bytes | Variable | Type | Size |
|---|---|---|---|
| 0–1 | Active_Recipe_Index | Int | 2 bytes |
| 2–3 | Conveyor_Speed_Pct | Int | 2 bytes |
| 4.0 | Drive_Enable | Bool | 1 bit |
| 4.1 | Drive_Fault_Reset | Bool | 1 bit |
| 5 | (padding) | — | 1 byte |
| 6–7 | Actual_Speed_Pct | Int | 2 bytes |
| 8–9 | Product_Count | Int | 2 bytes |
| 10.0 | Line_Running | Bool | 1 bit |
| 10.1 | Fault_Active | Bool | 1 bit |
| 11 | (padding) | — | 1 byte |
| 12–13 | Active_Cap_Time | Int | 2 bytes |
| 14–15 | Active_Label_Time | Int | 2 bytes |
| 16–17 | Active_Batch_Target | Int | 2 bytes |

---

## Why Modbus TCP Instead of OPC-UA

TIA Portal V18 has a confirmed bug (Siemens support thread #317080) that causes the application to crash when compiling OPC-UA server interfaces on certain Windows configurations. Modbus TCP via the built-in `MB_SERVER` instruction was chosen as the communication layer because:

- It is natively supported in S7-1500 without additional licensing
- It provides reliable bidirectional read/write access
- It is widely deployed in real industrial environments alongside Siemens PLCs
- The `MB_SERVER` block is stable and well-documented
- Ignition's Modbus TCP driver is mature and production-proven

The tradeoff is that string data (Product_Name) cannot be transmitted over Modbus registers. This was solved by using an expression in Ignition's Perspective dashboard — a `case` statement maps the integer recipe index to the product name string on the SCADA side, eliminating the need to transmit strings over Modbus entirely.
