# Maple6 Test Fixture

Hardware plan and BOM for a dedicated Maple6 production/bench test fixture.

**Document status:** hardware plan v0.1  
**DUT test point:** **8S full-charge equivalent = 33.6 V**


> **Prototype policy:** no fuse is fitted in the current fixture revision. The bench PSU must have current limiting enabled. The PRECHECK branch remains mandatory.

> **External equipment already owned:** one 4-UART USB module and the programmers. They are not included in the fixture BOM.


## 1. Goals

The fixture shall:

- power Maple6 from a controlled 33.6 V bench supply;
- run a low-current PRECHECK before applying full DUT power;
- verify internal 10 V and 5 V rails;
- measure input voltage/current/power;
- measure switched-output voltage/current/power;
- apply an internal ~5 A resistive load;
- support an external/non-resistive load while still measuring it;
- emulate receiver-side and flight-controller-side UART traffic from the PC;
- actuate the Maple6 button electrically;
- electrically check the LED/status control line; the operator visually confirms that the LED lights;
- keep the programmer external.

## 2. Fixed operating conditions

| Item | Value |
|---|---:|
| DUT input voltage | **33.6 V** |
| Battery equivalent | **8S full charge** |
| Target internal load current | **~5.04 A** |
| UART test baud | **420000 baud** |
| Service rail | **12.0 V** |
| Fixture controller | **Arduino Nano ESP32** |
| CRSF traffic generator/analyzer | **PC + existing 4-UART USB module** |

The Nano ESP32 is a fixture supervisor only. It does **not** sit in the CRSF data path.

## 3. Block diagram

```text
                                  PC
                     ┌────────────┼────────────┐
                     │            │            │
                Nano ESP32    UART RX      UART FC
                supervisor        │            │
                     │         ADuM1201    Maple6 onboard
                     │            │         FC-side ADuM
                     │            │            │
                     └────────── Maple6 ───────┘
                                  │
                        INA228 / ADS1115
                                  │
                              load bank
```

## 4. DUT power path

Only BAT+ is switched. DUT GND remains permanently common with fixture GND.

```text
33.6 V PSU +
      │
      ├────────────── XL7015 ───────► 12 V service rail
      │
      ├── 330 Ω / 10 W ─ PRECHECK ─┐
      │                             ├── INA228 #1 ───► Maple6 BAT+
      └──── RT424012 POWER ─────────┘

PSU GND ─────────────────────────────────────────────► Maple6 GND
```

### POWER relay

Use **RT424012, 12 V coil**.

Use both NO poles **in series on BAT+**. Do not switch DUT GND.

## 5. PRECHECK

### Hardware

- PRECHECK relay: **Omron G6K-2P-Y DC12** or equivalent small 12 V signal relay with at least 60 VDC / 0.1 A contact capability;
- current-limiting resistor: **330 Ω / 10 W**.

Hard-short values:

```text
Ishort ≈ 33.6 / 330 = 0.102 A
Presistor ≈ 3.42 W
```

### Sequence

```text
POWER OFF
PRECHECK ON
read DUT voltage/current
   ├─ abnormal / short → PRECHECK OFF → FAIL
   └─ normal           → POWER ON
                         wait 100...200 ms
                         PRECHECK OFF
```

The numeric PASS/FAIL threshold is a commissioning parameter derived from known-good boards.

## 6. Internal load bank

Use **3 × 20 Ω / 100 W aluminium chassis resistors in parallel**.

```text
Req = 20 / 3 = 6.667 Ω
I = 33.6 / 6.667 ≈ 5.04 A
Ptotal ≈ 169.3 W
Presistor ≈ 56.4 W
```

All resistors mount to a user-selected aluminium thermal plate with thermal compound.

Recommended initial production load pulse: **3–5 s**.

```text
Maple6 PW_OUT
      │
   INA228 #2
      │
 RT424012 LOAD
      │
3 × 20 Ω / 100 W
      │
     GND
```

The external-load connector should branch **after INA228 #2** so external loads are also measured.

## 7. Measurements

### INA228

Use **2 × Matek I2C-INA-BM**:

- `INA228 #1`: DUT input;
- `INA228 #2`: switched output / load.

Assign different I²C addresses.

### ADS1115

Use **1 × ADS1115 I²C module**, powered from 3.3 V.

Recommended divider network:

| Channel | Rail | Rtop | Rbottom | Nominal ADC voltage |
|---|---:|---:|---:|---:|
| AIN0 | 10 V | **33 kΩ** | **10 kΩ** | ~2.326 V |
| AIN1 | 5 V | **12 kΩ** | **10 kΩ** | ~2.273 V |
| AIN2 | spare | — | — | — |
| AIN3 | spare | — | — | — |

Use **1% resistors**.

Add **100 nF from each populated ADC input node to GND**.

The divider values intentionally leave margin below the 3.3 V ADC supply instead of scaling nominal rails to exactly 3.3 V.

## 8. UART

### Receiver side

Use **1 × external ADuM1201 module**:

```text
existing USB-UART RX channel ↔ ADuM1201 ↔ Maple6 receiver UART
```

Power the Maple-side isolator domain from the appropriate Maple6 5 V rail.

### Flight-controller side

Maple6 already contains an onboard ADuM on the FC UART. **Do not add another isolator.**

The fixture supplies the external side of the onboard isolator as a real FC would.

UART requirements:

- two independent full-duplex UARTs;
- 420000 baud;
- simultaneous bidirectional operation;
- no RTS/CTS;
- FC UART also used at 115200 for ROM bootloader/programming workflow where needed.

## 9. Button and LED/status

Use **1 × HW-279 2-channel 12 V relay module**, configured for HIGH-trigger operation.

| HW-279 channel | Function |
|---|---|
| CH1 | Maple6 BUTTON |
| CH2 | spare |

BUTTON contact:

```text
COM → BUTTON test point
NO  → GND
```

Add a **10 kΩ pulldown** on the HW-279 input so ESP32 reset/boot means relay OFF.

LED/status test scope:

- fixture checks the electrical control/data line;
- operator confirms visible illumination;
- no camera or photodiode required.

## 10. Relay driver

Use **1 × CJMCU-2803 module based on ULN2803A**.

Suggested allocation:

| Channel | Function |
|---|---|
| CH1 | POWER RT424012 |
| CH2 | LOAD RT424012 |
| CH3 | PRECHECK G6K |
| CH4–CH8 | spare |

Use **10 kΩ pulldowns** on POWER, LOAD and PRECHECK driver inputs so reset/boot defaults to OFF.

The HW-279 module has its own relay drivers and is driven directly by Nano ESP32 GPIO.

## 11. Service power

Use **1 × XL7015 buck module**, adjusted to **12.0 V**.

Input is the fixed 33.6 V bench supply.

The 12 V rail powers:

- POWER relay coil;
- LOAD relay coil;
- PRECHECK relay coil;
- HW-279 relay module.

The Nano ESP32 and low-voltage logic are powered from PC USB.

## 12. Display

Selected model:

**Waveshare 1.3inch OLED (B), product code 10451**

- 1.3";
- 128×64;
- SH1106;
- I²C/SPI capable;
- use **I²C**;
- 3.3 V logic/power arrangement for this fixture.

Displayed information can include fixture state, PRECHECK, measured rails, current, load state and PASS/FAIL.

## 13. Programmer

The programmer stays external.

Prototype assumption:

- programmer signal pins are high-impedance while idle;
- programmer does not materially back-power Maple6 with DUT POWER OFF.

## 14. Proposed automated sequence

```text
1. Fixture ready; POWER=OFF, LOAD=OFF.
2. Operator inserts Maple6.
3. PRECHECK=ON.
4. Validate DUT voltage/current.
5. POWER=ON.
6. Wait 100...200 ms.
7. PRECHECK=OFF.
8. Verify 10 V and 5 V rails.
9. Verify idle input current.
10. Exercise BUTTON.
11. Run RX ↔ FC UART/CRSF tests from the PC.
12. Check LED/status line; operator confirms visible LED.
13. LOAD=ON for ~3–5 s.
14. Measure VOUT, IOUT, power and switched-path voltage drop.
15. LOAD=OFF.
16. POWER=OFF.
17. Report PASS/FAIL.
```

# BOM

## Modules / relays

| Qty | Part | Purpose |
|---:|---|---|
| 1 | Arduino Nano ESP32 | Fixture supervisor |
| 1 | XL7015 buck module | 33.6 V → 12 V service rail |
| 1 | CJMCU-2803 / ULN2803A module | POWER / LOAD / PRECHECK coil driver |
| 2 | RT424012, 12 V coil | POWER and LOAD |
| 1 | Omron G6K-2P-Y DC12 | PRECHECK |
| 1 | HW-279 2-channel 12 V relay module | BUTTON + spare |
| 2 | Matek I2C-INA-BM | Input and output V/I/P |
| 1 | ADS1115 I²C module | 10 V and 5 V rail measurement |
| 1 | ADuM1201 module | Receiver UART isolation |
| 1 | Waveshare 10451 1.3" SH1106 OLED | Display |

## Power resistors

| Qty | Part | Purpose |
|---:|---|---|
| 1 | 330 Ω / 10 W wirewound/cement | PRECHECK limiter |
| 3 | 20 Ω / 100 W aluminium chassis resistor | Main ~5 A load bank |

## ADC / control passives

| Qty | Value | Purpose |
|---:|---|---|
| 1 | 33 kΩ, 1% | 10 V divider top |
| 1 | 12 kΩ, 1% | 5 V divider top |
| 2 | 10 kΩ, 1% | ADS1115 divider bottoms |
| 2 | 100 nF | ADS1115 input filters |
| 4 minimum | 10 kΩ | POWER, LOAD, PRECHECK, BUTTON control pulldowns |

## Existing / mechanical / user-selected

Not included in the purchase BOM:

- existing 4-UART USB module;
- existing programmer;
- bench PSU with current limiting;
- aluminium load-bank plate/heatsink;
- pogo pins and fixture mechanics;
- wiring, terminal blocks and enclosure hardware;
- external-load connector.

## Hardware status

The electronic architecture is considered defined for the prototype. Remaining values such as PRECHECK thresholds, rail tolerance windows, idle-current limits and PSU current-limit settings are commissioning/software parameters, not BOM blockers.
