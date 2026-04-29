---
name: ios-uikit-mvvm-baseline
description: >-
  基于 UIKit + MVVM(+Coordinator) 搭建 iOS 模块的脚手架规范，约定 BaseViewController、
  ViewModel 数据流（输入/输出）、Coordinator 路由、依赖注入与生命周期。适用于在 UIKit 项目
  中新增页面/模块、或将既有 MVC 重构为 MVVM 时使用。
---

# iOS UIKit + MVVM 脚手架

为 UIKit 项目新增模块时的标准分层。**布局** 由 `ios-snapkit-layout` skill 负责；**网络** 由 `ios-moya-network-layer` skill 负责；本 skill 只规约 **VC / VM / Coordinator** 三者关系。

## 0. 命名占位符

沿用 `ios-moya-network-layer` 的约定：用 `{P}` 表示项目模块前缀（如 `SD`、`App`），仓库内统一一个，禁止写死 `FF`/`LL` 之类历史前缀。

## 1. 文件与目录

新模块按特性切分（feature folder），不要按类型分（避免 `ViewControllers/` 平铺几十个文件）：

```
Modules/
└── Login/
    ├── LoginViewController.swift
    ├── LoginViewModel.swift
    ├── LoginCoordinator.swift
    ├── Views/                 // 自定义 cell / subview
    └── Models/                // 仅本模块的 DTO/UI Model
```

公共能力放 `Common/`、`Base/`、`Extensions/`，跨模块复用的网络模型放 `Network/`。

## 2. BaseViewController 约定

项目应有且**仅有一个** `{P}BaseViewController`，承担：

- 统一 `view.backgroundColor`、导航栏返回、状态栏样式。
- 提供 `loadingView` / `emptyView` / `errorView` 占位（隐藏式 API，VM 触发显示）。
- 统一 `addObserver` / `removeObserver` 的生命周期对接点。
- 不直接持有业务，业务全部在子类 + ViewModel。

**禁止**在 BaseVC 里塞业务（如登录态判断、埋点 SDK 初始化），这些走 Coordinator 或 App 层。

## 3. ViewModel 三件套

ViewModel 不持有 `UIView`/`UIViewController`，**只暴露状态与命令**。推荐 input/output 风格：

```swift
final class LoginViewModel {
    struct Input {
        let phone: AnyPublisher<String, Never>
        let code:  AnyPublisher<String, Never>
        let tap:   AnyPublisher<Void, Never>
    }
    struct Output {
        let canSubmit:    AnyPublisher<Bool, Never>
        let loading:      AnyPublisher<Bool, Never>
        let errorMessage: AnyPublisher<String, Never>
        let didLogin:     AnyPublisher<Void, Never>   // 路由信号，给 Coordinator
    }

    func transform(_ input: Input) -> Output { /* 组合网络/校验/状态 */ }
}
```

- **未引入 Combine/RxSwift 的项目**：用闭包 `var onLoading: ((Bool) -> Void)?` + 方法 `func login(phone:code:)`，但仍保持「VM 不引用 UI」。
- VM 内部依赖（Service、Repository）走构造器注入，方便单测。
- VM 不做 `UIAlertController.show`，错误以 `Output.errorMessage` 抛给 VC。

## 4. ViewController 职责（**只做三件事**）

1. 组装 UI（创建子视图、调用 SnapKit）。
2. 把 UI 事件喂给 VM（`input`），订阅 VM 输出（`output`）刷新 UI。
3. 在 `didLogin` 之类的路由信号上回调 Coordinator。

VC 内**禁止**：直接发起网络请求、直接 `UserDefaults` 读写业务字段、直接 `present` 下一页 VC（路由由 Coordinator 决定）。

## 5. Coordinator 路由

每个模块/Tab 一个 Coordinator，持有 `UINavigationController`，负责 `start()` 和子 Coordinator 串联：

```swift
final class LoginCoordinator: Coordinator {
    let nav: UINavigationController
    var onFinish: (() -> Void)?
    func start() {
        let vm = LoginViewModel(service: AuthService())
        let vc = LoginViewController(viewModel: vm)
        vm.output.didLogin.sink { [weak self] in self?.onFinish?() }
        nav.setViewControllers([vc], animated: false)
    }
}
```

- AppDelegate / SceneDelegate 持有根 Coordinator。
- 子模块跳子模块通过 Coordinator 间通信，不在 VC 里 `pushViewController`。
- 小项目/单页面也可省略 Coordinator，但**整个项目要么都用、要么都不用**，不混搭。

## 6. 依赖注入

- 构造器注入优先，其次属性注入；**不要**用全局单例直接在 VM 里 `XxxManager.shared`。
- 跨模块共享 service（如 `AuthService`、`AnalyticsService`）通过 App 层容器（轻量手写或 Swinject）下发。

## 7. 异步与线程

- 新代码优先 `async/await`；既有 closure/Combine 风格保持一致即可。
- VM 里**不要**主动 `DispatchQueue.main.async`，把线程切换收口在 Service 边界或 VC 订阅处（用 `.receive(on: DispatchQueue.main)`）。

## 8. Agent 新增页面清单

1. 新建 `Modules/<Feature>/` 目录与 `XxxViewController/ViewModel/Coordinator.swift`。
2. VC 继承 `{P}BaseViewController`，不要继承 `UIViewController`。
3. 布局调用 `ios-snapkit-layout` skill。
4. 网络走 `{P}Network.shared.request(...)`（`ios-moya-network-layer` skill）。
5. 在父 Coordinator 的 `start()` 里挂载新 Coordinator。
6. 如有跳转，新 VC 不直接 `present/push`，通过 VM output 信号交给 Coordinator。
7. 改完跑 `ios-xcode-build-loop` skill 验证编译。

## 9. 禁止项

- VM 引用任何 `UIKit` 类型（`UIImage` 这种纯数据类型可放过，但 `UIView` 一律禁止）。
- VC 里写「网络 → 解析 → 持久化 → 弹窗」的全流程业务串。
- 不要为了「快」在 BaseVC 里塞 if-else 业务分支。
- 不复制粘贴另一个 VC 改成新 VC——先抽 ViewModel，再差异化 UI。
