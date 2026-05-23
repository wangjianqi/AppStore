---
name: location-maps
description: 涉及地图、定位、CoreLocation、MapKit、CLLocationManager、地理围栏、后台定位、Significant Location Changes、iBeacon 的任务
---

# 地图与位置服务

## CoreLocation

### CLLocationManager 配置

```swift
import CoreLocation

final class LocationManager: NSObject, CLLocationManagerDelegate {
    static let shared = LocationManager()

    private let manager = CLLocationManager()
    private var completion: ((Result<CLLocation, Error>) -> Void)?

    private override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyHundredMeters
        manager.distanceFilter = 100
    }

    func requestLocation() async throws -> CLLocation {
        try await withCheckedThrowingContinuation { continuation in
            self.completion = { result in
                switch result {
                case .success(let location):
                    continuation.resume(returning: location)
                case .failure(let error):
                    continuation.resume(throwing: error)
                }
            }
            manager.requestLocation()
        }
    }

    func startUpdating() {
        manager.startUpdatingLocation()
    }

    func stopUpdating() {
        manager.stopUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        guard let location = locations.last else { return }
        completion?(.success(location))
        completion = nil
    }

    func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
        completion?(.failure(error))
        completion = nil
    }
}
```

### 精度选择

| 精度 | 常量 | 耗电 | 适用场景 |
|------|------|------|---------|
| 最佳 | `kCLLocationAccuracyBestForNavigation` | 极高 | 导航 |
| 高 | `kCLLocationAccuracyBest` | 高 | 精确定位 |
| 近 | `kCLLocationAccuracyNearestTenMeters` | 中 | 附近搜索 |
| 百米 | `kCLLocationAccuracyHundredMeters` | 低 | 天气、城市级 |
| 公里 | `kCLLocationAccuracyKilometer` | 极低 | 省级定位 |
| 三公里 | `kCLLocationAccuracyThreeKilometers` | 最低 | 国家级 |

**原则：用能满足需求的最低精度**

### 权限处理

```swift
func requestLocationPermission() {
    let status = manager.authorizationStatus
    switch status {
    case .notDetermined:
        manager.requestWhenInUseAuthorization()
    case .authorizedWhenInUse:
        if requiresAlways {
            manager.requestAlwaysAuthorization()
        } else {
            startUpdating()
        }
    case .authorizedAlways:
        startUpdating()
    case .denied, .restricted:
        showLocationDeniedAlert()
    @unknown default:
        break
    }
}
```

### Info.plist 权限描述

| Key | 用途 | 何时需要 |
|-----|------|---------|
| `NSLocationWhenInUseUsageDescription` | 前台定位 | 必填 |
| `NSLocationAlwaysAndWhenInUseUsageDescription` | 后台定位 | 需要后台定位时 |
| `NSLocationAlwaysUsageDescription` | 后台定位（iOS 10 及以下） | 兼容旧系统 |

**描述文案必须具体**：如"需要获取您的位置以推荐附近商家"，禁止写"需要定位权限"

### 后台定位

#### 方式一：Background Modes → Location updates
- Info.plist 勾选 `location` 后台模式
- 设置 `manager.allowsBackgroundLocationUpdates = true`
- 设置 `manager.pausesLocationUpdatesAutomatically = true`（自动暂停省电）
- **审核要求：必须提供用户可感知的后台定位功能**（导航、轨迹记录等），否则会被拒

#### 方式二：Significant Location Changes
- `manager.startMonitoringSignificantLocationChanges()`
- 基站/WiFi 变化时触发（约 500m+ 变化）
- 极低耗电，不需要 Background Modes
- App 被杀后也能唤醒（系统级唤醒）

#### 方式三：Visits
- `manager.startMonitoringVisits()`
- 检测用户到达/离开某地点
- 极低耗电，适合"你去过这里"类功能

---

## 地理围栏

### 创建围栏

```swift
func startMonitoring(region: CLRegion) {
    manager.startMonitoring(for: region)
}

func createCircularRegion(center: CLLocationCoordinate2D, radius: CLLocationDistance, identifier: String) -> CLCircularRegion {
    let region = CLCircularRegion(center: center, radius: radius, identifier: identifier)
    region.notifyOnEntry = true
    region.notifyOnExit = true
    return region
}
```

### 监听回调

```swift
func locationManager(_ manager: CLLocationManager, didEnterRegion region: CLRegion) {
    if let circularRegion = region as? CLCircularRegion {
        sendLocalNotification(title: "到达目的地", body: "您已进入 \(circularRegion.identifier) 区域")
    }
}

func locationManager(_ manager: CLLocationManager, didExitRegion region: CLRegion) {
    // 离开区域
}
```

### 规范
- 同时最多监听 20 个围栏
- 围栏半径最小 1-10 米（室内定位），室外建议 100 米以上
- 围栏回调可能有 1-3 分钟延迟，**禁止用于精确触发**
- 模拟器测试围栏：Xcode → Debug → Simulate Location → Custom Location

---

## iBeacon

### 监听 Beacon

```swift
func startMonitoringBeacon(uuid: UUID, major: CLBeaconMajorValue, minor: CLBeaconMinorValue) {
    let constraint = CLBeaconIdentityConstraint(uuid: uuid, major: major, minor: minor)
    let region = CLBeaconRegion(beaconIdentityConstraint: constraint, identifier: uuid.uuidString)
    manager.startMonitoring(for: region)
    manager.startRangingBeacons(satisfying: constraint)
}

func locationManager(_ manager: CLLocationManager, didRange beacons: [CLBeacon], satisfying beaconConstraint: CLBeaconIdentityConstraint) {
    for beacon in beacons {
        switch beacon.proximity {
        case .immediate:   // < 0.5m
            break
        case .near:        // 0.5 - 2m
            break
        case .far:         // 2 - 5m+
            break
        case .unknown:
            break
        @unknown default:
            break
        }
    }
}
```

### 规范
- iBeacon 需要蓝牙权限：`NSBluetoothAlwaysUsageDescription`
- `startRangingBeacons` 只在前台有效，后台只能 `startMonitoring`
- Beacon 的 proximity 不精确，**禁止用于精确测距**
- iOS 13+ 使用 `CLBeaconIdentityConstraint` 替代旧的 `CLBeaconRegion` 初始化

---

## MapKit

### 基础地图

```swift
import MapKit

final class MapViewController: UIViewController {
    private let mapView = MKMapView()

    override func viewDidLoad() {
        super.viewDidLoad()
        setupMapView()
    }

    private func setupMapView() {
        mapView.delegate = self
        mapView.showsUserLocation = true
        mapView.userTrackingMode = .follow

        let region = MKCoordinateRegion(
            center: CLLocationCoordinate2D(latitude: 39.9042, longitude: 116.4074),
            span: MKCoordinateSpan(latitudeDelta: 0.01, longitudeDelta: 0.01)
        )
        mapView.setRegion(region, animated: true)
    }
}
```

### 添加标注

```swift
class PlaceAnnotation: NSObject, MKAnnotation {
    let coordinate: CLLocationCoordinate2D
    let title: String?
    let subtitle: String?
    let placeId: String

    init(coordinate: CLLocationCoordinate2D, title: String, subtitle: String?, placeId: String) {
        self.coordinate = coordinate
        self.title = title
        self.subtitle = subtitle
        self.placeId = placeId
    }
}

func addAnnotation(coordinate: CLLocationCoordinate2D, title: String) {
    let annotation = PlaceAnnotation(coordinate: coordinate, title: title, subtitle: nil, placeId: UUID().uuidString)
    mapView.addAnnotation(annotation)
}
```

### 自定义标注视图

```swift
func mapView(_ mapView: MKMapView, viewFor annotation: MKAnnotation) -> MKAnnotationView? {
    guard !annotation.isKind(of: MKUserLocation.self) else { return nil }

    let identifier = "PlaceAnnotation"
    var view = mapView.dequeueReusableAnnotationView(withIdentifier: identifier) as? MKMarkerAnnotationView
    if view == nil {
        view = MKMarkerAnnotationView(annotation: annotation, reuseIdentifier: identifier)
        view?.canShowCallout = true
        view?.rightCalloutAccessoryView = UIButton(type: .detailDisclosure)
    } else {
        view?.annotation = annotation
    }
    return view
}
```

### 地理编码

```swift
func geocode(address: String) async throws -> CLLocationCoordinate2D {
    let geocoder = CLGeocoder()
    let placemarks = try await geocoder.geocodeAddressString(address)
    guard let location = placemarks.first?.location else {
        throw LocationError.geocodeFailed
    }
    return location.coordinate
}

func reverseGeocode(location: CLLocation) async throws -> String {
    let geocoder = CLGeocoder()
    let placemarks = try await geocoder.reverseGeocodeLocation(location)
    guard let placemark = placemarks.first else {
        throw LocationError.reverseGeocodeFailed
    }
    return [placemark.locality, placemark.subLocality, placemark.thoroughfare]
        .compactMap { $0 }
        .joined(separator: "")
}
```

### 路线规划

```swift
func requestDirections(from: MKMapItem, to: MKMapItem) async throws -> MKRoute {
    let request = MKDirections.Request()
    request.source = from
    request.destination = to
    request.transportType = .automobile

    let directions = MKDirections(request: request)
    let response = try await directions.calculate()
    guard let route = response.routes.first else {
        throw LocationError.noRoute
    }
    return route
}

func drawRoute(_ route: MKRoute) {
    mapView.addOverlay(route.polyline)
    let region = MKCoordinateRegion(route.polyline.boundingMapRect)
    mapView.setRegion(region, animated: true)
}
```

---

## 规范

### 定位规范
- **用完即停**：获取位置后立即 `stopUpdatingLocation()`，禁止持续定位不停止
- **精度按需设置**：天气类 App 用百米精度，导航类才用最佳精度
- **禁止在后台持续高精度定位**（审核必拒），后台用 Significant Location Changes
- 定位超时处理：10 秒内未获取位置，降级或提示用户

### 地图规范
- 地图默认不跟踪用户位置，用户主动触发后才开启
- 标注数量超过 100 个时使用聚合（`MKClusterAnnotation`）
- 自定义地图样式使用 `MKMapView` 的 `overrideUserInterfaceStyle`
- 中国地图注意：MapKit 在中国自动使用高德地图数据，坐标系为 GCJ-02

### 审核注意
- 后台定位必须在 App 内提供明显提示（蓝条/弹窗）
- `requestAlwaysAuthorization` 必须先获得 `whenInUse` 授权
- 后台定位功能必须在 App Store Connect 的描述中说明
- **禁止仅为了广告定向而请求定位权限**

---

## 已知陷阱

- **CLLocationManager 必须持有强引用**，局部变量会被释放导致不回调
- **`requestLocation()` 只触发一次**，持续定位用 `startUpdatingLocation()`
- **模拟器定位需要手动设置**：Features → Location → Custom Location
- **GCJ-02 偏移**：中国大陆地图坐标有偏移，WGS-84 坐标直接显示会有几十到几百米偏差
- **`CLGeocoder` 有频率限制**，短时间内大量调用会返回错误
- **地理围栏在模拟器上行为不稳定**，必须真机测试
- **`MKMapView` 的 delegate 会在主线程回调**，禁止在回调中做耗时操作
- **iOS 14+ `accuracyAuthorization` 可能为 `.reducedAccuracy`**，需要检查并提示用户开启精确定位
