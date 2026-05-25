# 国内地图 SDK 集成

> 🎯 **本章目标**：了解国内地图 SDK 生态，掌握高德地图 SDK 的集成与使用，学会在 SwiftUI 中实现地图展示、定位、搜索、导航等核心功能。

---

## 国内地图生态概述

### Apple Maps 国内数据质量差

Apple Maps 在中国大陆的数据质量一直饱受诟病，主要问题包括：

- **POI 数据不完整**：大量本地商户、地标缺失或位置偏移
- **路线规划不准确**：公交换乘信息陈旧，驾车路线常绕远路
- **搜索体验差**：中文模糊搜索能力弱，无法识别简称和别名
- **地图更新滞后**：新建道路、建筑长时间不上图
- **无国内导航资质**：缺少导航电子地图资质，功能受限

这些问题导致面向国内用户的应用几乎必须接入第三方地图 SDK。

### 高德/百度/腾讯地图对比

| 特性 | 高德地图 | 百度地图 | 腾讯地图 |
|------|---------|---------|---------|
| 市场份额 | 最高（阿里系） | 较高 | 中等（微信生态） |
| POI 数据量 | 丰富 | 丰富 | 一般 |
| 路线规划 | 优秀 | 优秀 | 良好 |
| 导航功能 | 优秀 | 优秀 | 良好 |
| iOS SDK 体积 | ~50MB | ~60MB | ~40MB |
| SwiftUI 支持 | 需封装 | 需封装 | 需封装 |
| 免费配额 | 日 30,000 次 | 日 30,000 次 | 日 10,000 次 |
| 文档质量 | 优秀 | 良好 | 一般 |
| 社区活跃度 | 高 | 中 | 中 |
| 审图号 | GS(2024)xxx | GS(2024)xxx | GS(2024)xxx |

> 💡 **提示**：高德地图在 iOS 开发者中使用最广泛，文档完善、社区活跃，推荐作为首选方案。如果应用深度集成微信生态，可考虑腾讯地图。

### 为什么需要国内地图 SDK

1. **合规要求**：国内地图服务必须使用具有测绘资质的厂商数据
2. **数据质量**：本地化 POI 和路线规划远优于 Apple Maps
3. **功能完整**：提供搜索、导航、轨迹、围栏等 Apple Maps 不具备的能力
4. **用户体验**：国内用户习惯高德/百度的交互模式
5. **商业生态**：与支付宝、微信等平台的联动能力

---

## 高德地图 SDK 集成

### 开发者账号注册

1. 访问 [高德开放平台](https://lbs.amap.com/) 注册开发者账号
2. 完成实名认证（个人或企业）
3. 进入控制台创建应用
4. 添加 iOS 平台 Key，填写 Bundle Identifier
5. 获取 Key 和安全码

### SDK 安装

**CocoaPods 方式**：

```ruby
pod 'AMap3DMap'
pod 'AMapSearch'
pod 'AMapLocation'
pod 'AMapNavi'
```

**SPM 方式**：

高德官方尚未提供 SPM 支持，但可以通过 XCFramework 手动集成：

1. 从高德官网下载 SDK 压缩包
2. 将 `.xcframework` 拖入项目
3. 在 **General → Frameworks, Libraries, and Embedded Content** 中设为 **Embed & Sign**
4. 添加依赖的系统框架：`CoreLocation`、`QuartzCore`、`OpenGLES`、`SystemConfiguration`、`CoreTelephony`、`Security`

> ⚠️ **警告**：高德地图 SDK 体积较大，建议按需引入。仅展示地图只需 `AMap3DMap`，不需要导航功能则不引入 `AMapNavi`。

### Key 申请与配置

在 `AppDelegate` 或 App 入口处初始化 SDK：

```swift
import AMapFoundationKit

@main
struct MyApp: App {
    init() {
        AMapServices.shared().apiKey = "你的高德Key"
        AMapServices.shared().enableHTTPS = true
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

在 `Info.plist` 中添加隐私权限描述：

```xml
<key>NSLocationWhenInUseUsageDescription</key>
<string>用于展示您当前位置并在地图上标注</string>
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>用于持续追踪您的位置以提供导航服务</string>
```

### 基础地图展示

使用 UIKit 方式创建基础地图：

```swift
import AMap3DMap

class MapViewController: UIViewController {
    var mapView: MAMapView!

    override func viewDidLoad() {
        super.viewDidLoad()

        mapView = MAMapView(frame: view.bounds)
        mapView.delegate = self
        mapView.showsUserLocation = true
        mapView.userTrackingMode = .follow
        mapView.zoomLevel = 15
        view.addSubview(mapView)
    }

    override func viewDidLayoutSubviews() {
        super.viewDidLayoutSubviews()
        mapView.frame = view.bounds
    }
}

extension MapViewController: MAMapViewDelegate {
    func mapViewRequireLocationAuth(_ mapView: MAMapView!) {
        CLLocationManager().requestWhenInUseAuthorization()
    }
}
```

---

## SwiftUI 封装高德地图

### UIViewRepresentable 封装 MAMapView

```swift
import SwiftUI
import AMap3DMap

struct AMapViewRepresentable: UIViewRepresentable {
    @Binding var centerCoordinate: CLLocationCoordinate2D
    @Binding var zoomLevel: CGFloat
    var annotations: [MAPointAnnotation] = []

    func makeUIView(context: Context) -> MAMapView {
        let mapView = MAMapView()
        mapView.delegate = context.coordinator
        mapView.showsUserLocation = true
        mapView.userTrackingMode = .follow
        return mapView
    }

    func updateUIView(_ mapView: MAMapView, context: Context) {
        mapView.zoomLevel = zoomLevel
        mapView.setCenter(centerCoordinate, animated: true)

        mapView.removeAnnotations(mapView.annotations)
        mapView.addAnnotations(annotations)
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, MAMapViewDelegate {
        var parent: AMapViewRepresentable

        init(_ parent: AMapViewRepresentable) {
            self.parent = parent
        }

        func mapView(_ mapView: MAMapView!, regionDidChangeAnimated animated: Bool) {
            parent.centerCoordinate = mapView.centerCoordinate
            parent.zoomLevel = mapView.zoomLevel
        }

        func mapView(_ mapView: MAMapView!, viewFor annotation: MAAnnotation!) -> MAAnnotationView! {
            if annotation is MAPointAnnotation {
                let identifier = "pin"
                var pinView = mapView.dequeueReusableAnnotationView(withIdentifier: identifier) as? MAPinAnnotationView
                if pinView == nil {
                    pinView = MAPinAnnotationView(annotation: annotation, reuseIdentifier: identifier)
                }
                pinView?.canShowCallout = true
                pinView?.animatesDrop = true
                return pinView
            }
            return nil
        }

        func mapViewRequireLocationAuth(_ mapView: MAMapView!) {
            CLLocationManager().requestWhenInUseAuthorization()
        }
    }
}
```

### 地图交互（缩放/拖动/标注）

```swift
struct MapInteractionView: View {
    @State private var center = CLLocationCoordinate2D(latitude: 39.9087, longitude: 116.3975)
    @State private var zoom: CGFloat = 15
    @State private var annotations: [MAPointAnnotation] = []

    var body: some View {
        ZStack {
            AMapViewRepresentable(
                centerCoordinate: $center,
                zoomLevel: $zoom,
                annotations: annotations
            )
            .ignoresSafeArea()

            VStack {
                Spacer()
                HStack {
                    Button("放大") { zoom = min(zoom + 1, 20) }
                        .padding()
                        .background(Color.white)
                        .cornerRadius(8)
                        .shadow(radius: 2)

                    Button("缩小") { zoom = max(zoom - 1, 3) }
                        .padding()
                        .background(Color.white)
                        .cornerRadius(8)
                        .shadow(radius: 2)

                    Button("添加标注") {
                        let annotation = MAPointAnnotation()
                        annotation.coordinate = center
                        annotation.title = "标记点"
                        annotation.subtitle = "经度: \(center.longitude), 纬度: \(center.latitude)"
                        annotations.append(annotation)
                    }
                    .padding()
                    .background(Color.white)
                    .cornerRadius(8)
                    .shadow(radius: 2)
                }
                .padding()
            }
        }
    }
}
```

### 自定义标注与信息窗口

```swift
func mapView(_ mapView: MAMapView!, viewFor annotation: MAAnnotation!) -> MAAnnotationView! {
    guard annotation is MAPointAnnotation else { return nil }

    let identifier = "customPin"
    var annotationView = mapView.dequeueReusableAnnotationView(withIdentifier: identifier) as? MAAnnotationView

    if annotationView == nil {
        annotationView = MAAnnotationView(annotation: annotation, reuseIdentifier: identifier)
        annotationView?.canShowCallout = true

        let customImage = UIImage(systemName: "mappin.and.ellipse")!
        let size = CGSize(width: 40, height: 40)
        UIGraphicsBeginImageContextWithOptions(size, false, 0)
        customImage.draw(in: CGRect(origin: .zero, size: size))
        annotationView?.image = UIGraphicsGetImageFromCurrentImageContext()
        UIGraphicsEndImageContext()

        annotationView?.centerOffset = CGPoint(x: 0, y: -20)
    } else {
        annotationView?.annotation = annotation
    }

    let leftAccessory = UIImageView(frame: CGRect(x: 0, y: 0, width: 30, height: 30))
    leftAccessory.image = UIImage(systemName: "star.fill")
    leftAccessory.tintColor = .systemYellow
    annotationView?.leftCalloutAccessoryView = leftAccessory

    let rightAccessory = UIButton(type: .detailDisclosure)
    annotationView?.rightCalloutAccessoryView = rightAccessory

    return annotationView
}
```

> 💡 **提示**：自定义标注图片建议使用 `@2x` 和 `@3x` 资源，以保证在所有设备上清晰显示。图片大小控制在 40×40 点以内，避免遮挡地图内容。

---

## 定位服务

### 高德定位 SDK

高德定位 SDK 提供了比 Core Location 更精准的国内定位能力，融合了 Wi-Fi、基站和 GPS 多种定位方式：

```swift
import AMapLocationKit

class LocationManager: NSObject, ObservableObject {
    @Published var currentLocation: CLLocation?
    @Published var currentCity: String = ""
    @Published var authorizationStatus: CLAuthorizationStatus = .notDetermined

    private let locationManager = AMapLocationManager()

    override init() {
        super.init()
        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyHundredMeters
        locationManager.locatingWithReGeographic = true
        locationManager.pausesLocationUpdatesAutomatically = false
    }

    func requestLocation() {
        locationManager.requestLocation(withReGeocode: true) { [weak self] location, regeocode, error in
            if let error = error {
                print("定位失败: \(error.localizedDescription)")
                return
            }
            self?.currentLocation = location
            if let regeocode = regeocode {
                self?.currentCity = regeocode.city ?? ""
            }
        }
    }
}

extension LocationManager: AMapLocationManagerDelegate {
    func amapLocationManager(_ manager: AMapLocationManager!, didUpdate location: CLLocation!, reGeocode: AMapLocationReGeocode!) {
        currentLocation = location
        if let reGeocode = reGeocode {
            currentCity = reGeocode.city ?? ""
        }
    }

    func amapLocationManager(_ manager: AMapLocationManager!, didFailWithError error: Error!) {
        print("定位错误: \(error.localizedDescription)")
    }
}
```

### 持续定位与单次定位

| 特性 | 单次定位 | 持续定位 |
|------|---------|---------|
| 方法 | `requestLocation(withReGeocode:)` | `startUpdatingLocation()` |
| 电量消耗 | 低 | 高 |
| 适用场景 | 获取用户当前城市 | 导航、轨迹记录 |
| 精度控制 | 通过 `desiredAccuracy` | 通过 `desiredAccuracy` |
| 后台支持 | 不支持 | 需开启 Background Modes |

持续定位配置：

```swift
func startContinuousLocation() {
    locationManager.desiredAccuracy = kCLLocationAccuracyBestForNavigation
    locationManager.allowsBackgroundLocationUpdates = true
    locationManager.startUpdatingLocation()
}

func stopContinuousLocation() {
    locationManager.stopUpdatingLocation()
}
```

> ⚠️ **警告**：持续定位会显著增加电量消耗。在不需要时务必调用 `stopUpdatingLocation()` 停止定位。后台定位需要在 `Info.plist` 中启用 **Location updates** Background Mode。

### 地理编码与反地理编码

```swift
import AMapSearchKit

class GeocoderService: ObservableObject {
    private let searchAPI = AMapSearchAPI()

    func geocode(address: String, city: String = "", completion: @escaping (CLLocationCoordinate2D?) -> Void) {
        let request = AMapGeocodeSearchRequest()
        request.address = address
        request.city = city
        searchAPI.geocodeSearch(request)
    }

    func reverseGeocode(coordinate: CLLocationCoordinate2D, completion: @escaping (String?) -> Void) {
        let request = AMapReGeocodeSearchRequest()
        request.location = AMapGeoPoint(location: coordinate)
        request.requireExtension = true
        searchAPI.reGeocodeSearch(request)
    }
}
```

### SwiftUI 定位状态管理

```swift
struct LocationView: View {
    @StateObject private var locationManager = LocationManager()

    var body: some View {
        VStack(spacing: 16) {
            if let location = locationManager.currentLocation {
                Text("纬度: \(location.coordinate.latitude)")
                Text("经度: \(location.coordinate.longitude)")
                Text("城市: \(locationManager.currentCity)")
            } else {
                Text("正在获取位置...")
            }

            Button("获取当前位置") {
                locationManager.requestLocation()
            }
            .buttonStyle(.borderedProminent)
        }
        .padding()
    }
}
```

---

## 搜索与路线规划

### POI 搜索

```swift
class SearchService: NSObject, ObservableObject {
    @Published var poiResults: [AMapPOI] = []
    private let searchAPI = AMapSearchAPI()

    override init() {
        super.init()
        searchAPI.delegate = self
    }

    func searchPOI(keyword: String, city: String, around center: CLLocationCoordinate2D? = nil) {
        let request = AMapPOISearchSearchRequest()
        request.keywords = keyword
        request.city = city
        request.requireExtension = true

        if let center = center {
            request.location = AMapGeoPoint(location: center)
            request.radius = 3000
            request.sortrule = .distance
        }

        searchAPI.aMapPOISearchSearch(request)
    }
}

extension SearchService: AMapSearchDelegate {
    func onPOISearchDone(_ request: AMapPOISearchSearchRequest, response: AMapPOISearchSearchResponse) {
        poiResults = response.pois ?? []
    }

    func onSearchRequestFailed(_ request: Any!, error: Error!) {
        print("搜索失败: \(error.localizedDescription)")
    }
}
```

### 关键词搜索

```swift
func searchKeyword(_ keyword: String, in city: String) {
    let request = AMapInputTipsSearchRequest()
    request.keywords = keyword
    request.city = city
    searchAPI.aMapInputTipsSearch(request)
}

func onInputTipsSearchDone(_ request: AMapInputTipsSearchRequest, response: AMapInputTipsSearchResponse) {
    let tips = response.tips ?? []
    for tip in tips {
        print("\(tip.name) - \(tip.district ?? "")")
    }
}
```

### 路线规划

**驾车路线规划**：

```swift
func planDrivingRoute(from: CLLocationCoordinate2D, to: CLLocationCoordinate2D) {
    let request = AMapDrivingRouteSearchRequest()
    request.origin = AMapGeoPoint(location: from)
    request.destination = AMapGeoPoint(location: to)
    searchAPI.aMapDrivingRouteSearch(request)
}

func onDrivingRouteSearchDone(_ request: AMapDrivingRouteSearchRequest, response: AMapDrivingRouteSearchResponse) {
    guard let route = response.route, let paths = route.paths, let firstPath = paths.first else { return }

    let distance = firstPath.distance
    let duration = firstPath.duration
    print("驾车距离: \(distance)米, 预计用时: \(duration)秒")
}
```

**步行路线规划**：

```swift
func planWalkingRoute(from: CLLocationCoordinate2D, to: CLLocationCoordinate2D) {
    let request = AMapWalkingRouteSearchRequest()
    request.origin = AMapGeoPoint(location: from)
    request.destination = AMapGeoPoint(location: to)
    searchAPI.aMapWalkingRouteSearch(request)
}
```

**公交路线规划**：

```swift
func planTransitRoute(from: CLLocationCoordinate2D, to: CLLocationCoordinate2D, city: String) {
    let request = AMapTransitRouteSearchRequest()
    request.origin = AMapGeoPoint(location: from)
    request.destination = AMapGeoPoint(location: to)
    request.city = city
    searchAPI.aMapTransitRouteSearch(request)
}
```

### SwiftUI 搜索界面

```swift
struct SearchView: View {
    @StateObject private var searchService = SearchService()
    @State private var searchText = ""
    @State private var selectedPOI: AMapPOI?

    var body: some View {
        VStack {
            HStack {
                TextField("搜索地点", text: $searchText)
                    .textFieldStyle(.roundedBorder)
                    .onSubmit {
                        searchService.searchPOI(keyword: searchText, city: "北京")
                    }

                Button("搜索") {
                    searchService.searchPOI(keyword: searchText, city: "北京")
                }
                .buttonStyle(.borderedProminent)
            }
            .padding()

            List(searchService.poiResults, id: \.uid) { poi in
                VStack(alignment: .leading) {
                    Text(poi.name)
                        .font(.headline)
                    Text(poi.address ?? "")
                        .font(.subheadline)
                        .foregroundColor(.secondary)
                    if let distance = poi.distance {
                        Text("\(distance)m")
                            .font(.caption)
                            .foregroundColor(.blue)
                    }
                }
                .onTapGesture {
                    selectedPOI = poi
                }
            }
        }
    }
}
```

---

## 合规与注意事项

### 地图审图号

根据《地图管理条例》，在中国境内发布地图必须标注审图号：

- 高德地图 SDK 内置审图号，会在地图右下角自动显示
- 自定义地图界面时**不可遮挡**审图号
- 审图号会随 SDK 版本更新，确保使用最新版 SDK
- 应用内截图展示地图时，审图号必须可见

### 测绘资质

- 使用高德/百度/腾讯等持证厂商的 SDK 即满足测绘资质要求
- **禁止**自行采集地理坐标数据并渲染为地图
- 应用内展示的地图数据必须来自具有甲级测绘资质的供应商
- 开发者无需单独申请测绘资质，但需在隐私政策中声明地图数据来源

### 隐私政策条款

应用隐私政策中必须包含以下内容：

1. 收集的位置信息类型（GPS、Wi-Fi、基站）
2. 位置信息的使用目的（导航、搜索、推荐等）
3. 位置信息的第三方共享（高德地图 SDK）
4. 用户的位置信息控制权（可随时关闭定位）
5. 第三方 SDK 的隐私政策链接

> ⚠️ **警告**：自 2021 年 11 月起，《个人信息保护法》要求在首次使用定位功能前必须获得用户明确同意。不得以拒绝提供服务为由强制用户授权位置权限。

### Apple 审核注意事项

| 审核要点 | 说明 |
|---------|------|
| 定位权限说明 | `Info.plist` 中的描述必须具体说明用途 |
| 后台定位 | 需要提供合理的后台定位理由，否则会被拒 |
| 地图审图号 | 地图界面审图号不可被遮挡或隐藏 |
| 隐私政策 | 必须包含位置信息收集和共享的说明 |
| 第三方 SDK 合规 | 高德 SDK 版本需为最新，确保合规声明有效 |
| 位置模拟 | 审核时 Apple 可能测试拒绝定位后的应用行为 |

> 💡 **提示**：提交审核时，在审核备注中说明应用使用高德地图 SDK 的原因（Apple Maps 国内数据不足），并附上定位权限的使用场景说明，有助于加快审核通过。

---

## 小结

| 主题 | 关键要点 |
|------|---------|
| 国内地图生态 | Apple Maps 国内数据差，高德/百度/腾讯三足鼎立，高德推荐首选 |
| SDK 集成 | 注册开发者账号、CocoaPods/手动安装、Key 配置、权限声明 |
| SwiftUI 封装 | UIViewRepresentable 封装 MAMapView、交互绑定、自定义标注 |
| 定位服务 | 高德定位 SDK、单次/持续定位、地理编码、SwiftUI 状态管理 |
| 搜索与路线 | POI 搜索、关键词提示、驾车/步行/公交路线规划 |
| 合规事项 | 审图号不可遮挡、测绘资质由 SDK 厂商提供、隐私政策必备、审核注意事项 |

国内地图 SDK 集成是面向国内用户应用的必经之路。高德地图 SDK 提供了从地图展示到导航的完整能力链，通过 SwiftUI 封装可以在现代 iOS 开发框架中流畅使用。务必重视合规要求，确保应用顺利通过审核。

← [地图与定位](./地图与定位.md) | [App Intents 与 Siri 快捷指令](./App-Intents与Siri快捷指令.md) →