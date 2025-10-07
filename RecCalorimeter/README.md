# ECAL Bit Layout

`DecodeEcal()` unpacks a compact integer encoding used for calorimeter hit identification.

It uses bit masks and shifts to recover physical indices and material type.

The mapping supports signed coordinates, layer/slice segmentation, and dual-crystal symmetry.

```cpp
void DecodeEcal (long long int ihitchan,
                 int& idet, int& ix, int& iy,
                 int& islice, int& ilayer,
                 int& wc, int& type)
```

```pgsql
Bit index (LSB → MSB)
┌───────────────────────────────────────────────────────────────────────────────────┐
│  63 ……………… 26 │25│24│23│22│21│20│19│18│17│16│15│14│13│12│11│10│9│8│7│6│5│4│3│2│1│0│
└───────────────────────────────────────────────────────────────────────────────────┘
         ↑               ↑        ↑       ↑     ↑______________↑   ↑_________↑ ↑___↑
         │               │        │       │                    │             │     │
         │               │        │       │                    │             │     └── idet (bits 0–2)
         │               │        │       │                    │             └──────── ix (bits 3–8)
         │               │        │       │                    └────────────────────── iy (bits 10–15)
         │               │        │       └─────────────────────────────────────────── islice (bits 17–19)
         │               │        └─────────────────────────────────────────────────── ilayer (bits 20–22)
         │               └──────────────────────────────────────────────────────────── wc (bits 23–25)
         └──────────────────────────────────────────────────────────────────────────── unused/reserved
```

| Field    | Bits  | Mask   | Meaning                     | Range  | Notes                         |
| -------- | ----- | ------ | --------------------------- | ------ | ----------------------------- |
| `idet`   | 0–2   | `0x07` | Detector ID                 | 0–7    | Which calorimeter module      |
| `ix`     | 3–8   | `0x3F` | x-coordinate index          | −32…31 | Converted from unsigned 6-bit |
| `iy`     | 10–15 | `0x3F` | y-coordinate index          | −32…31 | Same signed trick             |
| `islice` | 17–19 | `0x07` | Slice number within layer   | 0–7    | “z” segmentation              |
| `ilayer` | 20–22 | `0x07` | Layer index                 | 0–7    | Front vs back crystal layer   |
| `wc`     | 23–25 | `0x07` | wafer/crystal ID            | 0–7    | Sub-element group             |
| —        | ≥26   | —      | unused or future extensions | —      | May hold timestamps, etc.     |

_(Bits 9, 16, etc. might be unused or reserved.)_


## Type Decoding Logic

Classify the `(ilayer, islice)` combination into a material type:

| ilayer | islice | type | Description |
| :----: | :----: | :--: | :---------- |
|    0   |    1   |   1  | photodiode  |
|    0   |    2   |   4  | resin       |
|    0   |    3   |   4  | cookie      |
|    0   |    4   |   2  | crystal     |
|    0   |    5   |   3  | air         |
|    1   |    1   |   2  | crystal     |
|    1   |    2   |   4  | cookie      |
|    1   |    3   |   4  | resin       |
|    1   |    4   |   1  | photodiode  |
|    1   |    5   |   3  | air         |

This reflects a mirrored geometry:

 * Layer 0: PD → resin → cookie → crystal → air
 * Layer 1: crystal → cookie → resin → PD → air

So the type tells the digitization code what material region the hit belongs to.

The `(ilayer, islice)` combination map to a component type (physical material or role) as follows:

```cpp
type = 0; // default

// Layer 0 pattern
if ((ilayer == 0) && (islice == 1)) type = 1;  // photodiode (pd)
if ((ilayer == 0) && (islice == 2)) type = 4;  // resin
if ((ilayer == 0) && (islice == 3)) type = 4;  // cookie
if ((ilayer == 0) && (islice == 4)) type = 2;  // crystal
if ((ilayer == 0) && (islice == 5)) type = 3;  // air

// Layer 1 pattern (reverse order)
if ((ilayer == 1) && (islice == 1)) type = 2;  // crystal
if ((ilayer == 1) && (islice == 2)) type = 4;  // cookie
if ((ilayer == 1) && (islice == 3)) type = 4;  // resin
if ((ilayer == 1) && (islice == 4)) type = 1;  // pd
if ((ilayer == 1) && (islice == 5)) type = 3;  // air
```
* type encodes what material or sub-element the hit corresponds to:

   * 1: photodiode
   * 2: crystal
   * 3: air
   * 4: resin or cookie

The geometry alternates between layer 0 and layer 1, reversing the stack order (since crystals and PDs alternate on opposite sides of the dual calorimeter).

# Fiber Channel Bit Layout

```pgsql
  Bit index (LSB → MSB)
  ┌────────────────────────────────────────────────────────────────────────────┐
  │63 ……………………………………………………………… 38 │37│36│35│34│33│32│31│30 … 20│19│18 … 8│7 … 0│
  └────────────────────────────────────────────────────────────────────────────┘
         ↑                               ↑        ↑        ↑          ↑     ↑
         │                               │        │        │          │     │
         │                               │        │        │          │     └── idet (bits 0–7) detector ID
         │                               │        │        │          └──────── ilayer (bits 8–19)
         │                               │        │        └─────────────────── itube (bits 20–31)
         │                               │        └──────────────────────────── iair (bits 32–34)
         │                               └───────────────────────────────────── itype (bits 35–37)
         └───────────────────────────────────────────────────────────────────── reserved / unused (bits 38–63)
```

## Field Summary

| Field    | Bits  | Mask    | Meaning             | Notes                                           |
| -------- | ----- | ------- | ------------------- | ----------------------------------------------- |
| `idet`   | 0–7   | `0xFF`  | Detector ID         | identifies which fiber module                   |
| `ilayer` | 8–19  | `0xFFF` | Layer index         | up to 4095 layers supported                     |
| `itube`  | 20–31 | `0xFFF` | Tube index          | up to 4095 tubes per layer                      |
| `iair`   | 32–34 | `0x7`   | Air/empty indicator | 0=solid, 1–2=air/hole                           |
| `itype`  | 35–37 | `0x7`   | Fiber type          | 0=generic, 1=scint, 2=quartz, 3/4=photodetector |
| —        | 38–63 | —       | reserved / unused   | future expansion                                |


# Sampling Calorimeter Bit Layout
```pgsql
  Bit index (LSB → MSB)
  ┌────────────────────────────────────────────────────────────────────────────┐
  │63 ………………………………………………………… 45 │44│43│42│41│40│39│38 … 27│26 … 15│14 … 3│2 … 0│
  └────────────────────────────────────────────────────────────────────────────┘
         ↑                       ↑        ↑           ↑       ↑       ↑     ↑
         │                       │        │           │       │       │     │
         │                       │        │           │       │       │     └── idet (bits 0–2) detector ID
         │                       │        │           │       │       └──────── iy (bits 3–14) y-coordinate
         │                       │        │           │       └──────────────── ix (bits 15–26) x-coordinate
         │                       │        │           └──────────────────────── ilayer (bits 27–38) layer index
         │                       │        └──────────────────────────────────── ibox2 (bits 39–40) box/module ID
         │                       └───────────────────────────────────────────── islice (bits 41–44) slice within layer
         └───────────────────────────────────────────────────────────────────── reserved / unused (bits 45–63)
```

## Field Summary

| Field    | Bits  | Mask    | Meaning            | Notes            |
| -------- | ----- | ------- | ------------------ | ---------------- |
| `idet`   | 0–2   | `0x07`  | Detector ID        | 0–7              |
| `iy`     | 3–14  | `0xFFF` | y-coordinate       | 12-bit           |
| `ix`     | 15–26 | `0xFFF` | x-coordinate       | 12-bit           |
| `ilayer` | 27–38 | `0xFFF` | Layer index        | 12-bit           |
| `ibox2`  | 39–40 | `0x03`  | Box/module ID      | 0–3              |
| `islice` | 41–44 | `0xF`   | Slice within layer | 0–15             |
| —        | 45–63 | —       | Reserved / unused  | Future expansion |
