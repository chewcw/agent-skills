# Example — Device Import with Catalog Validation

Demonstrates Pattern B end-to-end.

---

## Conversation

**User:**
> Import hardware from `hardware_list.xlsx`. Validate against catalog first.

---

**Agent step 1 — List sheets**

```
files_list_sheets(filePath="hardware_list.xlsx")
```

Result: `["Devices"]` — single sheet, used automatically.

---

**Agent step 2 — Read sheet**

```
files_read_excel(filePath="hardware_list.xlsx", sheetName="Devices")
```

Result: 8 rows — headers: `[DeviceName, OrderNumber, Position, Comment]`

---

**Agent step 3 — Validate columns**

```python
validate_columns(
    headers  = ["DeviceName", "OrderNumber", "Position", "Comment"],
    required = ["device_name", "order_number"],
    optional = ["position", "comment"],
)
```

Result: all columns resolved ✓

---

**Agent step 4 — Pre-validate order numbers (× 8)**

```
devices_search_catalog(query="6ES7 516-3AN02-0AB0")  → found: CPU 1516-3 PN/DP  ✓
devices_search_catalog(query="6AV2 124-0MC01-0AX0")  → found: Comfort Panel 15"  ✓
devices_search_catalog(query="6ES7 193-6BP20-0BA0")  → found: ET200SP Interface  ✓
devices_search_catalog(query="6ES7 134-6HD00-0BA1")  → found: Digital Input      ✓
devices_search_catalog(query="6GK5 208-0BA00-2AC2")  → found: Scalance X208      ✓
devices_search_catalog(query="6ES7 155-6AU01-0BN0")  → found: ET200SP IM         ✓
devices_search_catalog(query="6ES7 132-6BH01-0BA0")  → found: DQ 8x24VDC         ✓
devices_search_catalog(query="6ES7 999-0000-0AB0")   → NOT FOUND                 ✗
```

---

**Agent → User (checkpoint):**

```
⚠ 1 order number not found in the hardware catalog:

  Row 8 — DeviceName: "IO_Ext_1", OrderNumber: "6ES7 999-0000-0AB0"

Possible causes:
  - Typo in the order number
  - Hardware catalog not installed for this device family

Continue importing the 7 valid devices?
```

**User:**
> Yes, continue.

---

**Agent step 5 — Verify session**

```
projects_get_session_info()
```

Session active ✓

---

**Agent step 6 — Create devices (loop × 7)**

```
devices_create(deviceName="PLC_1",      orderNumber="6ES7 516-3AN02-0AB0", position="Rack_1")
devices_create(deviceName="HMI_1",      orderNumber="6AV2 124-0MC01-0AX0", position="Panel_1")
devices_create(deviceName="IO_Dev_1",   orderNumber="6ES7 193-6BP20-0BA0", position="Rack_2")
devices_create(deviceName="DI_Mod_1",   orderNumber="6ES7 134-6HD00-0BA1", position="Rack_2_Slot1")
devices_create(deviceName="Switch_1",   orderNumber="6GK5 208-0BA00-2AC2", position="Network")
devices_create(deviceName="IO_Dev_2",   orderNumber="6ES7 155-6AU01-0BN0", position="Rack_3")
devices_create(deviceName="DQ_Mod_1",   orderNumber="6ES7 132-6BH01-0BA0", position="Rack_3_Slot1")
```

All 7 succeed ✓

---

**Agent step 7 — Save**

```
projects_save()
```

---

**Agent response:**

```
✓ Device import completed

Total rows          : 8
Valid (catalog)     : 7   → 7 created
Skipped (no catalog): 1   — Row 8: IO_Ext_1 (6ES7 999-0000-0AB0)

Created devices:
  • PLC_1     — CPU 1516-3 PN/DP     (6ES7 516-3AN02-0AB0)
  • HMI_1     — Comfort Panel 15"    (6AV2 124-0MC01-0AX0)
  • IO_Dev_1  — ET200SP Interface    (6ES7 193-6BP20-0BA0)
  • DI_Mod_1  — Digital Input Module (6ES7 134-6HD00-0BA1)
  • Switch_1  — Scalance X208        (6GK5 208-0BA00-2AC2)
  • IO_Dev_2  — ET200SP IM           (6ES7 155-6AU01-0BN0)
  • DQ_Mod_1  — DQ 8x24VDC          (6ES7 132-6BH01-0BA0)

Next steps:
  - Fix order number for IO_Ext_1 and re-run import for that row
  - Configure Profinet topology
  - Compile hardware: compilation_project()
```
