
Memory:

(ABC lines of decoder go to A5, A6, and A7 respectively /E1 -> /M_request
 /E2 -> Reset; E3 -> +V)

0000-1FFF = BIOS ROM
2000-3FFF = Expansion port
4000-5FFF = Expansion port
6000-7FFF = SRAM (1K)
8000-9FFF = Cart
A000-BFFF = Cart
C000-DFFF = Cart
E000-FFFF = Cart



| Start | End  | Description    |
| ----- | ---- | -------------- |
| 0000  | 1FFF | BIOS ROM       |
| 2000  | 3FFF | Expansion Port |
| 4000  | 5FFF | Expansion Port |
|       |      |                |

IO Ports

| Start | End | Description           | Read/Write? |
| ----- | --- | --------------------- | ----------- |
| $00   | $1F | USUSED                | R           |
| $00   | $1F | USUSED                | W           |
| $20   | $3F | USUSED                | R           |
| $20   | $3F | USUSED                | W           |
| $40   | $5F | USUSED                | R           |
| $40   | $5F | Memory Bank Switch    | W           |
| $60   | $7F | UART                  | R           |
| $60   | $7F | UART                  | W           |
| $80   | $9F | SD Card               | R           |
| $80   | $9F | SD Card               | W           |
| $A0   | $BF | Video (VDC) registers | R           |
| $A0   | $BF | Video (VDC) registers | W           |
| $C0   | $DF | USB1                  | R           |
| $C0   | $DF | RESET/POWER           | W           |
| $E0   | $FF | USB2                  | R           |
| $E0   | $FF | Sound                 | W           |
