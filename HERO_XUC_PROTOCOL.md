# HERO XUC 串口通信协议

## 串口参数

- 下位机接口：USART1（PA9-TX / PA10-RX）
- 波特率：460800
- 字节序：小端
- CRC16：反射多项式 `0x8408`，初值 `0xffff`，CRC低字节先发送

## 下位机发送给 NUC：状态帧（28字节）

| 字节 | 字段 | 类型 |
| --- | --- | --- |
| 0 | header = `0x5A` | uint8 |
| 1 | detect_color/reset_tracker/reserved | 位域 |
| 2..5 | roll | float32 LE |
| 6..9 | pitch | float32 LE |
| 10..13 | yaw | float32 LE |
| 14..17 | aim_x | float32 LE |
| 18..21 | aim_y | float32 LE |
| 22..25 | aim_z | float32 LE |
| 26..27 | CRC16（覆盖0..25） | uint16 LE |

## NUC发送给下位机：控制帧v2（31字节）

| 字节 | 字段 | 类型 |
| --- | --- | --- |
| 0 | header = `0xA5` | uint8 |
| 1..4 | pitch（度） | float32 LE |
| 5..8 | yaw（度） | float32 LE |
| 9..12 | yaw_diff | float32 LE |
| 13..16 | pitch_diff（度） | float32 LE |
| 17..20 | distance | float32 LE |
| 21 | fireadvice bit0 | uint8 |
| 22..24 | reserved，发送端置0 | uint8[3] |
| 25..28 | v_y | float32 LE |
| 29..30 | CRC16（覆盖0..28） | uint16 LE |

NUC端组帧时先填写前29字节，再计算CRC16，并将CRC低、高字节分别写入29、30。下位机只有在帧头和CRC均正确时才更新目标。

## 旧29字节帧兼容

下位机暂时兼容CRC位于27..28的旧帧，以保证旧NUC仍能提供yaw、pitch目标。旧帧的`v_y`与CRC重叠，因此下位机收到旧帧时固定令`v_y = 0`。NUC升级为31字节帧后可获得完整`v_y`。

## 自瞄与发射安全

- 遥控器 `UP + DOWN` 进入AUTO模式。
- 连续50个有效目标帧后才设置 `track_flag`；超过100ms未收到有效帧即取消跟踪。
- AUTO模式只注入云台yaw/pitch目标。
- `fireadvice`只解析，不接入摩擦轮或拨弹控制；发射仍由遥控器门控。
