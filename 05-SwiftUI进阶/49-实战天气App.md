# 49-实战②：完成「天气 App」

> 🎯 **本章目标**：从零开始，用 Spec 驱动开发的方式，完成一个完整的天气 App。你将学会：调用第三方 API 获取真实天气数据、使用 CoreLocation 获取用户位置、设计数据模型与网络层、构建完整的 UI 界面、处理加载/错误等状态，最终产出一个可以展示在作品集中的完整项目。

---

## 项目概述

这是我们的第二个实战项目。和第一个待办事项 App 不同，天气 App 需要和**外部服务**交互——获取真实天气数据，这让它更接近真实的商业 App 开发。

💡 **通俗理解**：如果说待办事项 App 是"自己做菜"（数据都在本地），那天气 App 就是"点外卖"（数据从服务器获取）。你需要学会如何"下单"（发请求）、"接餐"（解析数据）、"上桌"（展示 UI）。

### 功能清单

| 功能 | 优先级 | 说明 |
|------|--------|------|
| 获取当前位置天气 | P0 | 自动定位并显示当前天气 |
| 显示当前温度/天气状况 | P0 | 温度、天气描述、图标 |
| 未来几天天气预报 | P0 | 5 天天气预报列表 |
| 搜索城市 | P1 | 输入城市名搜索天气 |
| 下拉刷新 | P1 | 下拉刷新天气数据 |
| 体感温度/湿度/风速 | P2 | 详细天气信息 |

### 最终效果预览

完成后的 App 界面大致如下：

```
┌─────────────────────────┐
│     📍 北京市            │
│                         │
│        ☀️               │
│       28°               │
│      晴天               │
│   体感 30° | 湿度 45%    │
│                         │
│  ─────────────────────  │
│  📅 未来天气预报         │
│  ─────────────────────  │
│  周一  ☀️  25° / 18°    │
│  周二  🌤  23° / 17°    │
│  周三  🌧  20° / 15°    │
│  周四  ⛅  22° / 16°    │
│  周五  ☀️  26° / 19°    │
└─────────────────────────┘
```

### API 选择

我们使用 **OpenWeatherMap** 的免费 API：

| 特性 | 说明 |
|------|------|
| **名称** | OpenWeatherMap |
| **免费额度** | 每分钟 60 次请求，每天 1000 次 |
| **注册地址** | https://openweathermap.org/api |
| **需要信用卡** | 不需要 |
| **数据丰富度** | 当前天气、5 天预报、地理编码 |

#### 注册获取 API Key

1. 访问 https://openweathermap.org/
2. 点击 "Sign Up" 注册账号
3. 登录后进入 "My API keys" 页面
4. 复制你的 API Key（一串长字符串）

⚠️ **警告**：注册后 API Key 可能需要等待 1~2 小时才能生效。如果请求返回 401 错误，请耐心等待。

#### API 文档简介

我们主要使用两个 API：

| API | 用途 | URL |
|-----|------|-----|
| Current Weather | 获取当前天气 | `https://api.openweathermap.org/data/2.5/weather` |
| 5 Day Forecast | 获取 5 天预报 | `https://api.openweathermap.org/data/2.5/forecast` |

请求参数示例：

```
https://api.openweathermap.org/data/2.5/weather?lat=39.9&lon=116.4&appid=你的API_KEY&units=metric&lang=zh_cn
```

| 参数 | 说明 |
|------|------|
| `lat` / `lon` | 经纬度 |
| `appid` | 你的 API Key |
| `units=metric` | 使用摄氏度 |
| `lang=zh_cn` | 返回中文天气描述 |

---

## 用 Spec 驱动开发天气 App

💡 **通俗理解**：Spec 驱动开发就是"先写需求文档，再写代码"——就像盖房子先画图纸，而不是边想边盖。这样能避免做到一半发现方向错了。

### 编写 PRD

PRD（Product Requirements Document）是产品需求文档，用来说明"这个 App 要做什么"。

#### 简版 PRD

```markdown
# 天气 App PRD

## 产品名称
简天气（SimpleWeather）

## 产品定位
一款简洁、美观的天气应用，帮助用户快速了解当前和未来天气状况。

## 核心用户
需要查看天气的普通用户

## 功能需求

### P0 - 必须实现
1. 自动获取用户当前位置，显示当前天气
2. 显示当前温度、天气状况、天气图标
3. 显示未来 5 天天气预报

### P1 - 应该实现
4. 搜索城市功能
5. 下拉刷新天气数据

### P2 - 可以实现
6. 显示体感温度、湿度、风速等详细信息
7. 天气背景随天气状况变化

## 非功能需求
- 启动时间 < 2 秒
- 网络请求失败时有友好的错误提示
- 支持深色模式
```

### 拆解任务

将 PRD 拆解为可执行的开发任务：

| 序号 | 任务 | 预计时间 | 依赖 |
|------|------|---------|------|
| 1 | 创建项目，搭建目录结构 | 15 分钟 | 无 |
| 2 | 设计数据模型（WeatherResponse 等） | 30 分钟 | 无 |
| 3 | 封装网络层（WeatherService） | 45 分钟 | 2 |
| 4 | 集成 CoreLocation 获取位置 | 30 分钟 | 无 |
| 5 | 开发主界面——当前天气展示 | 60 分钟 | 2, 3, 4 |
| 6 | 开发天气预报列表 | 45 分钟 | 2, 3 |
| 7 | 实现搜索城市功能 | 30 分钟 | 3 |
| 8 | 实现下拉刷新和加载状态 | 30 分钟 | 5 |
| 9 | 错误处理与重试 | 30 分钟 | 3, 8 |
| 10 | UI 美化与深色模式适配 | 30 分钟 | 5, 6 |

💡 **提示**：任务拆解的关键是"小步快跑"——每个任务控制在 30~60 分钟内，完成后立刻能看到效果，保持成就感。

---

## 创建项目与基础架构

### 新建项目

1. 打开 Xcode → Create a new Xcode project
2. 选择 **iOS → App**
3. 填写项目信息：
   - Product Name: `SimpleWeather`
   - Interface: **SwiftUI**
   - Language: **Swift**
   - Storage: **None**
4. 选择保存位置，点击 Create

### 目录结构

在 Xcode 的 Project Navigator 中，右键 `SimpleWeather` 文件夹，创建以下分组（Group）：

```
SimpleWeather/
├── Models/           ← 数据模型
│   ├── WeatherResponse.swift
│   └── DailyForecast.swift
├── Services/         ← 网络请求和位置服务
│   ├── WeatherService.swift
│   └── LocationManager.swift
├── Views/            ← 页面视图
│   ├── WeatherView.swift
│   ├── CurrentWeatherView.swift
│   ├── ForecastListView.swift
│   └── SearchView.swift
├── Utilities/        ← 工具类
│   └── WeatherIcon.swift
└── SimpleWeatherApp.swift  ← App 入口
```

💡 **提示**：在 Xcode 中创建 Group 只是逻辑分组，不会在文件系统中创建真实文件夹。如果你希望文件系统也对应，可以在 Finder 中手动创建文件夹，然后在 Xcode 中 "Add Files to..." 添加。

---

## 数据模型设计

数据模型是 App 的"骨架"——它定义了天气数据的结构，让网络层知道如何解析 JSON，让 UI 层知道有哪些数据可以展示。

💡 **通俗理解**：数据模型就像快递单——上面规定了"收件人、地址、电话"等字段。API 返回的 JSON 数据就像快递包裹，数据模型告诉 Swift 怎么把包裹里的东西放到对应的字段里。

### 天气数据模型

#### WeatherResponse

对应 OpenWeatherMap Current Weather API 的返回数据：

```swift
import Foundation

struct WeatherResponse: Codable {
    let name: String
    let coord: Coord
    let weather: [Weather]
    let main: Main
    let wind: Wind
    let sys: Sys
    let dt: TimeInterval
}

struct Coord: Codable {
    let lat: Double
    let lon: Double
}

struct Weather: Codable {
    let id: Int
    let main: String
    let description: String
    let icon: String
}

struct Main: Codable {
    let temp: Double
    let feelsLike: Double
    let tempMin: Double
    let tempMax: Double
    let humidity: Int
    let pressure: Int

    enum CodingKeys: String, CodingKey {
        case temp
        case feelsLike = "feels_like"
        case tempMin = "temp_min"
        case tempMax = "temp_max"
        case humidity
        case pressure
    }
}

struct Wind: Codable {
    let speed: Double
    let deg: Int
    let gust: Double?

    enum CodingKeys: String, CodingKey {
        case speed
        case deg
        case gust
    }
}

struct Sys: Codable {
    let country: String
    let sunrise: TimeInterval
    let sunset: TimeInterval
}
```

#### CurrentWeather

用于 UI 展示的简化模型，将 `WeatherResponse` 转换为更易用的格式：

```swift
import Foundation

struct CurrentWeather: Identifiable {
    let id = UUID()
    let cityName: String
    let country: String
    let temperature: Double
    let feelsLike: Double
    let tempMin: Double
    let tempMax: Double
    let humidity: Int
    let pressure: Int
    let windSpeed: Double
    let description: String
    let iconCode: String
    let sunrise: Date
    let sunset: Date

    var temperatureString: String {
        String(format: "%.0f°", temperature)
    }

    var feelsLikeString: String {
        String(format: "%.0f°", feelsLike)
    }

    var windSpeedString: String {
        String(format: "%.1f m/s", windSpeed)
    }
}
```

#### DailyForecast

对应 5 Day Forecast API 的返回数据：

```swift
import Foundation

struct ForecastResponse: Codable {
    let list: [ForecastItem]
    let city: ForecastCity
}

struct ForecastCity: Codable {
    let name: String
    let country: String
}

struct ForecastItem: Codable {
    let dt: TimeInterval
    let main: ForecastMain
    let weather: [Weather]
    let dtTxt: String

    enum CodingKeys: String, CodingKey {
        case dt
        case main
        case weather
        case dtTxt = "dt_txt"
    }
}

struct ForecastMain: Codable {
    let temp: Double
    let tempMin: Double
    let tempMax: Double

    enum CodingKeys: String, CodingKey {
        case temp
        case tempMin = "temp_min"
        case tempMax = "temp_max"
    }
}

struct DailyForecast: Identifiable {
    let id = UUID()
    let date: Date
    let tempMin: Double
    let tempMax: Double
    let description: String
    let iconCode: String

    var dayOfWeek: String {
        let formatter = DateFormatter()
        formatter.locale = Locale(identifier: "zh_CN")
        formatter.dateFormat = "EEEE"
        return formatter.string(from: date)
    }

    var shortDate: String {
        let formatter = DateFormatter()
        formatter.locale = Locale(identifier: "zh_CN")
        formatter.dateFormat = "M/d"
        return formatter.string(from: date)
    }
}
```

### JSON 解析

#### 与 API 返回数据对应

OpenWeatherMap Current Weather API 返回的 JSON 结构大致如下：

```json
{
    "name": "Beijing",
    "coord": { "lat": 39.91, "lon": 116.40 },
    "weather": [
        { "id": 800, "main": "Clear", "description": "晴天", "icon": "01d" }
    ],
    "main": {
        "temp": 28.5,
        "feels_like": 30.2,
        "temp_min": 25.0,
        "temp_max": 31.0,
        "humidity": 45,
        "pressure": 1013
    },
    "wind": { "speed": 3.5, "deg": 180, "gust": 5.2 },
    "sys": { "country": "CN", "sunrise": 1716000000, "sunset": 1716050000 },
    "dt": 1716030000
}
```

💡 **提示**：你可以用浏览器直接访问 API URL 来查看返回的 JSON，方便对照建模。

#### CodingKeys 处理

API 返回的 JSON 字段名使用 **snake_case**（如 `feels_like`），而 Swift 的属性名使用 **camelCase**（如 `feelsLike`）。`CodingKeys` 枚举就是用来做这种映射的：

| JSON 字段 | Swift 属性 | CodingKeys 映射 |
|-----------|-----------|-----------------|
| `feels_like` | `feelsLike` | `feelsLike = "feels_like"` |
| `temp_min` | `tempMin` | `tempMin = "temp_min"` |
| `temp_max` | `tempMax` | `tempMax = "temp_max"` |
| `dt_txt` | `dtTxt` | `dtTxt = "dt_txt"` |

⚠️ **警告**：如果 JSON 字段名和 Swift 属性名不一致，但没有在 `CodingKeys` 中映射，解码时会报错！这是新手最常犯的错误之一。

---

## 网络层封装

### API 请求封装

#### WeatherService 类

我们将网络请求封装到一个 `WeatherService` 类中，提供清晰的接口：

```swift
import Foundation

enum WeatherError: LocalizedError {
    case invalidURL
    case networkError(Error)
    case decodingError
    case apiError(String)
    case noData

    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "无效的请求地址"
        case .networkError(let error):
            return "网络错误：\(error.localizedDescription)"
        case .decodingError:
            return "数据解析失败"
        case .apiError(let message):
            return "API 错误：\(message)"
        case .noData:
            return "没有数据"
        }
    }
}

class WeatherService {
    private let apiKey: String
    private let baseURL = "https://api.openweathermap.org/data/2.5"

    init(apiKey: String) {
        self.apiKey = apiKey
    }

    func fetchCurrentWeather(lat: Double, lon: Double) async throws -> WeatherResponse {
        let urlString = "\(baseURL)/weather?lat=\(lat)&lon=\(lon)&appid=\(apiKey)&units=metric&lang=zh_cn"
        return try await request(urlString: urlString)
    }

    func fetchCurrentWeather(city: String) async throws -> WeatherResponse {
        let encodedCity = city.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? city
        let urlString = "\(baseURL)/weather?q=\(encodedCity)&appid=\(apiKey)&units=metric&lang=zh_cn"
        return try await request(urlString: urlString)
    }

    func fetchForecast(lat: Double, lon: Double) async throws -> ForecastResponse {
        let urlString = "\(baseURL)/forecast?lat=\(lat)&lon=\(lon)&appid=\(apiKey)&units=metric&lang=zh_cn"
        return try await request(urlString: urlString)
    }

    func fetchForecast(city: String) async throws -> ForecastResponse {
        let encodedCity = city.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? city
        let urlString = "\(baseURL)/forecast?q=\(encodedCity)&appid=\(apiKey)&units=metric&lang=zh_cn"
        return try await request(urlString: urlString)
    }

    private func request<T: Decodable>(urlString: String) async throws -> T {
        guard let url = URL(string: urlString) else {
            throw WeatherError.invalidURL
        }

        let (data, response) = try await URLSession.shared.data(from: url)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw WeatherError.networkError(URLError(.badServerResponse))
        }

        guard httpResponse.statusCode == 200 else {
            if let errorBody = try? JSONDecoder().decode(APIError.self, from: data) {
                throw WeatherError.apiError(errorBody.message)
            }
            throw WeatherError.apiError("HTTP \(httpResponse.statusCode)")
        }

        do {
            return try JSONDecoder().decode(T.self, from: data)
        } catch {
            throw WeatherError.decodingError
        }
    }
}

private struct APIError: Decodable {
    let message: String
}
```

💡 **通俗理解**：`WeatherService` 就像一个"外卖员"——你告诉他"我要北京的天气"（传参数），他去 API 那里取数据，然后把数据带回来给你。如果出了问题（网络断了、API 报错），他会告诉你具体什么问题。

#### async/await 请求

我们使用了 Swift 的 `async/await` 语法来处理异步网络请求：

| 对比 | 旧方式（闭包） | 新方式（async/await） |
|------|--------------|---------------------|
| 代码风格 | 嵌套闭包，容易"回调地狱" | 线性代码，像同步一样 |
| 错误处理 | 闭包参数传递 | try/catch |
| 可读性 | 较差 | 更好 |
| 最低系统要求 | iOS 13+ | iOS 15+ |

```swift
// 旧方式（闭包）
func fetchWeather(completion: @escaping (Result<WeatherResponse, Error>) -> Void) {
    URLSession.shared.dataTask(with: url) { data, response, error in
        // 嵌套处理...
    }.resume()
}

// 新方式（async/await）
func fetchWeather() async throws -> WeatherResponse {
    let (data, response) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(WeatherResponse.self, from: data)
}
```

### 错误处理

#### 网络错误

| 错误类型 | 说明 | 处理方式 |
|---------|------|---------|
| 无网络连接 | 设备没有联网 | 提示用户检查网络 |
| 请求超时 | 网络太慢 | 提示用户稍后重试 |
| 服务器错误 | API 服务器故障 | 提示用户稍后重试 |

#### 解码错误

| 错误类型 | 说明 | 处理方式 |
|---------|------|---------|
| 字段缺失 | API 返回的数据缺少某个字段 | 检查模型是否和 API 对应 |
| 类型不匹配 | 期望 Int 但收到 String | 检查 CodingKeys 映射 |
| JSON 格式错误 | API 返回的不是有效 JSON | 检查 API 文档 |

#### API 错误

| HTTP 状态码 | 说明 | 常见原因 |
|------------|------|---------|
| 401 | 未授权 | API Key 无效或未激活 |
| 404 | 未找到 | 城市名拼写错误 |
| 429 | 请求过多 | 超出免费额度限制 |

💡 **提示**：开发时建议在 `request` 方法中打印返回的 JSON，方便调试：

```swift
if let jsonString = String(data: data, encoding: .utf8) {
    print("API 返回: \(jsonString)")
}
```

⚠️ **警告**：上线前务必删除所有 `print` 语句，避免泄露 API 返回的敏感数据！

---

## 位置权限与 CoreLocation

### 请求位置权限

#### Info.plist 添加权限描述

iOS 要求在请求位置权限前，必须说明为什么需要位置信息。在 `Info.plist` 中添加：

| Key | Value | 说明 |
|-----|-------|------|
| `NSLocationWhenInUseUsageDescription` | "需要获取您的位置以显示当地天气" | 使用时获取位置 |
| `NSLocationAlwaysAndWhenInUseUsageDescription` | "需要获取您的位置以显示当地天气" | 始终获取位置（可选） |

在 Xcode 中操作：
1. 点击项目 → 选择 Target → Info 标签页
2. 点击 "+" 添加 `Privacy - Location When In Use Usage Description`
3. 填写描述文字

⚠️ **警告**：如果不添加权限描述，App 会在请求位置时直接崩溃！

#### CLLocationManager

#### 获取当前位置

我们使用 `CLLocationManager` 来获取用户位置，并通过 `ObservableObject` 将位置变化传递给 SwiftUI：

```swift
import Foundation
import CoreLocation

class LocationManager: NSObject, ObservableObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    @Published var location: CLLocation?
    @Published var authorizationStatus: CLAuthorizationStatus = .notDetermined
    @Published var errorMessage: String?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyKilometer
    }

    func requestLocation() {
        switch manager.authorizationStatus {
        case .notDetermined:
            manager.requestWhenInUseAuthorization()
        case .authorizedWhenInUse, .authorizedAlways:
            manager.requestLocation()
        case .denied, .restricted:
            errorMessage = "位置权限被拒绝，请在设置中开启"
        @unknown default:
            break
        }
    }

    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        authorizationStatus = manager.authorizationStatus
        if manager.authorizationStatus == .authorizedWhenInUse || manager.authorizationStatus == .authorizedAlways {
            manager.requestLocation()
        }
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        location = locations.last
        errorMessage = nil
    }

    func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
        errorMessage = "获取位置失败：\(error.localizedDescription)"
    }
}
```

💡 **提示**：`desiredAccuracy` 设置为 `kCLLocationAccuracyKilometer`（公里级精度）就够了——天气 App 不需要精确到米。精度越高越耗电。

---

## 主界面开发

### 当前天气展示

主界面分为三个区域：城市名称、当前天气、天气预报列表。

#### 城市名称

```swift
HStack {
    Image(systemName: "location.fill")
        .foregroundStyle(.red)
    Text(currentWeather.cityName)
        .font(.title2.bold())
    Text(currentWeather.country)
        .font(.subheadline)
        .foregroundStyle(.secondary)
}
```

#### 温度显示

```swift
VStack(spacing: 4) {
    Text(currentWeather.temperatureString)
        .font(.system(size: 72, weight: .thin))
    Text(currentWeather.description)
        .font(.title3)
        .foregroundStyle(.secondary)
    HStack(spacing: 16) {
        Label(currentWeather.feelsLikeString, systemImage: "thermometer")
        Label("\(currentWeather.humidity)%", systemImage: "drop.fill")
        Label(currentWeather.windSpeedString, systemImage: "wind")
    }
    .font(.subheadline)
    .foregroundStyle(.secondary)
}
```

#### 体感温度/湿度/风速

| 信息 | 图标 | 格式 | 示例 |
|------|------|------|------|
| 体感温度 | 🌡 thermometer | `XX°` | 30° |
| 湿度 | 💧 drop.fill | `XX%` | 45% |
| 风速 | 🌬 wind | `X.X m/s` | 3.5 m/s |

### 天气图标

#### SF Weather Symbols

Apple 提供了一组天气相关的 SF Symbols，可以直接使用：

| 天气状况 | SF Symbol | 说明 |
|---------|-----------|------|
| 晴天（白天） | `sun.max.fill` | 太阳 |
| 晴天（夜晚） | `moon.fill` | 月亮 |
| 多云 | `cloud.fill` | 云 |
| 晴转多云 | `cloud.sun.fill` | 太阳+云 |
| 小雨 | `cloud.rain.fill` | 云+雨 |
| 大雨 | `cloud.heavyrain.fill` | 云+大雨 |
| 雷阵雨 | `cloud.bolt.fill` | 云+闪电 |
| 雪 | `snowflake` | 雪花 |
| 雾 | `cloud.fog.fill` | 雾 |

#### 根据天气状况选择图标

```swift
import SwiftUI

enum WeatherIcon {
    static func systemName(for code: String) -> String {
        switch code {
        case "01d": return "sun.max.fill"
        case "01n": return "moon.fill"
        case "02d": return "cloud.sun.fill"
        case "02n": return "cloud.moon.fill"
        case "03d", "03n": return "cloud.fill"
        case "04d", "04n": return "cloud.fill"
        case "09d", "09n": return "cloud.rain.fill"
        case "10d": return "cloud.sun.rain.fill"
        case "10n": return "cloud.moon.rain.fill"
        case "11d", "11n": return "cloud.bolt.fill"
        case "13d", "13n": return "snowflake"
        case "50d", "50n": return "cloud.fog.fill"
        default: return "cloud.fill"
        }
    }

    static func color(for code: String) -> Color {
        switch code {
        case "01d": return .yellow
        case "01n": return .indigo
        case "02d", "02n": return .blue
        case "03d", "03n", "04d", "04n": return .gray
        case "09d", "09n", "10d", "10n": return .blue
        case "11d", "11n": return .purple
        case "13d", "13n": return .cyan
        case "50d", "50n": return .gray
        default: return .blue
        }
    }
}
```

💡 **提示**：OpenWeatherMap 返回的 `icon` 字段是一个编码，如 `01d` 表示白天晴天，`10n` 表示夜晚小雨。`d` = day（白天），`n` = night（夜晚）。

### 温度与体感温度

#### 格式化显示

```swift
extension Double {
    var temperatureString: String {
        String(format: "%.0f°", self)
    }
}
```

| 温度值 | 格式化结果 |
|--------|-----------|
| 28.5 | "29°" |
| -3.2 | "-3°" |
| 0.0 | "0°" |

---

## 动态 UI 更新

### 下拉刷新

#### .refreshable

SwiftUI 提供了 `.refreshable` 修饰符，只需一行代码就能实现下拉刷新：

```swift
ScrollView {
    // 天气内容
}
.refreshable {
    await loadWeather()
}
```

💡 **通俗理解**：`.refreshable` 就像微信朋友圈的"下拉刷新"——用户往下拉，出现加载指示器，数据加载完成后自动消失。

### 加载状态

#### Loading/Loaded/Error 状态管理

我们定义一个枚举来管理三种状态：

```swift
enum LoadState<T> {
    case loading
    case loaded(T)
    case error(String)
}
```

| 状态 | UI 表现 | 用户操作 |
|------|---------|---------|
| `.loading` | 显示加载动画 | 等待 |
| `.loaded(data)` | 显示天气数据 | 正常浏览 |
| `.error(message)` | 显示错误信息 | 点击重试 |

### 错误状态

#### 错误提示

```swift
case .error(let message):
    VStack(spacing: 16) {
        Image(systemName: "exclamationmark.triangle.fill")
            .font(.system(size: 48))
            .foregroundStyle(.orange)
        Text("加载失败")
            .font(.title3.bold())
        Text(message)
            .font(.subheadline)
            .foregroundStyle(.secondary)
            .multilineTextAlignment(.center)
        Button("重试") {
            Task { await loadWeather() }
        }
        .buttonStyle(.borderedProminent)
    }
    .padding()
```

#### 重试按钮

重试按钮调用 `loadWeather()` 方法重新加载数据：

```swift
Button("重试") {
    Task {
        await loadWeather()
    }
}
.buttonStyle(.borderedProminent)
```

⚠️ **警告**：在 SwiftUI 中调用 `async` 函数，必须使用 `Task { await ... }` 包裹。直接调用会导致编译错误。

---

## 用 AI 辅助完成开发

当你遇到问题时，可以用 AI 来辅助开发。以下是几个实用的 Prompt 模板：

### Prompt 示例

**场景 1：数据模型不确定**

```
我正在使用 OpenWeatherMap API 开发天气 App。
API 返回的 JSON 如下：
{粘贴 JSON 数据}

请帮我编写对应的 Swift Codable 数据模型，
注意处理 snake_case 到 camelCase 的映射。
```

**场景 2：网络请求出错**

```
我的 SwiftUI 天气 App 在请求 OpenWeatherMap API 时报错：
{粘贴错误信息}

我的请求代码如下：
{粘贴代码}

请帮我分析原因并修复。
```

**场景 3：UI 布局问题**

```
我想在 SwiftUI 中实现这样的天气 App 界面：
- 顶部显示城市名和定位图标
- 中间显示大号温度和天气图标
- 底部显示 5 天天气预报列表

请帮我编写完整的 SwiftUI 代码，要求美观、支持深色模式。
```

**场景 4：功能完善**

```
我的天气 App 已经实现了基本功能，现在想添加以下特性：
1. 搜索城市
2. 下拉刷新
3. 加载状态和错误处理

请帮我逐步实现，给出完整可运行的代码。
```

💡 **提示**：使用 AI 辅助开发时，**提供尽可能多的上下文**（代码、错误信息、API 文档），AI 给出的回答会更准确。

---

## 完整代码

以下是天气 App 的完整可运行代码。将所有代码放在 `SimpleWeather` 项目中即可运行。

⚠️ **注意**：请将 `YOUR_API_KEY` 替换为你在 OpenWeatherMap 注册的真实 API Key。

### SimpleWeatherApp.swift

```swift
import SwiftUI

@main
struct SimpleWeatherApp: App {
    var body: some Scene {
        WindowGroup {
            WeatherView()
        }
    }
}
```

### Models/WeatherResponse.swift

```swift
import Foundation

struct WeatherResponse: Codable {
    let name: String
    let coord: Coord
    let weather: [WeatherInfo]
    let main: MainInfo
    let wind: WindInfo
    let sys: SysInfo
    let dt: TimeInterval
}

struct Coord: Codable {
    let lat: Double
    let lon: Double
}

struct WeatherInfo: Codable {
    let id: Int
    let main: String
    let description: String
    let icon: String
}

struct MainInfo: Codable {
    let temp: Double
    let feelsLike: Double
    let tempMin: Double
    let tempMax: Double
    let humidity: Int
    let pressure: Int

    enum CodingKeys: String, CodingKey {
        case temp
        case feelsLike = "feels_like"
        case tempMin = "temp_min"
        case tempMax = "temp_max"
        case humidity
        case pressure
    }
}

struct WindInfo: Codable {
    let speed: Double
    let deg: Int
    let gust: Double?
}

struct SysInfo: Codable {
    let country: String
    let sunrise: TimeInterval
    let sunset: TimeInterval
}
```

### Models/ForecastResponse.swift

```swift
import Foundation

struct ForecastResponse: Codable {
    let list: [ForecastItem]
    let city: ForecastCity
}

struct ForecastCity: Codable {
    let name: String
    let country: String
}

struct ForecastItem: Codable {
    let dt: TimeInterval
    let main: ForecastMainInfo
    let weather: [WeatherInfo]
    let dtTxt: String

    enum CodingKeys: String, CodingKey {
        case dt, main, weather
        case dtTxt = "dt_txt"
    }
}

struct ForecastMainInfo: Codable {
    let temp: Double
    let tempMin: Double
    let tempMax: Double

    enum CodingKeys: String, CodingKey {
        case temp
        case tempMin = "temp_min"
        case tempMax = "temp_max"
    }
}
```

### Models/CurrentWeather.swift

```swift
import Foundation

struct CurrentWeather: Identifiable {
    let id = UUID()
    let cityName: String
    let country: String
    let temperature: Double
    let feelsLike: Double
    let tempMin: Double
    let tempMax: Double
    let humidity: Int
    let pressure: Int
    let windSpeed: Double
    let description: String
    let iconCode: String
    let sunrise: Date
    let sunset: Date

    var temperatureString: String {
        String(format: "%.0f°", temperature)
    }

    var feelsLikeString: String {
        String(format: "%.0f°", feelsLike)
    }

    var windSpeedString: String {
        String(format: "%.1f m/s", windSpeed)
    }

    var tempRangeString: String {
        String(format: "%.0f° / %.0f°", tempMax, tempMin)
    }
}

extension CurrentWeather {
    init(from response: WeatherResponse) {
        self.cityName = response.name
        self.country = response.sys.country
        self.temperature = response.main.temp
        self.feelsLike = response.main.feelsLike
        self.tempMin = response.main.tempMin
        self.tempMax = response.main.tempMax
        self.humidity = response.main.humidity
        self.pressure = response.main.pressure
        self.windSpeed = response.wind.speed
        self.description = response.weather.first?.description ?? ""
        self.iconCode = response.weather.first?.icon ?? "01d"
        self.sunrise = Date(timeIntervalSince1970: response.sys.sunrise)
        self.sunset = Date(timeIntervalSince1970: response.sys.sunset)
    }
}
```

### Models/DailyForecast.swift

```swift
import Foundation

struct DailyForecast: Identifiable {
    let id = UUID()
    let date: Date
    let tempMin: Double
    let tempMax: Double
    let description: String
    let iconCode: String

    var dayOfWeek: String {
        let formatter = DateFormatter()
        formatter.locale = Locale(identifier: "zh_CN")
        formatter.dateFormat = "EEEE"
        return formatter.string(from: date)
    }

    var shortDate: String {
        let formatter = DateFormatter()
        formatter.locale = Locale(identifier: "zh_CN")
        formatter.dateFormat = "M/d"
        return formatter.string(from: date)
    }

    var tempMinString: String {
        String(format: "%.0f°", tempMin)
    }

    var tempMaxString: String {
        String(format: "%.0f°", tempMax)
    }
}

extension DailyForecast {
    static func from(forecast: ForecastResponse) -> [DailyForecast] {
        let calendar = Calendar.current
        var dailyMap: [String: (items: [ForecastItem], date: Date)] = [:]

        for item in forecast.list {
            let date = Date(timeIntervalSince1970: item.dt)
            let key = calendar.startOfDay(for: date).description

            if dailyMap[key] == nil {
                dailyMap[key] = (items: [], date: calendar.startOfDay(for: date))
            }
            dailyMap[key]?.items.append(item)
        }

        let today = calendar.startOfDay(for: Date())

        return dailyMap.values
            .filter { $0.date > today }
            .sorted { $0.date < $1.date }
            .prefix(5)
            .map { entry in
                let temps = entry.items.map { $0.main.temp }
                let minTemp = temps.min() ?? 0
                let maxTemp = temps.max() ?? 0
                let midItem = entry.items[entry.items.count / 2]

                return DailyForecast(
                    date: entry.date,
                    tempMin: minTemp,
                    tempMax: maxTemp,
                    description: midItem.weather.first?.description ?? "",
                    iconCode: midItem.weather.first?.icon ?? "01d"
                )
            }
    }
}
```

### Services/WeatherService.swift

```swift
import Foundation

enum WeatherError: LocalizedError {
    case invalidURL
    case networkError(Error)
    case decodingError
    case apiError(String)
    case noData

    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "无效的请求地址"
        case .networkError(let error):
            return "网络错误：\(error.localizedDescription)"
        case .decodingError:
            return "数据解析失败"
        case .apiError(let message):
            return message
        case .noData:
            return "没有数据"
        }
    }
}

struct APIErrorMessage: Decodable {
    let message: String
}

class WeatherService {
    private let apiKey: String
    private let baseURL = "https://api.openweathermap.org/data/2.5"

    init(apiKey: String) {
        self.apiKey = apiKey
    }

    func fetchCurrentWeather(lat: Double, lon: Double) async throws -> WeatherResponse {
        let urlString = "\(baseURL)/weather?lat=\(lat)&lon=\(lon)&appid=\(apiKey)&units=metric&lang=zh_cn"
        return try await request(urlString: urlString)
    }

    func fetchCurrentWeather(city: String) async throws -> WeatherResponse {
        let encodedCity = city.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? city
        let urlString = "\(baseURL)/weather?q=\(encodedCity)&appid=\(apiKey)&units=metric&lang=zh_cn"
        return try await request(urlString: urlString)
    }

    func fetchForecast(lat: Double, lon: Double) async throws -> ForecastResponse {
        let urlString = "\(baseURL)/forecast?lat=\(lat)&lon=\(lon)&appid=\(apiKey)&units=metric&lang=zh_cn"
        return try await request(urlString: urlString)
    }

    func fetchForecast(city: String) async throws -> ForecastResponse {
        let encodedCity = city.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? city
        let urlString = "\(baseURL)/forecast?q=\(encodedCity)&appid=\(apiKey)&units=metric&lang=zh_cn"
        return try await request(urlString: urlString)
    }

    private func request<T: Decodable>(urlString: String) async throws -> T {
        guard let url = URL(string: urlString) else {
            throw WeatherError.invalidURL
        }

        let (data, response) = try await URLSession.shared.data(from: url)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw WeatherError.networkError(URLError(.badServerResponse))
        }

        guard httpResponse.statusCode == 200 else {
            if let errorBody = try? JSONDecoder().decode(APIErrorMessage.self, from: data) {
                throw WeatherError.apiError(errorBody.message)
            }
            throw WeatherError.apiError("服务器错误 (\(httpResponse.statusCode))")
        }

        do {
            return try JSONDecoder().decode(T.self, from: data)
        } catch {
            throw WeatherError.decodingError
        }
    }
}
```

### Services/LocationManager.swift

```swift
import Foundation
import CoreLocation

class LocationManager: NSObject, ObservableObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    @Published var location: CLLocation?
    @Published var errorMessage: String?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyKilometer
    }

    func requestLocation() {
        switch manager.authorizationStatus {
        case .notDetermined:
            manager.requestWhenInUseAuthorization()
        case .authorizedWhenInUse, .authorizedAlways:
            manager.requestLocation()
        case .denied, .restricted:
            errorMessage = "位置权限被拒绝，请在设置中开启"
        @unknown default:
            break
        }
    }

    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        if manager.authorizationStatus == .authorizedWhenInUse || manager.authorizationStatus == .authorizedAlways {
            manager.requestLocation()
        } else if manager.authorizationStatus == .denied {
            errorMessage = "位置权限被拒绝，请在设置中开启"
        }
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        location = locations.last
        errorMessage = nil
    }

    func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
        errorMessage = "获取位置失败：\(error.localizedDescription)"
    }
}
```

### Utilities/WeatherIcon.swift

```swift
import SwiftUI

enum WeatherIcon {
    static func systemName(for code: String) -> String {
        switch code {
        case "01d": return "sun.max.fill"
        case "01n": return "moon.fill"
        case "02d": return "cloud.sun.fill"
        case "02n": return "cloud.moon.fill"
        case "03d", "03n": return "cloud.fill"
        case "04d", "04n": return "cloud.fill"
        case "09d", "09n": return "cloud.rain.fill"
        case "10d": return "cloud.sun.rain.fill"
        case "10n": return "cloud.moon.rain.fill"
        case "11d", "11n": return "cloud.bolt.fill"
        case "13d", "13n": return "snowflake"
        case "50d", "50n": return "cloud.fog.fill"
        default: return "cloud.fill"
        }
    }

    static func color(for code: String) -> Color {
        switch code {
        case "01d": return .yellow
        case "01n": return .indigo
        case "02d": return .blue
        case "02n": return .indigo
        case "03d", "03n", "04d", "04n": return .gray
        case "09d", "09n", "10d", "10n": return .blue
        case "11d", "11n": return .purple
        case "13d", "13n": return .cyan
        case "50d", "50n": return .gray
        default: return .blue
        }
    }

    static func backgroundGradient(for code: String) -> [Color] {
        switch code {
        case "01d": return [.orange, .yellow]
        case "01n": return [.indigo, .purple]
        case "02d": return [.blue, .cyan]
        case "02n": return [.indigo, .blue]
        case "03d", "03n", "04d", "04n": return [.gray, .blue.opacity(0.5)]
        case "09d", "09n", "10d", "10n": return [.blue, .teal]
        case "11d", "11n": return [.purple, .gray]
        case "13d", "13n": return [.cyan, .white.opacity(0.5)]
        case "50d", "50n": return [.gray, .gray.opacity(0.5)]
        default: return [.blue, .purple]
        }
    }
}
```

### Views/WeatherView.swift

```swift
import SwiftUI

enum LoadState<T> {
    case loading
    case loaded(T)
    case error(String)
}

struct WeatherView: View {
    @StateObject private var locationManager = LocationManager()
    @State private var currentWeatherState: LoadState<CurrentWeather> = .loading
    @State private var forecastState: LoadState<[DailyForecast]> = .loading
    @State private var showSearch = false
    @State private var searchCity = ""

    private let weatherService = WeatherService(apiKey: "YOUR_API_KEY")

    var body: some View {
        NavigationStack {
            ZStack {
                backgroundGradient
                    .ignoresSafeArea()

                switch (currentWeatherState, forecastState) {
                case (.loading, _):
                    loadingView
                case (.error(let message), _):
                    errorView(message: message)
                case (.loaded(let weather), .loaded(let forecasts)):
                    weatherContent(weather: weather, forecasts: forecasts)
                case (.loaded(let weather), .loading):
                    weatherContent(weather: weather, forecasts: [])
                case (.loaded(let weather), .error):
                    weatherContent(weather: weather, forecasts: [])
                default:
                    loadingView
                }
            }
            .navigationTitle("")
            .toolbar {
                ToolbarItem(placement: .topBarTrailing) {
                    Button {
                        showSearch = true
                    } label: {
                        Image(systemName: "magnifyingglass")
                            .foregroundStyle(.white)
                    }
                }
            }
            .sheet(isPresented: $showSearch) {
                SearchView(weatherService: weatherService) { lat, lon in
                    Task {
                        await loadWeatherByLocation(lat: lat, lon: lon)
                    }
                }
            }
            .task {
                locationManager.requestLocation()
            }
            .onChange(of: locationManager.location) { _, newLocation in
                if let location = newLocation {
                    Task {
                        await loadWeatherByLocation(
                            lat: location.coordinate.latitude,
                            lon: location.coordinate.longitude
                        )
                    }
                }
            }
            .onChange(of: locationManager.errorMessage) { _, message in
                if let message = message {
                    currentWeatherState = .error(message)
                }
            }
        }
    }

    private var backgroundGradient: LinearGradient {
        let colors: [Color]
        if case .loaded(let weather) = currentWeatherState {
            colors = WeatherIcon.backgroundGradient(for: weather.iconCode)
        } else {
            colors = [.blue, .purple]
        }
        return LinearGradient(colors: colors, startPoint: .topLeading, endPoint: .bottomTrailing)
    }

    private var loadingView: some View {
        VStack(spacing: 20) {
            ProgressView()
                .scaleEffect(1.5)
                .tint(.white)
            Text("正在获取天气数据...")
                .foregroundStyle(.white.opacity(0.8))
        }
    }

    private func errorView(message: String) -> some View {
        VStack(spacing: 16) {
            Image(systemName: "exclamationmark.triangle.fill")
                .font(.system(size: 48))
                .foregroundStyle(.white.opacity(0.8))
            Text("加载失败")
                .font(.title3.bold())
                .foregroundStyle(.white)
            Text(message)
                .font(.subheadline)
                .foregroundStyle(.white.opacity(0.7))
                .multilineTextAlignment(.center)
                .padding(.horizontal)
            Button("重试") {
                locationManager.requestLocation()
                currentWeatherState = .loading
                forecastState = .loading
            }
            .buttonStyle(.borderedProminent)
            .tint(.white.opacity(0.3))
        }
    }

    private func weatherContent(weather: CurrentWeather, forecasts: [DailyForecast]) -> some View {
        ScrollView {
            VStack(spacing: 24) {
                currentWeatherSection(weather: weather)

                if !forecasts.isEmpty {
                    forecastSection(forecasts: forecasts)
                }

                detailSection(weather: weather)
            }
            .padding()
        }
        .refreshable {
            if let location = locationManager.location {
                await loadWeatherByLocation(
                    lat: location.coordinate.latitude,
                    lon: location.coordinate.longitude
                )
            }
        }
    }

    private func currentWeatherSection(weather: CurrentWeather) -> some View {
        VStack(spacing: 8) {
            HStack {
                Image(systemName: "location.fill")
                    .font(.caption)
                Text(weather.cityName)
                    .font(.title2.bold())
                Text(weather.country)
                    .font(.subheadline)
                    .foregroundStyle(.white.opacity(0.7))
            }
            .foregroundStyle(.white)

            Image(systemName: WeatherIcon.systemName(for: weather.iconCode))
                .font(.system(size: 80))
                .symbolRenderingMode(.multicolor)
                .padding(.vertical, 4)

            Text(weather.temperatureString)
                .font(.system(size: 72, weight: .thin))
                .foregroundStyle(.white)

            Text(weather.description)
                .font(.title3)
                .foregroundStyle(.white.opacity(0.9))

            Text(weather.tempRangeString)
                .font(.subheadline)
                .foregroundStyle(.white.opacity(0.7))
        }
    }

    private func forecastSection(forecasts: [DailyForecast]) -> some View {
        VStack(alignment: .leading, spacing: 12) {
            Text("未来天气预报")
                .font(.headline)
                .foregroundStyle(.white)

            VStack(spacing: 0) {
                ForEach(forecasts) { forecast in
                    HStack {
                        Text(forecast.dayOfWeek)
                            .font(.subheadline)
                            .frame(width: 60, alignment: .leading)

                        Image(systemName: WeatherIcon.systemName(for: forecast.iconCode))
                            .foregroundStyle(WeatherIcon.color(for: forecast.iconCode))
                            .frame(width: 30)

                        Text(forecast.description)
                            .font(.caption)
                            .foregroundStyle(.white.opacity(0.7))
                            .frame(maxWidth: .infinity, alignment: .leading)

                        Text(forecast.tempMinString)
                            .font(.subheadline)
                            .foregroundStyle(.white.opacity(0.6))

                        Text(forecast.tempMaxString)
                            .font(.subheadline.bold())
                            .foregroundStyle(.white)
                    }
                    .padding(.vertical, 8)

                    if forecast.id != forecasts.last?.id {
                        Divider().background(.white.opacity(0.2))
                    }
                }
            }
            .padding()
            .background(.ultraThinMaterial, in: RoundedRectangle(cornerRadius: 16))
        }
    }

    private func detailSection(weather: CurrentWeather) -> some View {
        VStack(spacing: 12) {
            HStack(spacing: 12) {
                DetailItem(icon: "thermometer", title: "体感温度", value: weather.feelsLikeString)
                DetailItem(icon: "drop.fill", title: "湿度", value: "\(weather.humidity)%")
            }

            HStack(spacing: 12) {
                DetailItem(icon: "wind", title: "风速", value: weather.windSpeedString)
                DetailItem(icon: "gauge", title: "气压", value: "\(weather.pressure) hPa")
            }
        }
    }

    private func loadWeatherByLocation(lat: Double, lon: Double) async {
        do {
            async let currentTask = weatherService.fetchCurrentWeather(lat: lat, lon: lon)
            async let forecastTask = weatherService.fetchForecast(lat: lat, lon: lon)

            let currentResponse = try await currentTask
            let forecastResponse = try await forecastTask

            withAnimation(.spring(duration: 0.4, bounce: 0.2)) {
                currentWeatherState = .loaded(CurrentWeather(from: currentResponse))
                forecastState = .loaded(DailyForecast.from(forecast: forecastResponse))
            }
        } catch let error as WeatherError {
            currentWeatherState = .error(error.errorDescription ?? "未知错误")
        } catch {
            currentWeatherState = .error(error.localizedDescription)
        }
    }
}

struct DetailItem: View {
    let icon: String
    let title: String
    let value: String

    var body: some View {
        VStack(spacing: 8) {
            Image(systemName: icon)
                .font(.title2)
                .foregroundStyle(.white.opacity(0.8))
            Text(title)
                .font(.caption)
                .foregroundStyle(.white.opacity(0.6))
            Text(value)
                .font(.title3.bold())
                .foregroundStyle(.white)
        }
        .frame(maxWidth: .infinity)
        .padding()
        .background(.ultraThinMaterial, in: RoundedRectangle(cornerRadius: 12))
    }
}
```

### Views/SearchView.swift

```swift
import SwiftUI

struct SearchView: View {
    let weatherService: WeatherService
    let onSelect: (Double, Double) -> Void

    @Environment(\.dismiss) private var dismiss
    @State private var searchText = ""
    @State private var isSearching = false
    @State private var errorMessage: String?

    var body: some View {
        NavigationStack {
            VStack(spacing: 20) {
                searchField

                if isSearching {
                    ProgressView()
                        .padding()
                }

                if let errorMessage = errorMessage {
                    VStack(spacing: 12) {
                        Image(systemName: "exclamationmark.triangle.fill")
                            .font(.title2)
                            .foregroundStyle(.orange)
                        Text(errorMessage)
                            .font(.subheadline)
                            .foregroundStyle(.secondary)
                            .multilineTextAlignment(.center)
                    }
                    .padding()
                }

                Spacer()

                Text("输入城市名称搜索天气\n例如：Beijing、Shanghai、Tokyo")
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
                    .multilineTextAlignment(.center)
                    .padding()
            }
            .navigationTitle("搜索城市")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button("取消") {
                        dismiss()
                    }
                }
            }
        }
    }

    private var searchField: some View {
        HStack {
            Image(systemName: "magnifyingglass")
                .foregroundStyle(.secondary)

            TextField("输入城市名称", text: $searchText)
                .textFieldStyle(.plain)
                .onSubmit {
                    searchCity()
                }

            if !searchText.isEmpty {
                Button {
                    searchText = ""
                    errorMessage = nil
                } label: {
                    Image(systemName: "xmark.circle.fill")
                        .foregroundStyle(.secondary)
                }
            }

            Button("搜索") {
                searchCity()
            }
            .buttonStyle(.borderedProminent)
            .disabled(searchText.isEmpty || isSearching)
        }
        .padding()
        .background(Color.gray.opacity(0.1))
        .clipShape(RoundedRectangle(cornerRadius: 12))
        .padding(.horizontal)
    }

    private func searchCity() {
        guard !searchText.isEmpty else { return }

        isSearching = true
        errorMessage = nil

        Task {
            do {
                let response = try await weatherService.fetchCurrentWeather(city: searchText)
                onSelect(response.coord.lat, response.coord.lon)
                dismiss()
            } catch let error as WeatherError {
                isSearching = false
                errorMessage = error.errorDescription ?? "搜索失败"
            } catch {
                isSearching = false
                errorMessage = error.localizedDescription
            }
        }
    }
}
```

---

## 小结

本章我们完成了一个完整的天气 App，涉及的知识点非常丰富：

| 知识点 | 本章应用 |
|--------|---------|
| **数据模型设计** | WeatherResponse、CurrentWeather、DailyForecast |
| **JSON 解析** | Codable、CodingKeys 映射 |
| **网络请求** | async/await、URLSession、错误处理 |
| **CoreLocation** | CLLocationManager、位置权限 |
| **状态管理** | LoadState 枚举、@StateObject、@Published |
| **SwiftUI 布局** | ScrollView、VStack/HStack、Grid |
| **SF Symbols** | 天气图标映射、multicolor 渲染 |
| **下拉刷新** | .refreshable |
| **搜索功能** | .sheet、TextField、异步搜索 |
| **Spec 驱动开发** | PRD、任务拆解 |

🔑 **核心记忆点**：
1. 数据模型要和 API 返回的 JSON 严格对应，注意 CodingKeys 映射
2. 网络请求用 async/await，比闭包更清晰
3. 位置权限必须在 Info.plist 中添加描述，否则会崩溃
4. 用 LoadState 枚举管理加载/成功/错误三种状态
5. `async let` 可以并发请求多个 API，加快加载速度
6. 开发前先写 PRD 和拆解任务，避免"想到哪做到哪"

🚀 **进阶挑战**：
- 添加天气背景动画（如下雨粒子效果）
- 支持多城市收藏
- 添加小时级天气预报
- 使用 Core Data 缓存天气数据，离线也能查看
- 添加 Widget 小组件显示天气

📖 **上一章**：[26-动画与手势](./26-动画与手势.md)

← [-网络请求：获取真实数据](./48-网络请求.md) | [-推送通知](./50-推送通知.md) →
