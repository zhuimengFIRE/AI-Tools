---
name: ios-swift-feature
description: >-
  在 iOS UIKit + Swift 项目中开发一个新页面/模块的落地节奏，覆盖 ViewController /
  ViewModel / 子视图分层、状态管理（闭包 / Combine / RxSwift 选型）、生命周期、
  开发期数据预览、与 SwiftUI 混合栈的桥接（UIHostingController / UIViewControllerRepresentable）。
  与 ios-uikit-mvvm-baseline 配合：baseline 管「项目级约定」，本 skill 管「具体页面落地」。
---

# iOS UIKit + Swift 模块开发

> 本 skill 聚焦「新建一个 UIKit 页面」的实操节奏。**架构约定（VC/VM/Coordinator 三者关系、依赖注入、BaseVC）由 `ios-uikit-mvvm-baseline` 负责**；**布局** 由 `ios-snapkit-layout` 负责；**网络** 由 `ios-moya-network-layer` 负责。本文件不重复这些。

## 0. 前置判断

进入新模块前先确认三件事，写在 PR 描述里：

1. **最低部署版本**：决定能否用 `async/await`（iOS 13+）、`Combine`（iOS 13+）、`@MainActor`（iOS 13+ + Swift 5.5+）。
2. **响应式栈**：项目里是 `Combine` / `RxSwift` / 纯闭包？**新模块跟随既有项目，不要混栈**。
3. **是否混合栈**：宿主是纯 UIKit 还是 UIKit+SwiftUI 混合。决定是否需要 `UIHostingController` 桥接。

## 1. 文件分层

按特性切分（feature folder），结构对齐 `ios-uikit-mvvm-baseline`：

```
Modules/
└── Profile/
    ├── ProfileViewController.swift     // 视图（pure UI + 订阅）
    ├── ProfileViewModel.swift          // 状态 + 行为
    ├── ProfileCoordinator.swift        // 路由（小项目可省略）
    ├── Views/
    │   ├── ProfileHeaderView.swift     // 自定义子视图
    │   └── ProfileCell.swift           // Cell
    └── Models/
        └── ProfileUIModel.swift        // 仅本模块的 UI Model（区别于网络 DTO）
```

约定：

- 一个 `*ViewController.swift` 只暴露**一个**对外 VC，私有子视图用 `private final class` 或 `extension` 收纳。
- ViewModel 不导入 `UIKit`（除 `UIImage` 这种纯数据类型），保持可单测。
- 跨模块复用的 Cell / View 抽到 `Common/Views/`，不要从其它 feature 反向依赖。

## 2. 状态管理选型

| 项目栈 | 选型 |
|---|---|
| 纯闭包（既有项目无 Combine/Rx） | VM 暴露 `var onStateChange: ((State) -> Void)?` + 方法 `func load()` |
| Combine（iOS 13+ 新项目） | VM 暴露 `@Published var state: State` 或 `let state: AnyPublisher<State, Never>` |
| RxSwift | VM 暴露 `let state: Driver<State>`（UI 流统一 `Driver`） |
| 纯 async/await（页面级简单数据） | VC 直接 `Task { try await vm.load() }`，VM 用 `@MainActor` + `var state: State` |

**选型原则**：

- VM 对外暴露**一个**可订阅的「状态流」，不要散落 `isLoading: Bool` / `error: String?` / `data: [T]?` 三件套。
- 状态用 `enum State` 表达：`idle / loading / loaded(T) / failed(Error)`。VC 用 `switch` 渲染。

## 3. ViewModel 模板（Combine 版）

```swift
import Combine
import Foundation

final class ProfileViewModel {
    enum State { case idle, loading, loaded(Profile), failed(String) }

    @Published private(set) var state: State = .idle

    private let service: ProfileServicing
    private var bag = Set<AnyCancellable>()

    init(service: ProfileServicing) { self.service = service }

    func load() {
        state = .loading
        service.fetch()
            .receive(on: DispatchQueue.main)
            .sink(
                receiveCompletion: { [weak self] in
                    if case .failure(let e) = $0 { self?.state = .failed(e.localizedDescription) }
                },
                receiveValue: { [weak self] in self?.state = .loaded($0) }
            )
            .store(in: &bag)
    }
}
```

要点：

- 服务依赖（`service`）走协议 + 构造器注入，方便 mock 单测。
- 线程切换收口在 VM 边界（`.receive(on: .main)`），VC 拿到的状态都已经在主线程。
- `@Published` 只对外**只读**（`private(set)`），外部要改状态只能调用方法。

### async/await 版（更现代）

```swift
@MainActor
final class ProfileViewModel {
    private(set) var state: State = .idle { didSet { onStateChange?(state) } }
    var onStateChange: ((State) -> Void)?

    private let service: ProfileServicing
    init(service: ProfileServicing) { self.service = service }

    func load() async {
        state = .loading
        do { state = .loaded(try await service.fetch()) }
        catch { state = .failed(error.localizedDescription) }
    }
}
```

> 选其中一种，**全项目保持一致**。

## 4. ViewController 模板

```swift
import UIKit
import Combine

final class ProfileViewController: {P}BaseViewController {
    private let vm: ProfileViewModel
    private var bag = Set<AnyCancellable>()

    private let headerView = ProfileHeaderView()
    private let retryButton = UIButton(type: .system)

    init(vm: ProfileViewModel) {
        self.vm = vm
        super.init(nibName: nil, bundle: nil)
    }
    required init?(coder: NSCoder) { fatalError() }

    override func viewDidLoad() {
        super.viewDidLoad()
        setupSubviews()        // 加 subview
        setupConstraints()     // SnapKit
        bindViewModel()        // 订阅
        vm.load()              // 首次加载
    }

    private func bindViewModel() {
        vm.$state
            .receive(on: DispatchQueue.main)
            .sink { [weak self] in self?.render($0) }
            .store(in: &bag)
    }

    private func render(_ state: ProfileViewModel.State) {
        switch state {
        case .idle, .loading: showLoading()
        case .loaded(let p):  hideLoading(); headerView.configure(p)
        case .failed(let m):  hideLoading(); showError(m) { [weak self] in self?.vm.load() }
        }
    }
}
```

要点：

- `viewDidLoad` 三段式：**搭 UI → 订阅 VM → 触发首次加载**，顺序固定。
- 渲染入口收口到一个 `render(_ state:)`，便于断点 + 单元测试 VC。
- 订阅一律 `[weak self]`，`bag` 随 VC 一起释放。
- `loading` / `error` 占位由 `{P}BaseViewController` 提供（见 baseline skill），不要每个 VC 自造一份。

## 5. 导航

详见 `ios-uikit-mvvm-baseline` 的 Coordinator 章节。本 skill 只补充**落地时的判断点**：

| 场景 | 处理 |
|---|---|
| 模块内子页跳转（详情页） | VM 抛 output 信号 → Coordinator `pushViewController` |
| 弹一个独立 modal（图片预览、弹窗） | VC 内直接 `present`，不走 Coordinator（无业务后续） |
| 跨模块跳转（登录、个人主页） | 只能通过 Coordinator 间通信，**禁止** `import Login` 跨模块导入 VC 直接 push |

## 6. 与 SwiftUI 互操作（混合栈）

- **UIKit 嵌 SwiftUI 视图**：`UIHostingController(rootView:)`，作为子 VC 加进来：

  ```swift
  let host = UIHostingController(rootView: ProfileHeaderSwiftUIView(vm: vm))
  addChild(host)
  view.addSubview(host.view)
  host.view.snp.makeConstraints { $0.edges.equalToSuperview() }
  host.didMove(toParent: self)
  host.view.backgroundColor = .clear   // 透出宿主背景
  ```

- **SwiftUI 嵌 UIKit 视图/控制器**：`UIViewControllerRepresentable`。`updateUIViewController` 里**只更新数据**，不要重建实例。
- 数据双向：`@Binding`（SwiftUI 侧）↔︎ closure / Combine（UIKit 侧）。**禁止** `NotificationCenter` 跨栈兜底。
- 与 SwiftUI 模块开发协同时，参考 `ios-swiftui-feature` skill。

## 7. 开发期数据预览（替代 SwiftUI 的 #Preview）

UIKit 没有 SwiftUI Preview，但仍要保证**改完不用走完真实流程**就能看到不同状态。落地手段：

1. **Sample Data 工厂**：每个 UI Model 有 `static var stub: Self` / `static var stubFailed: Self`。
2. **VM 提供预设状态构造器**：`static func preview(loaded: Profile) -> ProfileViewModel`，开发分支用 mock service。
3. **DEBUG 启动开关**：`#if DEBUG` 下读环境变量 `PREVIEW_STATE=loading|loaded|failed`，VC 启动时直接进对应状态。
4. **SwiftUI Preview 桥接（可选）**：用 `UIViewControllerRepresentable` 把 VC 包成 SwiftUI 预览，跑在 Xcode Canvas 里：

   ```swift
   #if DEBUG
   import SwiftUI
   struct ProfileVCPreview: UIViewControllerRepresentable {
       func makeUIViewController(context: Context) -> ProfileViewController {
           ProfileViewController(vm: .preview(loaded: .stub))
       }
       func updateUIViewController(_ vc: ProfileViewController, context: Context) {}
   }
   #Preview { ProfileVCPreview() }
   #endif
   ```

  > Xcode 15+ 支持 `#Preview` 直接作用于 UIKit `UIViewController`，更简洁。

## 8. 性能

- **列表必用复用**：`UITableView` / `UICollectionView` 始终走 `dequeueReusableCell`，禁止 `for-loop` 塞 cell 进 stackView 的"快糙猛"写法。
- **Cell 高度**：能用 `UITableView.automaticDimension` + `estimatedRowHeight` 就别手算；**频繁滚动卡顿才**预算缓存。
- **图层**：圆角 / 阴影 / 模糊优先用 `cornerCurve = .continuous` + 离屏离屏告警检查，不要无脑 `shouldRasterize = true`。
- **图片**：用 SDWebImage / Kingfisher 等带磁盘缓存方案，禁止主线程 `UIImage(named:)` 大图。
- **频繁刷新的状态**拆到子 ViewModel / 子 View，不要触发整个父 VC `render` 全量重算。
- 滚动卡顿先 Profile（Time Profiler / Animation Hitches），再优化；**不要凭感觉**改。

## 9. Agent 新增页面清单

执行下面的步骤完成一个新页面：

1. 在 `Modules/<Feature>/` 建 `ViewController` + `ViewModel` + `Views/`（视情况加 `Coordinator`、`Models/`）。
2. VC 继承 `{P}BaseViewController`（baseline skill），不直接继承 `UIViewController`。
3. VM 用一个 `enum State` 表达页面状态；服务依赖通过协议构造器注入。
4. 布局调用 `ios-snapkit-layout` skill。
5. 网络调用 `ios-moya-network-layer` skill 的 `{P}Network.shared.request(...)`。
6. `viewDidLoad` 三段式：`setupSubviews → setupConstraints → bindViewModel → vm.load()`。
7. 渲染入口收口到 `render(_ state:)`，用 `switch` 分发。
8. 给 UI Model 写 `static var stub`，至少能在 DEBUG 下进 `loaded` / `failed` 两种态。
9. 跳转走 Coordinator，不在 VC 里 `pushViewController`。
10. 改完跑 `ios-xcode-build-loop` skill 编译验证；UI 改动跑 `ios-simulator-qa` skill 做 UI 验收。

## 10. 禁止项

- VM 引用 `UIView` / `UIViewController`（`UIImage` 纯数据可放过）。
- VC 里散落 `isLoading: Bool` / `errorMessage: String?` / `data: [T]?` 三件套——**用 `enum State` 替代**。
- VC 里写「网络 → 解析 → 持久化 → 弹窗」全流程业务串。
- 在 `viewDidLoad` 之外的生命周期里第一次 `addSubview`（导致重复加 subview / 约束冲突）。
- 在 cell 复用回调里发起网络请求（用 prefetch API 或 VM 内置缓存）。
- 用 `NotificationCenter` 替代 VC↔VM 数据流。
- 拷贝另一个 VC 改成新 VC——先抽公共 ViewModel 父类或协议，再差异化 UI。
- 在 cell 内部直接 `UIApplication.shared.keyWindow?.rootViewController?.present(...)`——所有跳转回到 Coordinator。
