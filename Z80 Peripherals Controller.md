# Z80 I/O Subsystem — SD + Dual-USB-HID via SIO + ATmega1284P 💾(:retro)

## 1) Overall architecture

### 1.1 Goals
- Keep **Z80-side integration simple** and CP/M-friendly.
- Separate **HID (tiny + urgent)** from **SD (chunky + not urgent)**.
- Avoid “MCU on the Z80 bus” complexity: the MCU is **behind SIO UART links**.

### 1.2 Major blocks
- **Z80 host**
- **2× Z80 SIO/0** (4 serial channels total)
- **Z80 CTC** (system tick generator; also useful for housekeeping timers)
- **ATmega1284P** (5V) as “I/O co-processor”
- **MAX3421E** USB host controller (SPI)
- **Full-speed USB hub** (2 downstream ports)
- **SD card socket** (SPI, 3.3V)

### 1.3 Channel assignment (recommended)
You have 4 SIO channels; this mapping keeps things clean:

- **SIO0 Channel A**: Console UART (to FT232H / external serial)
- **SIO0 Channel B**: User UART (spare “real serial port”)
- **SIO1 Channel A**: **USB HID stream** (MCU USART0)
- **SIO1 Channel B**: **SD service channel** (MCU USART1)

> This is just the *role mapping*. Port addresses are whatever your decode chooses.

### 1.4 Data paths

#### HID path (read-only to Z80)
`USB devices → Hub → MAX3421E → (SPI) → ATmega1284P → (USART0) → SIO_USB → Z80`

- Z80 never “commands” HID. It just receives **events**.
- Z80 uses **SIO RX interrupts (IM2)** to pull bytes into a ring buffer.

#### SD path (commands + data)
`Z80 → SIO_SD → (USART1) → ATmega1284P → (SPI) → SD card`

- MCU holds exactly **one 512-byte working sector** in RAM.
- Host uses **LOAD / READSEQ / READRND / WRITESEQ / WRITERND / FLUSH / GETSTAT**.
- No unsolicited bytes from MCU (prevents FIFO poisoning).

---

## 2) Clocking (locked)

### 2.1 Master clock source
- Use a single **14.7456 MHz oscillator** as the master reference (`CLK_14M7456`).

Rationale:
- 14.7456 MHz divides cleanly to common UART baud clocks (classic choice).
- One clean reference keeps SIO baud generation and any derived clocks deterministic.

### 2.2 Derived clocks (divide-down)
Derive these from `CLK_14M7456`:

- **SIO baud reference clock**: `CLK_BAUD = 14.7456 MHz / 4 = 3.6864 MHz`
  - Feed `CLK_BAUD` to the SIO’s `TRxC` (or your chosen SIO clock pins) as the baud source.

- **Optional “SPI bit clock reference”** (if you want a board-wide SPI clock rail for experiments):
  - `SPI_CLK_REF = 14.7456 MHz / N` (pick N to land where you want)
  - Note: MCU SPI can generate its own SCK; this is only if you want external clocking / test options.

### 2.3 Divider implementation (YOLO-simple)
- A single small divider IC is enough (no clock-tree drama):
  - e.g., a binary counter/divider (74HC/HCT class) producing /2, /4, /8…
- Route `CLK_BAUD` to the few consumers (SIO(s), optional headers/dividers).
- Keep it sane: short-ish traces, don’t do anything that looks like you’re trying to broadcast RF on purpose.

---

## 3) Parts list (silicon only)

### Z80-side
- 2× **Zilog Z80 SIO/0**
- 1× **Zilog Z80 CTC**

### MCU-side
- 1× **ATmega1284P** (5V)

### USB
- 1× **MAX3421E** (USB host controller, SPI)
- 1× **Full-speed USB hub** (e.g., TUSB2046B class; 2 downstream ports)
- 1× **Dual USB power switch** (for VBUS1/VBUS2) *(optional if you want; recommended in practice)*

### SD
- SD card socket (mechanical)

### Clocking
- 1× **14.7456 MHz oscillator** (`CLK_14M7456`)
- 1× **divider** (counter/divider IC) to generate at least `/4` (`CLK_BAUD = 3.6864 MHz`)

### Level shifting (minimal)
- 1× **SN74LVC125A** (or similar) for MCU(5V) → 3.3V SPI outputs (SCK/MOSI/CS/RST)
- 1× **74HCT125** (or similar) for 3.3V → 5V inputs if you want guaranteed VIH margin on MISO/INT

---

## 4) Net list

### 4.1 Core Z80 bus + interrupts
- `A[15:0]`, `D[7:0]`
- `/IORQ`, `/RD`, `/WR`, `/M1`, `/RESET`, `CLK`
- IM2 daisy chain:
  - `/INT` to CPU
  - `IEI/IEO` chain across interrupting devices (CTC, SIOs, VDP, etc.)

### 4.2 Clock nets
- `CLK_14M7456` (oscillator output)
- `CLK_BAUD` (= `CLK_14M7456/4` = 3.6864 MHz) to SIO baud clock pins

### 4.3 Z80 ↔ SIO nets (per channel)
Per SIO channel:
- `SIOxA_TXD`, `SIOxA_RXD`, `SIOxA_CTS/RTS` (optional)
- `SIOxB_TXD`, `SIOxB_RXD`, `SIOxB_CTS/RTS` (optional)

### 4.4 SIO ↔ ATmega1284P nets (two UART links)
**USB HID channel (USART0)**
- `MCU_TX0 → SIO_USB_RXD`
- `MCU_RX0 ← SIO_USB_TXD` *(optional / future-proof; can be left unused)*
- `GND`

**SD channel (USART1)**
- `MCU_TX1 → SIO_SD_RXD`
- `MCU_RX1 ← SIO_SD_TXD` *(optional / future-proof; can be left unused)*
- `GND`

### 4.5 ATmega1284P ↔ MAX3421E (SPI + sideband)
Shared SPI:
- `SPI_SCK`, `SPI_MOSI`, `SPI_MISO`

USB-specific:
- `USB_CS#`
- `USB_INT`  (MAX3421E → MCU)
- `USB_RST#` (MCU → MAX3421E)

### 4.6 MAX3421E ↔ Hub ↔ USB ports
- Upstream: `USB_UP_DP`, `USB_UP_DM`
- Downstream Port 1: `USB1_DP`, `USB1_DM`, `VBUS1_5V`, `GND`
- Downstream Port 2: `USB2_DP`, `USB2_DM`, `VBUS2_5V`, `GND`

### 4.7 ATmega1284P ↔ SD card (SPI)
- Shared: `SPI_SCK`, `SPI_MOSI`, `SPI_MISO`
- SD-specific:
  - `SD_CS#`
  - `SD_CD#` (optional card detect)
- Power:
  - `+3V3_PERI`, `GND`

---

## 5) Programming model for Z80

### 5.1 HID (USART0 / SIO_USB): 2-byte event records (commandless)
HID is a pure **byte stream** from MCU to Z80. Z80 reads it via SIO RX ISR.

#### Record format (2 bytes)
- **Byte0 = META**
  - `b7..b4 TYPE` (0..15)
  - `b3..b2 PORT` (00=port1, 01=port2)
  - `b1..b0 FLAGS` (currently 00)
- **Byte1 = DATA**
  - “which key / which button / which modifier bitfield”

Suggested TYPE values:
- `0x0 KEY_DOWN`   DATA = keycode
- `0x1 KEY_UP`     DATA = keycode
- `0x2 MODS`       DATA = mods bitfield
- `0x3 DPAD_STATE` DATA = bitfield
- `0x4 DEV_PRESENT` DATA = 0/1

MODS bitfield (DATA for TYPE=MODS):
- b0 LCTRL, b1 LSHIFT, b2 LALT, b3 LGUI, b4 RCTRL, b5 RSHIFT, b6 RALT, b7 RGUI

> Keycode policy: keep it “neutral” (HID usage IDs or your own table). CP/M can translate in software.

#### Z80 ISR sketch (ring buffer)
```asm
; Called from IM2 when SIO_USB RX interrupt fires.
; Goal: drain SIO RX FIFO quickly, store bytes into ring.

SIO_USB_ISR:
.loop:
    in   a,(SIO_USB_RR0)
    bit  0,a              ; RX char available?
    jr   z,.done
    in   a,(SIO_USB_DATA)
    call RING_PUSH_A
    jr   .loop
.done:
    ei
    reti
````

Foreground parser consumes in 2-byte chunks:

```asm
; if ring has >=2 bytes:
;   meta = pop(); data = pop();
;   switch(meta>>4) ...
```

---

### 5.2 SD (USART1 / SIO_SD): single-sector RAM buffer protocol 
#### Key idea

- MCU holds **one sector** in RAM.    
- Host decides when to load, read sequentially, patch, write sequentially, and flush.    
- MCU never emits unsolicited status bytes.    

#### Opcodes (1 byte each)

- `0x10 LOAD` + `LBA32` (5 bytes total)    
- `0x11 READSEQ` (1 byte cmd, MCU sends 512 bytes)    
- `0x12 READRND` + `IDX16` (3 bytes cmd, MCU sends 1 byte)    
- `0x13 WRITESEQ` + `LEN16` + data (3+LEN bytes)    
- `0x14 WRITERND` + `IDX16` + data8 (4 bytes)    
- `0x15 FLUSH` (1 byte)    
- `0x16 GETSTAT` (1 byte, MCU sends 1 status byte)    

Status byte (`GETSTAT` response):
- `0x00 OK`    
- nonzero = error (pick a tiny set; example)    
    - `0x01 NOSECTOR`        
    - `0x02 RANGE`        
    - `0x10 SD_IO` (read/write failure)        

Locked policy:

- On `LOAD(new_lba)`: if current buffer is dirty → **flush it** → then load new.    
- `READSEQ`: sends exactly **512 bytes**; host counts; no trailer.    

#### Z80 helper routines (sketch)

Assume you already have `SIO_SD_PUTC` and `SIO_SD_GETC` for the channel.

**LOAD**

```asm
; HL:DE = LBA32 (little-endian or your choice; just be consistent)
SD_LOAD:
    ld   a,0x10
    call SIO_SD_PUTC
    ld   a,l  : call SIO_SD_PUTC
    ld   a,h  : call SIO_SD_PUTC
    ld   a,e  : call SIO_SD_PUTC
    ld   a,d  : call SIO_SD_PUTC
    ret
```

**GETSTAT**

```asm
SD_GETSTAT:
    ld   a,0x16
    call SIO_SD_PUTC
    call SIO_SD_GETC      ; returns A=status
    ret
```

**READSEQ → buffer[512]**

```asm
; IX points to destination buffer
SD_READSEQ_512:
    ld   a,0x11
    call SIO_SD_PUTC
    ld   b,0              ; 256
    ld   c,2              ; 2 blocks of 256 = 512
.next256:
    push bc
    ld   b,0
.loop256:
    call SIO_SD_GETC
    ld   (ix+0),a
    inc  ix
    djnz .loop256
    pop  bc
    dec  c
    jr   nz,.next256
    ret
```

**WRITESEQ (512) from buffer**

```asm
; IX points to source buffer
SD_WRITESEQ_512:
    ld   a,0x13
    call SIO_SD_PUTC
    ; LEN16 = 0x0200
    xor  a
    call SIO_SD_PUTC      ; LEN lo = 0x00
    ld   a,0x02
    call SIO_SD_PUTC      ; LEN hi = 0x02

    ld   b,0
    ld   c,2
.next256w:
    push bc
    ld   b,0
.loop256w:
    ld   a,(ix+0)
    inc  ix
    call SIO_SD_PUTC
    djnz .loop256w
    pop  bc
    dec  c
    jr   nz,.next256w
    ret
```

**FLUSH**

```asm
SD_FLUSH:
    ld   a,0x15
    call SIO_SD_PUTC
    ret
```

**Recommended calling pattern**

- Read sector:    
    - `SD_LOAD(lba)` → `SD_GETSTAT()` (optional check)        
    - `SD_READSEQ_512()` → (use data)
        
- Write whole sector:    
    - `SD_LOAD(lba)` (optional if you want read-modify-write)        
    - `SD_WRITESEQ_512()`        
    - `SD_FLUSH()` → `SD_GETSTAT()` (check)
        
---

## 6) MCU firmware notes (non-normative, just to align expectations)

- FreeRTOS is optional; with ATmega1284P it’s feasible.
    
- Priority rule:
    
    - HID task is “fast + frequent” (driven by MAX3421E INT)        
    - SD task is “slow + chunky” (SPI block transfers)        
- Since HID and SD have **separate UARTs**, you don’t get priority inversion on the Z80 side.
    

---

## 7) What’s deliberately _not_ specified 

- Exact keycode mapping (ASCII vs HID usages vs your own table): **software choice**    
- Any checksum/CRC/framing on UART: **nope**    
- Fancy caches: **nope** (only one working sector)    
- “Perfect reliability under abuse”: **nope** (bench machine, not a product)

