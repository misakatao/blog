---
title: "使用 Swift 实现 Tesla BLE Key：从协议到工程落地"
description: "基于 CoreBluetooth、CryptoKit、SwiftProtobuf 实现 Tesla 蓝牙车钥匙的完整技术路径，涵盖 ECDH 密钥交换、AES-GCM 加密、Session 管理、状态机设计与后台蓝牙"
date: 2026-05-21T10:00:00+08:00
slug: tesla-ble-key-implementation
image: "cover.svg"
categories:
    - iOS
tags:
    - Swift
    - BLE
    - CoreBluetooth
    - CryptoKit
    - Tesla
---

上一篇文章从原理层面解释了 Tesla Phone Key 为什么能做到靠近自动解锁。这篇文章换一个角度：如果要用 Swift 从零实现一个 Tesla BLE Key，技术上需要做哪些事情。内容涵盖 BLE 通信、ECDH 密钥交换、AES-GCM 加密、Session 管理、protobuf 协议、状态机设计、后台蓝牙与安全模型。

<!--more-->

## 技术栈总览

| 模块 | 技术 |
|------|------|
| 蓝牙通信 | CoreBluetooth |
| 密钥交换 | CryptoKit（ECDH / P-256） |
| 数据加密 | AES-GCM |
| 协议序列化 | SwiftProtobuf |
| 并发模型 | Swift Concurrency（Actor） |
| 密钥存储 | Keychain / Secure Enclave |

整体架构分层如下：

```text
┌──────────────────────────┐
│        iOS App           │
├──────────────────────────┤
│    BLE Session Layer     │
├──────────────────────────┤
│   AES-GCM Encryption    │
├──────────────────────────┤
│   ECDH Key Agreement    │
├──────────────────────────┤
│  Tesla BLE Protocol      │
├──────────────────────────┤
│     CoreBluetooth        │
├──────────────────────────┤
│   Bluetooth Hardware     │
└──────────────────────────┘
```

## BLE 通信基础

Tesla 车辆作为 BLE Peripheral 持续广播，手机作为 Central 扫描并发起连接。车辆暴露专用 GATT Service，包含三个核心 Characteristic：

| Characteristic | 用途 |
|---|---|
| Write | 手机向车辆发送命令 |
| Notify | 车辆向手机推送数据 |
| Read | 状态同步 |

完整的 BLE 交互流程：

```text
扫描车辆 → 建立连接 → 服务发现 → 特征发现
    → ECDH 密钥交换 → 建立 Session Key
    → Challenge 验证 → AES-GCM 加密通信
    → 下发 Vehicle Command
```

### 扫描与连接

```swift
import CoreBluetooth

final class TeslaBLEManager: NSObject {
    private var central: CBCentralManager!

    override init() {
        super.init()
        central = CBCentralManager(delegate: self, queue: .main)
    }
}

extension TeslaBLEManager: CBCentralManagerDelegate {
    func centralManagerDidUpdateState(_ central: CBCentralManager) {
        guard central.state == .poweredOn else { return }
        central.scanForPeripherals(withServices: nil, options: nil)
    }

    func centralManager(
        _ central: CBCentralManager,
        didDiscover peripheral: CBPeripheral,
        advertisementData: [String: Any],
        rssi RSSI: NSNumber
    ) {
        // 解析 Manufacturer Data 识别 Tesla 车辆
        if let data = advertisementData[CBAdvertisementDataManufacturerDataKey] as? Data {
            // 解析 Vehicle Identifier、BLE Capability、Session Info
        }
    }
}
```

识别到目标车辆后发起连接，连接成功后依次发现 Service 和 Characteristic，然后订阅 Notify：

```swift
peripheral.setNotifyValue(true, for: notifyCharacteristic)
```

## ECDH 密钥交换

Tesla BLE 通信并非明文传输，而是基于 Elliptic Curve Diffie-Hellman 协商出共享密钥，再派生 AES Session Key。

### 生成密钥对

```swift
import CryptoKit

let privateKey = P256.KeyAgreement.PrivateKey()
let publicKey = privateKey.publicKey
```

### 协商共享密钥

手机将自己的 Public Key 发送给车辆，车辆返回 Vehicle Public Key。双方各自计算共享密钥：

```swift
let sharedSecret = try privateKey.sharedSecretFromKeyAgreement(
    with: vehiclePublicKey
)
```

### 派生 AES 对称密钥

使用 HKDF 从共享密钥派生出用于加密通信的 AES-256 Key：

```swift
let symmetricKey = sharedSecret.hkdfDerivedSymmetricKey(
    using: SHA256.self,
    salt: Data(),
    sharedInfo: Data(),
    outputByteCount: 32
)
```

## AES-GCM 加密通信

Session Key 建立后，所有后续通信都通过 AES-GCM 加密。GCM 模式同时提供加密和完整性校验。

```swift
// 加密
let sealed = try AES.GCM.seal(plaintextData, using: symmetricKey)

// 解密
let decrypted = try AES.GCM.open(sealedBox, using: symmetricKey)
```

Nonce 必须每次唯一，重复使用会导致 AES-GCM 完全失效：

```swift
let nonce = try AES.GCM.Nonce()
```

## Challenge-Response 认证

密钥交换完成后，车辆会发送一段随机 Challenge，手机需要用私钥签名后返回：

```text
Vehicle → Challenge（随机数）
Phone   → Sign(Challenge, PrivateKey)
Vehicle → Verify(Signature, PublicKey)
```

签名使用 ECDSA：

```swift
let signature = try privateKey.signature(for: challengeData)
```

认证通过后，Session 正式建立，可以开始发送加密命令。

## 完整认证时序

```text
Phone                               Tesla Vehicle
  |                                        |
  | -------- BLE Scan ------------------> |
  | <------- Advertisement -------------- |
  |                                        |
  | -------- Connect -------------------> |
  | <------- Connected ------------------ |
  |                                        |
  | -------- Discover Services ---------> |
  | <------- Characteristics ------------ |
  |                                        |
  | -------- Send Public Key -----------> |
  | <------- Vehicle Public Key --------- |
  |                                        |
  | ===== ECDH Shared Secret =====        |
  | ===== HKDF Session Key ======         |
  |                                        |
  | <------- Challenge ------------------ |
  | -------- Signed Challenge ----------> |
  |                                        |
  | <------- Auth Success --------------- |
  |                                        |
  | -------- AES-GCM Command -----------> |
  | <------- Command ACK ---------------- |
```

## Session 管理

Tesla BLE 维护一个 Session 对象，包含会话 ID、加密密钥和防重放计数器：

```swift
struct VehicleSession {
    let sessionId: UInt32
    var counter: UInt32
    let key: SymmetricKey
    var lastSeen: Date
}
```

### Counter 同步

车辆要求 Counter 严格单调递增，否则拒绝命令。使用 Actor 保证线程安全：

```swift
actor CounterManager {
    private var counter: UInt32 = 0

    func next() -> UInt32 {
        defer { counter += 1 }
        return counter
    }
}
```

## protobuf 协议

Tesla Vehicle Command Protocol 使用 protobuf 序列化。典型的命令结构：

```protobuf
message BLECommand {
    uint32 opcode = 1;
    bytes payload = 2;
    uint32 counter = 3;
    bytes nonce = 4;
}
```

常见命令：

| Command | 作用 |
|---|---|
| unlock | 解锁车门 |
| lock | 上锁 |
| open_trunk | 开后备箱 |
| start_drive | 启动车辆 |
| honk | 鸣笛 |

Swift 侧使用 SwiftProtobuf 生成模型代码：

```bash
protoc --swift_out=. command.proto
```

## BLE 数据包结构与分包

Tesla BLE 数据包并非文本协议，而是二进制结构：

```text
[Header][Opcode][Counter][Payload][MAC]
```

由于 BLE MTU 限制（通常 20～185 bytes），较大的命令需要分包传输：

```swift
struct PacketFragment {
    let packetId: UInt16
    let index: UInt8
    let total: UInt8
    let payload: Data
}
```

发送时拆包，接收时按 packetId 和 index 重组。

## 状态机设计

Tesla BLE 交互本质上是一个状态机，推荐用枚举建模：

```swift
enum BLEState {
    case idle
    case scanning
    case connecting
    case discovering
    case exchangingKey
    case authenticating
    case ready
    case disconnected
    case failed(Error)
}
```

状态流转：

```text
idle → scanning → connecting → discovering
    → exchangingKey → authenticating → ready
```

BLE 不适合并发写入，命令需要串行发送：

```swift
actor CommandQueue {
    // 串行发送，每个 command 独立 timeout
}
```

## 自动解锁：RSSI 滤波

自动解锁的核心判断依据是 RSSI 信号强度。但原始 RSSI 波动很大，直接使用会导致误触发。推荐使用指数移动平均（EMA）平滑：

```swift
filteredRSSI = alpha * newRSSI + (1 - alpha) * oldRSSI
```

解锁条件：

```text
filteredRSSI > threshold AND 认证成功 AND 持续存在
```

需要注意，RSSI 到实际距离的映射非常不准确，受人体遮挡、金属反射、天线方向、车体结构等因素影响极大。

## 密钥存储与安全

私钥绝对不能存普通文件或 UserDefaults。必须使用 Keychain 或 Secure Enclave。

### Secure Enclave

```swift
let privateKey = try SecureEnclave.P256.KeyAgreement.PrivateKey()
```

优势：私钥不可导出、硬件级保护、支持 Face ID 联动、系统级隔离。

### Keychain 配置

车钥匙需要锁屏状态下也能使用，因此 Keychain 访问级别应设为：

```swift
kSecAttrAccessibleAfterFirstUnlock
```

### 安全红线

| 风险 | 后果 |
|------|------|
| 私钥泄露 | 车辆被盗 |
| 无 Counter | Replay Attack |
| 无认证 | 任意控制 |
| 日志泄露 Key | 完全失陷 |

日志系统中绝对不能打印密钥、Session、明文 Command。

## 后台蓝牙

iOS 后台 BLE 是整个实现中最大的工程挑战。

### 开启后台模式

Xcode → Signing & Capabilities → Background Modes → Uses Bluetooth LE accessories

### 状态恢复

App 被系统杀死后，可以通过恢复标识符重新接管 BLE 会话：

```swift
CBCentralManager(
    delegate: self,
    queue: nil,
    options: [
        CBCentralManagerOptionRestoreIdentifierKey: "tesla_ble_restore"
    ]
)
```

### iOS 后台限制

| 场景 | 限制 |
|---|---|
| 后台扫描 | 频率降低 |
| App 被杀 | BLE 停止 |
| 长时间扫描 | 系统节电 |

Tesla 官方 App 拥有特殊 entitlement，第三方应用无法做到同等稳定性。

## 推荐工程结构

```text
TeslaBLE/
 ├── BLE/
 │    ├── BLEManager
 │    ├── PeripheralScanner
 │    └── CharacteristicRouter
 ├── Crypto/
 │    ├── ECDHManager
 │    ├── AESGCMCipher
 │    └── KeyStore
 ├── Protocol/
 │    ├── PacketBuilder
 │    ├── PacketParser
 │    └── CommandEncoder
 ├── Vehicle/
 │    ├── VehicleSession
 │    ├── VehicleAuthenticator
 │    └── VehicleCommander
 ├── Session/
 │    └── CounterManager
 └── Transport/
      └── CommandQueue
```

推荐使用 Actor 隔离并发状态：

```text
BLEActor     → Scan / Connect / Notify
CryptoActor  → ECDH / AES
SessionActor → Counter / Session
```

## 推荐开发顺序

1. BLE 扫描与连接
2. Service / Characteristic 发现
3. Notify 订阅与数据接收
4. protobuf 集成
5. ECDH 密钥交换
6. AES-GCM 加密解密
7. Session 管理与 Counter 同步
8. Command 发送与 ACK 处理
9. 自动解锁逻辑
10. 后台蓝牙与状态恢复

## 测试策略

**BLE Mock。** 模拟 Vehicle Peripheral，不依赖真车即可跑通主流程。

**单元测试重点。** AES 加解密、Counter 递增、protobuf 编解码、Packet 分包重组。

**压力测试重点。** 长时间后台运行、RSSI 波动场景、断连重连、Session 恢复。

## 小结

Tesla BLE Key 的本质是：

```text
BLE + ECDH + AES-GCM + Challenge-Response
```

真正困难的不是 BLE 通信本身，而是协议实现的正确性、安全模型的严密性、Session 同步的可靠性、以及 iOS 后台蓝牙的稳定性。这是一个车联网、BLE 安全、移动系统底层三者交叉的综合工程。

## 延伸阅读

- [Apple CoreBluetooth Programming Guide](https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/AboutCoreBluetooth/Introduction.html)
- [Apple CryptoKit Documentation](https://developer.apple.com/documentation/cryptokit)
- [Swift Concurrency（Actor / AsyncStream）](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [SwiftProtobuf](https://github.com/apple/swift-protobuf)
- [Tesla Vehicle Command Protocol](https://github.com/teslamotors/vehicle-command)
- [BLE Reverse Engineering](https://github.com/teslamotors/vehicle-command/blob/main/pkg/protocol/protocol.md)
