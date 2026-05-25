# Core Bluetooth 蓝牙开发

> 🎯 **本章目标**：理解 Core Bluetooth 框架的核心概念，掌握蓝牙设备扫描、连接、通信的完整流程，学会在 SwiftUI 项目中实现蓝牙功能，了解蓝牙权限与隐私合规要求。

---

## 1. Core Bluetooth 概述

### 1.1 蓝牙技术演进

蓝牙技术从诞生至今经历了多次重大升级，理解其演进有助于选择正确的开发方案：

| 版本 | 代号 | 核心特性 | 传输速率 | 典型应用 |
|------|------|----------|----------|----------|
| 1.0–3.0 | BR/EDR | 经典蓝牙，面向音频与文件传输 | 1–24 Mbps | 蓝牙耳机、文件传输 |
| 4.0 | BLE | 低功耗蓝牙，极简协议栈 | 1 Mbps | 健康手环、传感器 |
| 4.2 | BLE | 数据长度扩展、IPv6 支持 | 1 Mbps | IoT 设备 |
| 5.0 | BLE 5.0 | 2x 速率、4x 范围、8x 广播容量 | 2 Mbps | 室内定位、智能家居 |
| 5.1 | BLE 5.1 | 方向寻向（AoA/AoD） | 2 Mbps | 厘米级定位 |
| 5.3 | BLE 5.3 | 周期性广播增强、信道分类 | 2 Mbps | 音频共享、电子货架标签 |
| 5.4 | BLE 5.4 | PAwR（广播等距响应） | 2 Mbps | 电子货架标签、双向通信 |

> 💡 **提示**：Core Bluetooth 框架仅支持 BLE（蓝牙低功耗），不支持经典蓝牙（BR/EDR）。如果你的 App 需要使用 A2DP、HFP 等经典蓝牙协议，需要使用 ExternalAccessory 框架或 MultipeerConnectivity 框架。

### 1.2 Core Bluetooth 核心角色

Core Bluetooth 框架定义了两种核心角色：

| 角色 | 对应类 | 职责 | 典型场景 |
|------|--------|------|----------|
| **Central（中心）** | `CBCentralManager` | 扫描、连接、读写外围设备的数据 | iPhone App 连接智能手环 |
| **Peripheral（外围）** | `CBPeripheralManager` | 广播自身数据、响应中心设备的读写请求 | iPhone 模拟一个 BLE 传感器 |

```
┌─────────────────────────────────────────────────────┐
│              Core Bluetooth 通信模型                  │
│                                                     │
│   Central（中心）              Peripheral（外围）      │
│   ┌──────────────┐            ┌──────────────┐      │
│   │ CBCentral   │  扫描/连接   │ CBPeripheral │      │
│   │ Manager     │───────────→│              │      │
│   │             │  读取/写入   │ CBPeripheral │      │
│   │             │───────────→│ Manager      │      │
│   │             │  订阅通知    │              │      │
│   │             │←───────────│              │      │
│   └──────────────┘            └──────────────┘      │
└─────────────────────────────────────────────────────┘
```

### 1.3 BLE 核心概念：Service、Characteristic、Descriptor

BLE 设备的数据组织遵循严格的层级结构：

```
Peripheral（外围设备）
  └── Service（服务）— 一组相关功能的集合
        ├── Characteristic（特征）— 具体的数据项
        │     └── Descriptor（描述符）— 特征的元数据
        └── Characteristic
              └── Descriptor
```

| 概念 | 对应类 | 说明 | 类比 |
|------|--------|------|------|
| **Service** | `CBService` / `CBMutableService` | 一组相关特征的集合，由 UUID 标识 | 餐厅的菜单分类（主食区、饮品区） |
| **Characteristic** | `CBCharacteristic` / `CBMutableCharacteristic` | 具体的数据值，包含属性与权限 | 菜单上的具体菜品 |
| **Descriptor** | `CBDescriptor` / `CBMutableDescriptor` | 特征的附加描述信息 | 菜品的辣度标注、过敏原说明 |

> 💡 **生活类比**：蓝牙通信就像"餐厅点餐"——Central 是顾客，Peripheral 是餐厅，Service 是菜单分类（主食区、饮品区），Characteristic 是具体菜品（红烧肉、可乐），Descriptor 是菜品的附加说明（辣度、份量）。顾客先看菜单分类，再选具体菜品，最后下单（写入）或询问菜品信息（读取）。

特征的属性（Properties）决定了它可以支持的操作：

| 属性 | 说明 | 操作 |
|------|------|------|
| `read` | 可读取 | Central 可主动读取特征值 |
| `write` | 可写入 | Central 可向特征写入数据 |
| `writeWithoutResponse` | 无响应写入 | 写入后不需要 Peripheral 确认 |
| `notify` | 通知 | Peripheral 主动推送数据给 Central |
| `indicate` | 指示 | 类似 notify，但需要 Central 确认接收 |

### 1.4 BLE 通用服务 UUID

蓝牙 SIG 定义了许多标准服务 UUID（以 0x 开头的 16 位短 UUID），常见的包括：

| UUID | 服务名称 | 用途 |
|------|----------|------|
| 0x1800 | Generic Access | 设备名称、外观 |
| 0x1801 | Generic Attribute | GATT 服务 |
| 0x180A | Device Information | 厂商名、固件版本 |
| 0x180F | Battery Service | 电池电量 |
| 0x180D | Heart Rate | 心率数据 |
| 0x1816 | Cycling Speed and Cadence | 骑行速度与踏频 |

> 💡 **提示**：自定义服务和特征应使用 128 位 UUID（格式如 `XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX`）。可以使用 `uuidgen` 命令在终端生成唯一 UUID。

---

## 2. Central 角色开发

### 2.1 CBCentralManager 创建与状态检查

所有蓝牙操作的第一步是创建 `CBCentralManager` 实例并检查蓝牙状态：

```swift
import CoreBluetooth

class BluetoothManager: NSObject, CBCentralManagerDelegate, CBPeripheralDelegate {
    var centralManager: CBCentralManager!
    var discoveredPeripherals: [CBPeripheral] = []

    override init() {
        super.init()
        centralManager = CBCentralManager(delegate: self, queue: nil)
    }

    func centralManagerDidUpdateState(_ central: CBCentralManager) {
        switch central.state {
        case .poweredOn:
            print("蓝牙已开启，可以开始扫描")
        case .poweredOff:
            print("蓝牙已关闭，请开启蓝牙")
        case .unauthorized:
            print("App 未获得蓝牙使用权限")
        case .unsupported:
            print("设备不支持蓝牙")
        case .resetting:
            print("蓝牙正在重置")
        case .unknown:
            print("蓝牙状态未知")
        @unknown default:
            break
        }
    }
}
```

> ⚠️ **警告**：`CBCentralManager` 创建后不会立即可用，必须等待 `centralManagerDidUpdateState` 回调确认状态为 `.poweredOn` 后才能执行扫描等操作。在状态未就绪时调用扫描方法不会有任何效果。

### 2.2 扫描外围设备

```swift
func startScanning() {
    guard centralManager.state == .poweredOn else { return }

    centralManager.scanForPeripherals(
        withServices: [CBUUID(string: "180D")],
        options: [CBCentralManagerScanOptionAllowDuplicatesKey: true]
    )
}

func stopScanning() {
    centralManager.stopScan()
}

func centralManager(_ central: CBCentralManager,
                    didDiscover peripheral: CBPeripheral,
                    advertisementData: [String: Any],
                    rssi RSSI: NSNumber) {
    if !discoveredPeripherals.contains(where: { $0.identifier == peripheral.identifier }) {
        discoveredPeripherals.append(peripheral)
    }
}
```

`scanForPeripherals` 参数说明：

| 参数 | 说明 | 建议 |
|------|------|------|
| `withServices` | 过滤特定服务的 UUID，传 nil 扫描所有设备 | 传入目标服务 UUID 可减少无效扫描 |
| `options` | 扫描选项 | `AllowDuplicatesKey: true` 可接收重复广播 |

> 💡 **提示**：始终传入 `withServices` 参数来过滤扫描结果，这能显著降低功耗并减少无关设备的干扰。只有当你不确定目标设备的服务 UUID 时才传 nil。

### 2.3 连接与断开设备

```swift
func connect(to peripheral: CBPeripheral) {
    peripheral.delegate = self
    centralManager.connect(peripheral, options: nil)
}

func disconnect(peripheral: CBPeripheral) {
    centralManager.cancelPeripheralConnection(peripheral)
}

func centralManager(_ central: CBCentralManager,
                    didConnect peripheral: CBPeripheral) {
    print("已连接: \(peripheral.name ?? "未知设备")")
    peripheral.discoverServices(nil)
}

func centralManager(_ central: CBCentralManager,
                    didFailToConnect peripheral: CBPeripheral,
                    error: Error?) {
    print("连接失败: \(error?.localizedDescription ?? "未知错误")")
}

func centralManager(_ central: CBCentralManager,
                    didDisconnectPeripheral peripheral: CBPeripheral,
                    error: Error?) {
    print("已断开连接")
}
```

> ⚠️ **警告**：`cancelPeripheralConnection` 是异步操作，断开完成后会触发 `didDisconnectPeripheral` 回调。不要在断开后立即重新连接同一设备，应等待断开回调后再操作。

### 2.4 发现服务与特征

连接成功后，需要逐层发现服务和特征：

```swift
func peripheral(_ peripheral: CBPeripheral,
                didDiscoverServices error: Error?) {
    guard let services = peripheral.services else { return }

    for service in services {
        peripheral.discoverCharacteristics(nil, for: service)
    }
}

func peripheral(_ peripheral: CBPeripheral,
                didDiscoverCharacteristicsFor service: CBService,
                error: Error?) {
    guard let characteristics = service.characteristics else { return }

    for characteristic in characteristics {
        if characteristic.properties.contains(.read) {
            peripheral.readValue(for: characteristic)
        }
        if characteristic.properties.contains(.notify) {
            peripheral.setNotifyValue(true, for: characteristic)
        }
    }
}
```

> 💡 **提示**：`discoverServices` 和 `discoverCharacteristics` 的第一个参数传 nil 表示发现所有服务/特征。在生产环境中建议传入具体的 UUID 以提高效率，例如 `discoverServices([CBUUID(string: "180D")])`。

### 2.5 读取特征值

```swift
func peripheral(_ peripheral: CBPeripheral,
                didUpdateValueFor characteristic: CBCharacteristic,
                error: Error?) {
    guard let data = characteristic.value else { return }

    switch characteristic.uuid.uuidString {
    case "2A37":
        let heartRate = parseHeartRate(data: data)
        print("心率: \(heartRate) bpm")
    case "2A19":
        let batteryLevel = data.first.map { Int($0) } ?? 0
        print("电池电量: \(batteryLevel)%")
    default:
        break
    }
}

func parseHeartRate(data: Data) -> Int {
    let flags = data.first ?? 0
    if flags & 0x01 == 0 {
        return Int(data[1])
    } else {
        return Int(data[1]) | (Int(data[2]) << 8)
    }
}
```

### 2.6 写入特征值

```swift
func writeData(_ data: Data, to characteristic: CBCharacteristic) {
    if characteristic.properties.contains(.write) {
        peripheral.writeValue(data, for: characteristic, type: .withResponse)
    } else if characteristic.properties.contains(.writeWithoutResponse) {
        peripheral.writeValue(data, for: characteristic, type: .withoutResponse)
    }
}

func peripheral(_ peripheral: CBPeripheral,
                didWriteValueFor characteristic: CBCharacteristic,
                error: Error?) {
    if let error = error {
        print("写入失败: \(error.localizedDescription)")
    } else {
        print("写入成功")
    }
}
```

写入类型对比：

| 写入类型 | 说明 | 确认机制 | 适用场景 |
|----------|------|----------|----------|
| `.withResponse` | 需要外围设备确认 | 有 `didWriteValueFor` 回调 | 关键数据、配置命令 |
| `.withoutResponse` | 不需要确认 | 无回调，速度更快 | 高频数据、实时控制 |

### 2.7 订阅特征值通知

```swift
func subscribe(to characteristic: CBCharacteristic) {
    if characteristic.properties.contains(.notify) {
        peripheral.setNotifyValue(true, for: characteristic)
    }
}

func unsubscribe(from characteristic: CBCharacteristic) {
    peripheral.setNotifyValue(false, for: characteristic)
}

func peripheral(_ peripheral: CBPeripheral,
                didUpdateNotificationStateFor characteristic: CBCharacteristic,
                error: Error?) {
    if let error = error {
        print("订阅状态变更失败: \(error.localizedDescription)")
    } else {
        print("通知状态: \(characteristic.isNotifying ? "已开启" : "已关闭")")
    }
}
```

> 💡 **提示**：订阅通知是获取传感器实时数据的首选方式，比轮询读取（定期调用 `readValue`）更高效、更省电。心率带、温度传感器等设备通常通过 notify 方式推送数据。

### 2.8 完整代码示例：BLE 设备扫描与通信

```swift
import CoreBluetooth

class BLEDeviceManager: NSObject, ObservableObject {
    @Published var peripherals: [DiscoveredPeripheral] = []
    @Published var isConnected = false
    @Published var receivedData: String = ""

    private var centralManager: CBCentralManager!
    private var connectedPeripheral: CBPeripheral?
    private var targetCharacteristic: CBCharacteristic?

    struct DiscoveredPeripheral: Identifiable {
        let id: UUID
        let name: String
        let rssi: Int
        let peripheral: CBPeripheral
    }

    override init() {
        super.init()
        centralManager = CBCentralManager(delegate: self, queue: nil)
    }

    func startScan() {
        guard centralManager.state == .poweredOn else { return }
        peripherals.removeAll()
        centralManager.scanForPeripherals(withServices: nil, options: nil)
    }

    func stopScan() {
        centralManager.stopScan()
    }

    func connect(to peripheral: DiscoveredPeripheral) {
        connectedPeripheral = peripheral.peripheral
        connectedPeripheral?.delegate = self
        centralManager.connect(peripheral.peripheral, options: nil)
    }

    func sendCommand(_ command: String) {
        guard let characteristic = targetCharacteristic,
              let data = command.data(using: .utf8) else { return }
        connectedPeripheral?.writeValue(data, for: characteristic, type: .withResponse)
    }
}

extension BLEDeviceManager: CBCentralManagerDelegate {
    func centralManagerDidUpdateState(_ central: CBCentralManager) {
        if central.state == .poweredOn {
            startScan()
        }
    }

    func centralManager(_ central: CBCentralManager,
                        didDiscover peripheral: CBPeripheral,
                        advertisementData: [String: Any],
                        rssi RSSI: NSNumber) {
        if !peripherals.contains(where: { $0.id == peripheral.identifier }) {
            let device = DiscoveredPeripheral(
                id: peripheral.identifier,
                name: peripheral.name ?? "未知设备",
                rssi: RSSI.intValue,
                peripheral: peripheral
            )
            peripherals.append(device)
        }
    }

    func centralManager(_ central: CBCentralManager,
                        didConnect peripheral: CBPeripheral) {
        isConnected = true
        peripheral.discoverServices(nil)
    }

    func centralManager(_ central: CBCentralManager,
                        didDisconnectPeripheral peripheral: CBPeripheral,
                        error: Error?) {
        isConnected = false
        targetCharacteristic = nil
    }
}

extension BLEDeviceManager: CBPeripheralDelegate {
    func peripheral(_ peripheral: CBPeripheral,
                    didDiscoverServices error: Error?) {
        guard let services = peripheral.services else { return }
        for service in services {
            peripheral.discoverCharacteristics(nil, for: service)
        }
    }

    func peripheral(_ peripheral: CBPeripheral,
                    didDiscoverCharacteristicsFor service: CBService,
                    error: Error?) {
        guard let characteristics = service.characteristics else { return }
        for characteristic in characteristics {
            if characteristic.properties.contains(.notify) {
                peripheral.setNotifyValue(true, for: characteristic)
            }
            if characteristic.properties.contains(.read) {
                peripheral.readValue(for: characteristic)
            }
            if characteristic.properties.contains(.write) || characteristic.properties.contains(.writeWithoutResponse) {
                targetCharacteristic = characteristic
            }
        }
    }

    func peripheral(_ peripheral: CBPeripheral,
                    didUpdateValueFor characteristic: CBCharacteristic,
                    error: Error?) {
        guard let data = characteristic.value else { return }
        receivedData = String(data: data, encoding: .utf8) ?? data.map { String(format: "%02x", $0) }.joined()
    }
}
```

---

## 3. Peripheral 角色开发（选读）

### 3.1 CBPeripheralManager 创建

大多数 App 只需要 Central 角色，但某些场景（如 iPhone 作为信标、设备间通信）需要 Peripheral 角色：

```swift
class PeripheralManager: NSObject, CBPeripheralManagerDelegate {
    var peripheralManager: CBPeripheralManager!
    var transferCharacteristic: CBMutableCharacteristic!

    override init() {
        super.init()
        peripheralManager = CBPeripheralManager(delegate: self, queue: nil)
    }

    func peripheralManagerDidUpdateState(_ peripheral: CBPeripheralManager) {
        if peripheral.state == .poweredOn {
            setupServices()
        }
    }
}
```

### 3.2 添加服务与特征

```swift
func setupServices() {
    let serviceUUID = CBUUID(string: "E20A39F4-73F5-4BC4-A12F-17D1AD07A961")
    let characteristicUUID = CBUUID(string: "08590F7E-DB05-467E-8757-72F6FAD56EFD")

    transferCharacteristic = CBMutableCharacteristic(
        type: characteristicUUID,
        properties: [.read, .write, .notify],
        value: nil,
        permissions: [.readable, .writeable]
    )

    let service = CBMutableService(type: serviceUUID, primary: true)
    service.characteristics = [transferCharacteristic]

    peripheralManager.add(service)
}
```

特征属性与权限的对应关系：

| 属性（Properties） | 权限（Permissions） | 说明 |
|---------------------|---------------------|------|
| `.read` | `.readable` | 允许 Central 读取 |
| `.write` | `.writeable` | 允许 Central 写入 |
| `.notify` / `.indicate` | 无需额外权限 | 允许推送通知 |
| `.writeWithoutResponse` | `.writeable` | 允许无响应写入 |

### 3.3 广播数据

```swift
func startAdvertising() {
    peripheralManager.startAdvertising([
        CBAdvertisementDataServiceUUIDsKey: [CBUUID(string: "E20A39F4-73F5-4BC4-A12F-17D1AD07A961")],
        CBAdvertisementDataLocalNameKey: "MyBLEDevice"
    ])
}

func stopAdvertising() {
    peripheralManager.stopAdvertising()
}

func peripheralManagerDidStartAdvertising(_ peripheral: CBPeripheralManager,
                                           error: Error?) {
    if let error = error {
        print("广播启动失败: \(error.localizedDescription)")
    } else {
        print("广播已启动")
    }
}
```

> ⚠️ **警告**：广播数据有长度限制（通常 31 字节）。如果广播数据过长，iOS 会自动截断 `localName` 等字段。建议只广播服务 UUID，设备名称放在扫描响应中。

### 3.4 响应读写请求

```swift
func peripheralManager(_ peripheral: CBPeripheralManager,
                       didReceiveRead request: CBATTRequest) {
    guard request.characteristic.uuid == transferCharacteristic.uuid else {
        peripheralManager.respond(to: request, withResult: .attributeNotFound)
        return
    }

    guard let value = transferCharacteristic.value else {
        peripheralManager.respond(to: request, withResult: .invalidAttributeValueLength)
        return
    }

    request.value = value
    peripheralManager.respond(to: request, withResult: .success)
}

func peripheralManager(_ peripheral: CBPeripheralManager,
                       didReceiveWrite requests: [CBATTRequest]) {
    for request in requests {
        guard request.characteristic.uuid == transferCharacteristic.uuid else {
            peripheralManager.respond(to: request, withResult: .attributeNotFound)
            return
        }
        transferCharacteristic.value = request.value
    }
    peripheralManager.respond(to: requests[0], withResult: .success)

    notifySubscribers()
}

func notifySubscribers() {
    guard let value = transferCharacteristic.value else { return }
    peripheralManager.updateValue(
        value,
        for: transferCharacteristic,
        onSubscribedCentrals: nil
    )
}

func peripheralManager(_ peripheral: CBPeripheralManager,
                       central: CBCentral,
                       didSubscribeTo characteristic: CBCharacteristic) {
    print("Central 已订阅: \(central.identifier)")
}

func peripheralManager(_ peripheral: CBPeripheralManager,
                       central: CBCentral,
                       didUnsubscribeFrom characteristic: CBCharacteristic) {
    print("Central 已取消订阅: \(central.identifier)")
}
```

---

## 4. SwiftUI 集成

### 4.1 @Observable 封装蓝牙管理器

使用 Swift 5.9 的 Observation 框架封装蓝牙管理器，比传统的 `ObservableObject` + `@Published` 更高效：

```swift
import CoreBluetooth
import Observation

@Observable
class BluetoothViewModel: NSObject {
    var isBluetoothReady = false
    var isScanning = false
    var devices: [BLEDevice] = []
    var connectedDevice: BLEDevice?
    var receivedMessages: [String] = []
    var connectionState: ConnectionState = .disconnected

    enum ConnectionState {
        case disconnected
        case connecting
        case connected
        case disconnecting
    }

    struct BLEDevice: Identifiable {
        let id: UUID
        let name: String
        let rssi: Int
        let peripheral: CBPeripheral
    }

    private var centralManager: CBCentralManager!
    private var targetCharacteristic: CBCharacteristic?

    override init() {
        super.init()
        centralManager = CBCentralManager(delegate: self, queue: nil)
    }

    func startScan() {
        guard centralManager.state == .poweredOn else { return }
        devices.removeAll()
        isScanning = true
        centralManager.scanForPeripherals(withServices: nil, options: nil)
    }

    func stopScan() {
        centralManager.stopScan()
        isScanning = false
    }

    func connect(_ device: BLEDevice) {
        connectionState = .connecting
        device.peripheral.delegate = self
        centralManager.connect(device.peripheral, options: nil)
    }

    func disconnect() {
        guard let device = connectedDevice else { return }
        connectionState = .disconnecting
        centralManager.cancelPeripheralConnection(device.peripheral)
    }

    func send(_ message: String) {
        guard let characteristic = targetCharacteristic,
              let data = message.data(using: .utf8) else { return }
        connectedDevice?.peripheral.writeValue(data, for: characteristic, type: .withResponse)
    }
}
```

### 4.2 蓝牙设备列表界面

```swift
import SwiftUI

struct DeviceListView: View {
    @State private var viewModel = BluetoothViewModel()

    var body: some View {
        NavigationStack {
            Group {
                if viewModel.isBluetoothReady {
                    deviceList
                } else {
                    ContentUnavailableView(
                        "蓝牙未就绪",
                        systemImage: "bluetooth",
                        description: Text("请在系统设置中开启蓝牙")
                    )
                }
            }
            .navigationTitle("蓝牙设备")
            .toolbar {
                ToolbarItem(placement: .topBarTrailing) {
                    Button(viewModel.isScanning ? "停止" : "扫描") {
                        if viewModel.isScanning {
                            viewModel.stopScan()
                        } else {
                            viewModel.startScan()
                        }
                    }
                }
            }
        }
    }

    private var deviceList: some View {
        List(viewModel.devices) { device in
            DeviceRow(device: device, isConnected: viewModel.connectedDevice?.id == device.id) {
                viewModel.connect(device)
            }
        }
        .overlay {
            if viewModel.devices.isEmpty && !viewModel.isScanning {
                ContentUnavailableView(
                    "未发现设备",
                    systemImage: "antenna.radiowaves.left.and.right",
                    description: Text("点击右上角\"扫描\"搜索附近蓝牙设备")
                )
            }
        }
    }
}

struct DeviceRow: View {
    let device: BluetoothViewModel.BLEDevice
    let isConnected: Bool
    let onTap: () -> Void

    var body: some View {
        Button(action: onTap) {
            HStack {
                Image(systemName: isConnected ? "bluetooth.connected" : "bluetooth")
                    .foregroundStyle(isConnected ? .blue : .gray)
                VStack(alignment: .leading) {
                    Text(device.name)
                        .font(.headline)
                    Text("RSSI: \(device.rssi) dBm")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
                Spacer()
                if isConnected {
                    Text("已连接")
                        .font(.caption)
                        .foregroundStyle(.white)
                        .padding(.horizontal, 8)
                        .padding(.vertical, 4)
                        .background(.blue, in: Capsule())
                }
            }
        }
    }
}
```

### 4.3 设备详情与控制界面

```swift
struct DeviceDetailView: View {
    @Bindable var viewModel: BluetoothViewModel

    var body: some View {
        ScrollView {
            VStack(spacing: 20) {
                connectionStatusCard
                messageArea
                controlArea
            }
            .padding()
        }
        .navigationTitle(viewModel.connectedDevice?.name ?? "设备详情")
        .navigationBarTitleDisplayMode(.inline)
        .toolbar {
            ToolbarItem(placement: .topBarTrailing) {
                Button("断开") {
                    viewModel.disconnect()
                }
                .foregroundStyle(.red)
            }
        }
    }

    private var connectionStatusCard: some View {
        HStack {
            Circle()
                .fill(viewModel.connectionState == .connected ? .green : .orange)
                .frame(width: 12, height: 12)
            Text(statusText)
                .font(.subheadline)
            Spacer()
        }
        .padding()
        .background(.ultraThinMaterial, in: RoundedRectangle(cornerRadius: 12))
    }

    private var messageArea: some View {
        VStack(alignment: .leading, spacing: 8) {
            Text("接收数据")
                .font(.headline)
            if viewModel.receivedMessages.isEmpty {
                Text("暂无数据")
                    .foregroundStyle(.secondary)
                    .frame(maxWidth: .infinity, minHeight: 100)
                    .background(.gray.opacity(0.1), in: RoundedRectangle(cornerRadius: 8))
            } else {
                ScrollViewReader { proxy in
                    ScrollView {
                        LazyVStack(alignment: .leading, spacing: 4) {
                            ForEach(Array(viewModel.receivedMessages.enumerated()), id: \.offset) { index, message in
                                Text(message)
                                    .font(.system(.caption, design: .monospaced))
                                    .textSelection(.enabled)
                                    .id(index)
                            }
                        }
                    }
                    .frame(maxHeight: 200)
                    .onChange(of: viewModel.receivedMessages.count) { _, _ in
                        proxy.scrollTo(viewModel.receivedMessages.count - 1)
                    }
                }
            }
        }
    }

    private var controlArea: some View {
        VStack(spacing: 12) {
            Button("发送指令") {
                viewModel.send("HELLO")
            }
            .buttonStyle(.borderedProminent)
            .disabled(viewModel.connectionState != .connected)
        }
    }

    private var statusText: String {
        switch viewModel.connectionState {
        case .disconnected: return "未连接"
        case .connecting: return "连接中..."
        case .connected: return "已连接"
        case .disconnecting: return "断开中..."
        }
    }
}
```

### 4.4 蓝牙状态变化的 UI 响应

```swift
struct BluetoothStatusView: View {
    @State private var viewModel = BluetoothViewModel()

    var body: some View {
        VStack(spacing: 16) {
            statusIcon
            statusText
            actionButton
        }
        .padding()
    }

    private var statusIcon: some View {
        Image(systemName: iconForState)
            .font(.system(size: 60))
            .foregroundStyle(colorForState)
            .symbolEffect(.pulse, options: .repeating, isActive: viewModel.connectionState == .connecting)
    }

    private var statusText: some View {
        Text(textForState)
            .font(.title3)
            .foregroundStyle(.secondary)
    }

    private var actionButton: some View {
        Button(buttonText) {
            switch viewModel.connectionState {
            case .disconnected:
                viewModel.startScan()
            case .connected:
                viewModel.disconnect()
            default:
                break
            }
        }
        .buttonStyle(.borderedProminent)
    }

    private var iconForState: String {
        switch viewModel.connectionState {
        case .disconnected: return "bluetooth"
        case .connecting: return "bluetooth"
        case .connected: return "bluetooth.connected"
        case .disconnecting: return "bluetooth"
        }
    }

    private var colorForState: Color {
        switch viewModel.connectionState {
        case .disconnected: return .gray
        case .connecting: return .orange
        case .connected: return .blue
        case .disconnecting: return .orange
        }
    }

    private var textForState: String {
        switch viewModel.connectionState {
        case .disconnected: return "蓝牙未连接"
        case .connecting: return "正在连接..."
        case .connected: return "蓝牙已连接"
        case .disconnecting: return "正在断开..."
        }
    }

    private var buttonText: String {
        switch viewModel.connectionState {
        case .disconnected: return "扫描设备"
        case .connected: return "断开连接"
        default: return "请等待"
        }
    }
}
```

### 4.5 后台蓝牙通信

iOS 支持后台蓝牙操作，但需要在 Info.plist 中声明 `bluetooth-central` 或 `bluetooth-peripheral` 后台模式：

| 后台模式 | Key | 支持的操作 |
|----------|-----|-----------|
| `bluetooth-central` | `UIBackgroundModes` → `bluetooth-central` | 后台扫描、连接、读写、订阅通知 |
| `bluetooth-peripheral` | `UIBackgroundModes` → `bluetooth-peripheral` | 后台广播、响应读写请求 |

> ⚠️ **警告**：后台蓝牙扫描行为与前台不同——iOS 会降低扫描频率，且无法按服务 UUID 过滤广播数据。后台连接事件会通过 `connectionEvent` 通知而非 `didDiscover` 回调。

```swift
func setupBackgroundHandling() {
    let options: [String: Any] = [
        CBCentralManagerOptionRestoreIdentifierKey: "MyAppCentralManager"
    ]
    centralManager = CBCentralManager(delegate: self, queue: nil, options: options)
}

func centralManager(_ central: CBCentralManager,
                    willRestoreState dict: [String: Any]) {
    if let peripherals = dict[CBCentralManagerRestoredStatePeripheralsKey] as? [CBPeripheral] {
        for peripheral in peripherals {
            peripheral.delegate = self
            if peripheral.state == .connected {
                connectedDevice = BluetoothViewModel.BLEDevice(
                    id: peripheral.identifier,
                    name: peripheral.name ?? "恢复的设备",
                    rssi: 0,
                    peripheral: peripheral
                )
            }
        }
    }
}
```

> 💡 **提示**：使用 `CBCentralManagerOptionRestoreIdentifierKey` 可以在 App 被系统终止后重新启动时恢复蓝牙连接状态，避免丢失已建立的连接。这是后台蓝牙 App 的必备功能。

---

## 5. 蓝牙权限与隐私合规

### 5.1 Info.plist 权限描述

使用 Core Bluetooth 必须在 Info.plist 中添加权限描述：

| Key | 说明 | 必需 |
|-----|------|------|
| `NSBluetoothAlwaysUsageDescription` | 蓝牙始终使用权限描述（iOS 13+） | ✅ 必需 |
| `NSBluetoothPeripheralUsageDescription` | 蓝牙外围设备权限描述（iOS 12 及以下） | 仅需兼容旧系统时添加 |

```xml
<key>NSBluetoothAlwaysUsageDescription</key>
<string>此App需要使用蓝牙连接您的智能手环，以同步运动和健康数据</string>
```

> ⚠️ **警告**：权限描述必须具体说明 App 使用蓝牙的目的，不能使用模糊描述如"需要蓝牙"。Apple 审核会拒绝权限描述不清晰的 App。

### 5.2 隐私清单中的蓝牙声明

从 2024 年起，Apple 要求使用特定 API 的 App 提交隐私清单（PrivacyInfo.xcprivacy）。蓝牙相关的声明：

```xml
<key>NSPrivacyAccessedAPICategories</key>
<array>
    <dict>
        <key>NSPrivacyAccessedAPIType</key>
        <string>NSPrivacyAccessedAPICategoryBluetooth</string>
        <key>NSPrivacyAccessedAPITypeReasons</key>
        <array>
            <string>2B6F4V.9LU</string>
        </array>
    </dict>
</array>
```

### 5.3 Apple 审核对蓝牙使用的审查要点

| 审查要点 | 说明 | 应对策略 |
|----------|------|----------|
| 权限描述合理性 | 描述必须与 App 功能直接相关 | 明确说明蓝牙用途，如"连接智能体重秤" |
| 后台模式必要性 | 声明后台蓝牙模式需要充分理由 | 仅在确实需要后台通信时声明 |
| 数据用途透明 | 蓝牙收集的数据用途必须公开 | 在隐私政策中说明蓝牙数据用途 |
| 最小权限原则 | 不应请求超出功能需要的蓝牙权限 | 仅扫描目标服务 UUID，不扫描所有设备 |
| 不收集位置信息 | 蓝牙不能作为位置追踪手段 | 不要通过蓝牙信标推断用户位置 |

### 5.4 国内合规要求

在中国大陆发布蓝牙相关 App 还需注意：

| 要求 | 说明 |
|------|------|
| 蓝牙设备 SRRC 认证 | 连接的硬件设备需通过无线电型号核准 |
| 个人信息保护 | 蓝牙设备标识符属于个人信息，需遵守《个人信息保护法》 |
| 数据出境 | 蓝牙数据传至境外服务器需进行安全评估 |
| App 备案 | 含蓝牙功能的 App 在备案时需如实申报 |
| 权限最小化 | 不得因用户拒绝蓝牙权限而拒绝提供基本服务 |

> 💡 **提示**：如果你的 App 同时使用蓝牙和定位功能，需要分别请求权限，并在用户拒绝蓝牙权限时提供降级功能（如手动输入设备序列号），不能强制要求蓝牙权限。

---

## 6. 常见问题与最佳实践

### 6.1 连接不稳定排查

| 现象 | 可能原因 | 解决方案 |
|------|----------|----------|
| 连接后立即断开 | 特征发现失败、服务不匹配 | 检查服务 UUID 是否正确，添加错误处理 |
| 数据传输中断 | 信号弱、干扰 | 检查 RSSI 值，缩短通信距离 |
| 无法发现设备 | 广播间隔过长、UUID 过滤错误 | 放宽扫描条件，增加扫描时间 |
| 连接超时 | 设备休眠、广播关闭 | 确保目标设备处于广播状态 |
| 重连失败 | 系统缓存了旧连接信息 | 调用 `cancelPeripheralConnection` 后等待回调再重连 |

```swift
func robustConnect(to peripheral: CBPeripheral) {
    peripheral.delegate = self

    let options: [String: Any] = [
        CBConnectPeripheralOptionNotifyOnDisconnectionKey: true
    ]
    centralManager.connect(peripheral, options: options)
}

func centralManager(_ central: CBCentralManager,
                    didDisconnectPeripheral peripheral: CBPeripheral,
                    error: Error?) {
    if let error = error as? CBError,
       error.code == .connectionTimeout || error.code == .connectionFailed {
        DispatchQueue.main.asyncAfter(deadline: .now() + 2.0) { [weak self] in
            self?.centralManager.connect(peripheral, options: nil)
        }
    }
}
```

### 6.2 数据传输优化

BLE 单次传输数据量有限（默认 20 字节，协商后可达 512 字节），大数据需要分包：

```swift
func sendData(_ data: Data, in chunksOf mtu: Int = 20,
              to characteristic: CBCharacteristic,
              via peripheral: CBPeripheral) {
    let totalPackets = (data.count + mtu - 1) / mtu

    for index in 0..<totalPackets {
        let startIndex = index * mtu
        let endIndex = min(startIndex + mtu, data.count)
        let chunk = data[startIndex..<endIndex]

        peripheral.writeValue(Data(chunk), for: characteristic, type: .withoutResponse)
    }
}
```

| 优化策略 | 说明 |
|----------|------|
| 协商 MTU | 连接后请求更大 MTU，减少分包数量 |
| 使用 `.withoutResponse` | 减少确认开销，提高吞吐量 |
| 合并小包 | 将多个小数据合并为一个包发送 |
| 压缩数据 | 使用二进制协议替代 JSON 文本 |
| 避免频繁读写 | 合理设计通信协议，减少交互次数 |

### 6.3 电池消耗优化

| 策略 | 实现方式 | 效果 |
|------|----------|------|
| 按需扫描 | 传入 `withServices` 过滤，设置扫描超时 | 显著降低功耗 |
| 及时停止扫描 | 连接设备后立即 `stopScan()` | 避免持续扫描耗电 |
| 使用通知替代轮询 | `setNotifyValue` 替代定时 `readValue` | 减少无效通信 |
| 断开不用的设备 | 不使用时 `cancelPeripheralConnection` | 释放资源 |
| 降低连接参数 | 请求更大的连接间隔 | 降低通信频率 |

```swift
func startOptimizedScan() {
    guard centralManager.state == .poweredOn else { return }

    centralManager.scanForPeripherals(
        withServices: [CBUUID(string: "180D")],
        options: [CBCentralManagerScanOptionAllowDuplicatesKey: false]
    )

    DispatchQueue.main.asyncAfter(deadline: .now() + 10.0) { [weak self] in
        self?.centralManager.stopScan()
    }
}
```

### 6.4 多设备连接管理

Core Bluetooth 支持同时连接多个外围设备，但需要注意管理：

```swift
class MultiDeviceManager: NSObject, CBCentralManagerDelegate, CBPeripheralDelegate {
    var centralManager: CBCentralManager!
    var connectedDevices: [UUID: CBPeripheral] = [:]
    var deviceData: [UUID: [String: Any]] = [:]

    func connectMultiple(_ peripherals: [CBPeripheral]) {
        for peripheral in peripherals {
            peripheral.delegate = self
            centralManager.connect(peripheral, options: nil)
        }
    }

    func peripheral(_ peripheral: CBPeripheral,
                    didUpdateValueFor characteristic: CBCharacteristic,
                    error: Error?) {
        guard let data = characteristic.value else { return }
        deviceData[peripheral.identifier]?["lastUpdate"] = Date()
        deviceData[peripheral.identifier]?["data"] = data
    }
}
```

> 💡 **提示**：iOS 对同时连接的 BLE 设备数量没有官方限制，但实际测试中超过 6-8 个设备可能出现连接不稳定。建议根据 App 场景合理控制连接数量。

### 6.5 蓝牙调试工具

| 工具 | 平台 | 用途 |
|------|------|------|
| **nRF Connect** | iOS / Android | 扫描 BLE 设备、读写特征、日志分析 |
| **LightBlue** | iOS | 简洁的 BLE 调试工具，支持模拟外围设备 |
| **Bluetooth Explorer** | macOS (Xcode 附加工具) | 监控蓝牙连接状态与数据 |
| **Xcode Console** | macOS | 查看 `cblog` 系统日志 |
| **Wireshark** | macOS | 抓取蓝牙数据包进行深度分析 |

> 💡 **提示**：开发阶段强烈建议安装 nRF Connect，它可以独立验证 BLE 设备的功能是否正常，帮助区分问题是出在设备端还是 App 端。

---

## 小结

本章系统介绍了 Core Bluetooth 框架的开发方法，核心知识点总结如下：

| 主题 | 核心要点 |
|------|----------|
| **BLE 概念** | Service → Characteristic → Descriptor 层级结构；Central 与 Peripheral 两种角色 |
| **Central 开发** | 创建 CBCentralManager → 扫描 → 连接 → 发现服务/特征 → 读写/订阅 |
| **Peripheral 开发** | 创建 CBPeripheralManager → 添加服务/特征 → 广播 → 响应读写请求 |
| **SwiftUI 集成** | @Observable 封装、状态驱动的 UI、后台蓝牙与状态恢复 |
| **权限合规** | NSBluetoothAlwaysUsageDescription 必填、隐私清单声明、审核要点、国内合规 |
| **最佳实践** | 按需扫描、通知替代轮询、分包传输、MTU 协商、调试工具使用 |

> 💡 **提示**：Core Bluetooth 开发的关键在于理解异步回调链——每个操作（扫描、连接、发现、读写）都是异步的，必须在上一步的回调中触发下一步操作，不能同步调用。

← [HealthKit 与传感器](./HealthKit与传感器.md) | [深度链接与 Universal Links](./深度链接与Universal-Links.md) →