---
name: project-architecture
description: 创建新文件、新模块、新功能、重构现有代码、设计数据流的任务
---

# 项目架构 DNA

## 目录结构
```
App/
├── Features/              # 功能模块（每个模块自包含）
│   ├── Camera/
│   │   ├── CameraVC.swift
│   │   ├── CameraViewModel.swift
│   │   └── Views/
│   ├── Settings/
│   └── Paywall/
├── Core/                  # 跨模块基础设施
│   ├── Network/           # API 层
│   ├── Storage/           # 本地持久化
│   ├── Analytics/         # 埋点
│   └── Extensions/        # Swift 扩展
├── DesignSystem/          # UI 组件库
│   ├── AppColors.swift
│   ├── AppFonts.swift
│   └── Components/        # 可复用 UI 组件
└── Resources/
    ├── Assets.xcassets
    ├── Localizable.strings
    └── Models/            # CoreML 模型文件
```

## 架构模式：MVVM
```
View (VC + UIView)
  ↕ 绑定
ViewModel（业务逻辑 + 状态）
  ↕ 调用
Service / Repository（数据层）
```

- **VC 职责：** 初始化视图、绑定 ViewModel、响应用户事件
- **ViewModel 职责：** 处理业务逻辑、持有状态、调用 Service
- **Service 职责：** 网络请求、本地存储、设备能力（相机、麦克风）
- **禁止跨层直接调用**（VC 不直接调 Service，ViewModel 不操作 UIView）

## 命名约定
| 类型 | 规范 | 示例 |
|------|------|------|
| ViewController | XxxVC | `CameraVC`, `SettingsVC` |
| ViewModel | XxxViewModel | `CameraViewModel` |
| Service | XxxService | `CameraService`, `AuthService` |
| Protocol | XxxProtocol 或 Xxxable | `CameraServiceProtocol` |
| 枚举 | 大写驼峰，case 小写 | `AppError.networkTimeout` |
| 常量 | enum + static let | `Layout.padding = 16` |

## 错误处理
- 所有错误统一走 `AppError` 枚举（避免裸 `Error` 类型扩散）
- 用户可见错误必须有**中文提示文案**
- 网络错误区分：无网络 / 超时 / 服务端错误 / 业务错误
- **禁止 `try!` 和强制解包 `!`**（除非有充分注释说明原因）
- 错误日志统一通过 `Logger.error()` 输出（不用 `print`）

## 数据流
- 状态管理：ViewModel 用 `@Published` + Combine / closure 回调（选一种，全项目统一）
- 本地存储优先级：UserDefaults（轻量配置）> CoreData（结构化数据）> FileManager（文件）
- 敏感数据（token、密钥）：**必须存 Keychain**，禁止存 UserDefaults

## 新功能开发流程
1. 在 `Features/` 下建对应目录
2. 先定义 Protocol（接口），再写实现
3. ViewModel 先写，VC 后写
4. 单元测试文件与源文件同目录（`XxxViewModelTests.swift`）
5. 新增第三方库必须在 README 的依赖列表更新

## 依赖管理
- 包管理器：**Swift Package Manager**（禁止新增 CocoaPods 依赖）
- 引入新库前评估：是否有系统 API 替代？维护是否活跃？

## 代码质量
- 单个文件不超过 400 行（超出考虑拆分）
- 单个函数不超过 50 行
- 禁止注释掉的废弃代码（直接删除，Git 有历史）
