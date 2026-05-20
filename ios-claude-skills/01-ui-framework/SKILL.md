---
name: ui-framework
description: 任何涉及界面、视图、布局、动画、ViewController、导航的任务
---

# UI 框架约定

## 框架选择
- **主框架：UIKit**（非特殊说明不使用 SwiftUI）
- 布局：**SnapKit** 自动布局，禁止 frame 硬编码
- 入口：**SceneDelegate**，无 Storyboard，无 .xib
- 导航：**UINavigationController**，禁止使用 NavigationStack

## 设计系统
- 颜色：统一从 `AppColors.swift` 引用，禁止直接写 `UIColor(hex:)`
- 字体：统一从 `AppFonts.swift` 引用，使用 SF Pro 系列
- 间距：基于 **8pt 基础网格**（8 / 16 / 24 / 32）
- 圆角：统一使用 `CornerRadius` 枚举（small=8, medium=12, large=20）

## ViewController 规范
- 命名后缀：`XxxVC`（不是 XxxViewController）
- 遵循 **MVVM**，VC 只负责绑定和响应，禁止在 VC 里写业务逻辑
- 生命周期：viewDidLoad 只做初始化，数据请求放 viewWillAppear 或 ViewModel
- 释放：注意 delegate/closure 循环引用，必要时用 `[weak self]`

## 组件规范
- 列表：**UICollectionView + Compositional Layout**，禁止新建 UITableView
- 弹窗：自定义 ViewController + present，禁止使用第三方弹窗库
- 图片加载：**Kingfisher**（网络图）/ 直接 UIImage（本地资源）
- 加载状态：使用项目内 `LoadingView` 组件，禁止使用第三方 HUD

## 禁止事项
- 禁止在 VC 里直接操作数据库或网络
- 禁止使用 `.xib` 或 Storyboard 创建新组件
- 禁止 `DispatchQueue.main.async` 嵌套超过一层
- 禁止在 UI 初始化之外的地方修改 UI（必须在主线程）
