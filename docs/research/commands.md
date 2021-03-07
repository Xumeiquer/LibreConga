# Commands

These are the commands for Conga.

The values are represented as Little Endian, but they are transmited as Big Endian.

| OpCode (hex) | OpCode (dec) | Conga                        | Comment        |
| ------------ | ------------ | ---------------------------- | -------------- |
| 0x07 0xd5    | 2005         | [5490](models/ping.md#5490)  | Ping Request   |
| 0x07 0xd6    | 2006         | [5490](models/ping.md#5490)  | Ping Response  |
| 0x0b 0xb9    | 3001         | [5490](models/login.md#5490) | Login Request  |
| 0x0b 0xba    | 3002         | [5490](models/login.md#5490) | Login Response |
