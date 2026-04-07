# Siemens S7 STL Block Reference

This file contains complete block templates for direct use in Step7 Classic / TIA Portal.

---

## Organization Block (OB)

```
ORGANIZATION_BLOCK "OB1"
TITLE = Main Program Cycle
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1

VAR_TEMP
      RawAnalog    : Int;
      ScaledValue  : Real;
      HI           : Real;
      LOW          : Real;
      OUTPUT       : Real;
      StartPB      : Bool;
      StopPB       : Bool;
      Overload     : Bool;
      Temperature  : Real;
      MotorOutput  : Bool;
      AlarmOutput  : Bool;
END_VAR

BEGIN

NETWORK
TITLE = Read Analog Input
// Read raw analog value from peripheral input word
      L     PIW256;            // Peripheral Input Word
      T     #RawAnalog;

NETWORK
TITLE = Scale Analog Value
// Call FC to scale raw analog value to engineering units
      CALL "Analog Scaling"
      (  IN                          := #RawAnalog ,
         HI_LIM                      := #HI ,
         LO_LIM                      := #LOW ,
         OUT                         := #OUTPUT
      );

NETWORK
TITLE = Call Motor Control Function Block
// Call FB with inputs and map outputs to memory markers
      CALL "Motor Control", "Motor Control_DB"
      (  StartPB                     := #StartPB ,
         StopPB                      := #StopPB ,
         Overload                    := #Overload ,
         Temperature                 := #Temperature ,
         MotorOutput                 := #MotorOutput ,
         AlarmOutput                 := #AlarmOutput
      );

NETWORK
TITLE = Write Motor Output to Peripheral
// Write latched motor output to physical output word
      A     #MotorOutput;
      =     PQ4.0;              // Peripheral output bit

END_ORGANIZATION_BLOCK
```

---

## Function (FC) — Stateless

```
FUNCTION "Analog Scaling" : Void
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1

   VAR_INPUT
      IN      : Int;
      HI_LIM  : Real;
      LO_LIM  : Real;
   END_VAR

   VAR_OUTPUT
      OUT : Real;
   END_VAR

   VAR_TEMP
      TempReal : Real;
   END_VAR

BEGIN

NETWORK
TITLE = Convert INT to REAL
// Convert integer input to real for arithmetic
      L     #IN;
      ITD;
      DTR;
      T     #TempReal;

NETWORK
TITLE = Normalize to 0-27648
// Divide by full-scale analog range
      L     #TempReal;
      L     27648.0;
      /R;
      T     #TempReal;

NETWORK
TITLE = Scale to Engineering Units
// Apply engineering unit scaling: OUT = (HI_LIM - LO_LIM) * normalized + LO_LIM
      L     #HI_LIM;
      L     #LO_LIM;
      -R;
      L     #TempReal;
      *R;
      L     #LO_LIM;
      +R;
      T     #OUT;

END_FUNCTION
```

---

## Function Block (FB) — Stateful

```
FUNCTION_BLOCK "Motor Control"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1

VAR_INPUT
    StartPB      : BOOL ;
    StopPB       : BOOL ;
    Overload     : BOOL ;
    Temperature  : REAL ;
END_VAR

VAR_OUTPUT
    MotorOutput  : BOOL ;
    AlarmOutput  : BOOL ;
END_VAR

VAR
    StartEdge    : BOOL ;
    RunLatch     : BOOL ;
    HighTemp     : BOOL ;
    StartDelay   : TON ;
END_VAR

BEGIN

NETWORK
TITLE = Rising Edge Detection for Start
// Detect rising edge of StartPB to initiate start sequence
      A     #StartPB;
      FP    #StartEdge;
      =     #StartEdge;

NETWORK
TITLE = Start Latch Logic
// Motor starts if start edge detected, no stop signal, no overload
      A     #StartEdge;
      AN    #StopPB;
      AN    #Overload;
      S     #RunLatch;

NETWORK
TITLE = Stop and Overload Reset
// Motor stops on StopPB or overload condition
      A     #StopPB;
      R     #RunLatch;

      A     #Overload;
      R     #RunLatch;

NETWORK
TITLE = Start Delay Timer (3 seconds)
// Introduce 3-second delay before motor output activates
      A     #RunLatch;
      L     S5T#3S;
      T     #StartDelay.PT;

NETWORK
TITLE = Motor Output Logic
// Activate motor output after start delay has elapsed
      A     #StartDelay.Q;
      =     #MotorOutput;

NETWORK
TITLE = High Temperature Detection (>80 deg)
// Set HighTemp flag if temperature exceeds 80 degrees
      L     #Temperature;
      L     80.0;
      >R;
      =     #HighTemp;

NETWORK
TITLE = Alarm Output Logic
// Activate alarm on overload or high temperature
      A     #HighTemp;
      O     #Overload;
      =     #AlarmOutput;

END_FUNCTION_BLOCK
```

---

## FB Instance Data Block

```
DATA_BLOCK "Motor Control_DB"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
NON_RETAIN
"Motor Control"

BEGIN

END_DATA_BLOCK
```

---

## Global Data Block (DB)

```
DATA_BLOCK "Process_Data_DB"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
NON_RETAIN

   VAR
      RunCommand    : Bool;
      FaultCode     : Int;
      SetpointSpeed : Real;
      Motor1        { S7_SetPoint := 'False'} : "UDT_MOTOR";
   END_VAR

BEGIN

END_DATA_BLOCK
```

---

## User-Defined Data Type (UDT)

```
TYPE "UDT_MOTOR"
VERSION : 0.1
   STRUCT
      RUN_FEEDBACK  : Bool;
      FAULT         : Bool;
      SPEED         : Real;
      RUN_HOURS     : DInt;
   END_STRUCT;

END_TYPE
```

---

## FC with S5 Timer (use inside FC, not FB)

```
FUNCTION "Conveyor Sequence" : Void
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1

   VAR_INPUT
      Enable    : Bool;
      ResetCmd  : Bool;
   END_VAR

   VAR_OUTPUT
      ConveyorRun : Bool;
      StepDone    : Bool;
   END_VAR

BEGIN

NETWORK
TITLE = Start Conveyor with On-Delay Timer
// Start 5-second on-delay timer when Enable is active
      A     #Enable;
      L     S5T#5S;
      SE    T5;              // S5 on-delay timer T5

NETWORK
TITLE = Conveyor Run Output After Delay
// Activate conveyor output after timer elapses
      A     T5;
      =     #ConveyorRun;

NETWORK
TITLE = Step Done Pulse
// Assert StepDone for one scan on timer rising edge
      A     T5;
      FP    M10.0;           // Edge bit — must not be reused elsewhere
      =     #StepDone;

NETWORK
TITLE = Reset Timer on Reset Command
// Force timer off when reset command received
      A     #ResetCmd;
      R     T5;

END_FUNCTION
```

---

## FB with Parenthesized Boolean Logic

```
FUNCTION_BLOCK "Valve Control"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1

VAR_INPUT
    ManualOpen    : BOOL ;
    AutoOpen      : BOOL ;
    SystemReady   : BOOL ;
    EmergencyStop : BOOL ;
END_VAR

VAR_OUTPUT
    ValveOutput   : BOOL ;
END_VAR

VAR
    OpenLatch     : BOOL ;
END_VAR

BEGIN

NETWORK
TITLE = Valve Open Condition
// Open if (Manual OR Auto) AND SystemReady AND NOT EmergencyStop
      A(;
         O     #ManualOpen;
         O     #AutoOpen;
      );
      A     #SystemReady;
      AN    #EmergencyStop;
      =     #OpenLatch;

NETWORK
TITLE = Valve Output
// Drive physical valve output from latch
      A     #OpenLatch;
      =     #ValveOutput;

END_FUNCTION_BLOCK
```

---

## FC with Counter Logic (batch control)

```
FUNCTION "Batch Counter" : Void
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1

   VAR_INPUT
      PartSensor   : Bool;
      BatchReset   : Bool;
      BatchTarget  : Int;
   END_VAR

   VAR_OUTPUT
      BatchComplete : Bool;
      CurrentCount  : Int;
   END_VAR

BEGIN

NETWORK
TITLE = Count Parts on Sensor Rising Edge
// Increment counter C10 on each part detected
      A     #PartSensor;
      L     #BatchTarget;
      CU    C10;

NETWORK
TITLE = Reset Counter on Batch Reset
// Reset counter when batch reset command received
      A     #BatchReset;
      R     C10;

NETWORK
TITLE = Read Current Count
// Load counter value into output variable
      L     C10;
      T     #CurrentCount;

NETWORK
TITLE = Batch Complete Flag
// Set BatchComplete when counter has reached target
      A     C10;
      =     #BatchComplete;

END_FUNCTION
```

---

## FB with VAR_IN_OUT (read-modify-write parameter)

```
FUNCTION_BLOCK "PID Simple"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1

VAR_INPUT
    Setpoint    : REAL ;
    ProcessVal  : REAL ;
    Kp          : REAL ;
END_VAR

VAR_OUTPUT
    Error       : REAL ;
END_VAR

VAR_IN_OUT
    Output      : REAL ;     // Read current output, modify, write back
END_VAR

VAR
    PrevError   : REAL ;
END_VAR

BEGIN

NETWORK
TITLE = Calculate Process Error
// Error = Setpoint - ProcessValue
      L     #Setpoint;
      L     #ProcessVal;
      -R;
      T     #Error;

NETWORK
TITLE = Proportional Output Correction
// Output += Kp * Error
      L     #Kp;
      L     #Error;
      *R;
      L     #Output;
      +R;
      T     #Output;

NETWORK
TITLE = Store Previous Error
// Save error for next scan
      L     #Error;
      T     #PrevError;

END_FUNCTION_BLOCK
```

---

## Multiple Instances of the Same FB

When the same FB is used for multiple independent machines, each instance gets its own DB:

```
// In OB1 — two independent pump instances, same FB
NETWORK
TITLE = Pump 1 Control
// Call Pump Control FB for Pump 1 with its own instance DB
      CALL "Pump Control", "Pump1_DB"
      (  StartCmd                    := M10.0 ,
         StopCmd                     := M10.1 ,
         FaultInput                  := M10.2 ,
         PumpRun                     := M10.3
      );

NETWORK
TITLE = Pump 2 Control
// Call same FB for Pump 2 with a separate instance DB
      CALL "Pump Control", "Pump2_DB"
      (  StartCmd                    := M11.0 ,
         StopCmd                     := M11.1 ,
         FaultInput                  := M11.2 ,
         PumpRun                     := M11.3
      );
```

Each instance DB (`Pump1_DB`, `Pump2_DB`) must be declared separately as an FB Instance Data Block referencing `"Pump Control"`.

---

## RETAIN vs NON_RETAIN Data Blocks

Use `RETAIN` for data that must survive a CPU power cycle or STOP/RUN transition (e.g., recipe parameters, production counters). Use `NON_RETAIN` (default) for runtime state that can be re-initialised on restart.

```
DATA_BLOCK "Recipe_DB"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
RETAIN

   VAR
      BatchSize     : Int;
      TargetTemp    : Real;
      RecipeActive  : Bool;
   END_VAR

BEGIN

END_DATA_BLOCK
```

```
DATA_BLOCK "Runtime_DB"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
NON_RETAIN

   VAR
      CycleCount    : Int;
      LastFaultCode : Int;
   END_VAR

BEGIN

END_DATA_BLOCK
```

---

## Quick Reference: Timer Selection

| Context      | Timer type | Example                  |
|--------------|------------|--------------------------|
| Inside FC    | S5 Timer   | `SE T1;` with `S5T#5S`   |
| Inside FB    | IEC TON    | Declare `StartDelay : TON ;` in VAR |
| Off-delay    | S5: `SA`   | IEC: `TOF`               |
| Pulse        | S5: `SP`   | IEC: `TP`                |
| Retentive    | S5: `SS`/`SD` | IEC: store in RETAIN DB |