---
name: ios-snapkit-layout
description: >-
  使用 SnapKit 在 iOS UIKit 项目中编写约束的统一规范，覆盖 makeConstraints 写法、SafeArea
  与键盘适配、priority/hugging/compression、iPad/SE 等多尺寸防漂移、动态字体与暗黑模式。
  适用于新增视图、调整布局或定位 UI 漂移问题时。
---

# iOS SnapKit 布局规范

只管布局；视图分层、MVVM 由 `ios-uikit-mvvm-baseline` 负责。

## 1. 何时用什么

| 写法 | 场景 |
|---|---|
| `makeConstraints` | 首次添加约束（必须先 `addSubview`） |
| `remakeConstraints` | 整块重做，等价于先全删再新建 |
| `updateConstraints` | 仅修改已存在约束的常量值（offset/inset 改数值） |

**禁止**：在同一个视图同一组 anchor 上反复 `makeConstraints`，会越叠越多导致冲突。修改用 `update`，结构换用 `remake`。

## 2. 标准写法

```swift
view.addSubview(titleLabel)
titleLabel.snp.makeConstraints { make in
    make.top.equalTo(view.safeAreaLayoutGuide.snp.top).offset(16)
    make.leading.trailing.equalToSuperview().inset(16)
}
```

约定：

- **必须先 `addSubview` 再 `makeConstraints`**，否则崩溃。
- **顶部/底部默认走 `safeAreaLayoutGuide`**，不要直接贴 `view.snp.top`，否则刘海/灵动岛/Home 横条会被遮。
- 左右用 `inset(16)` 而不是 `offset(16) / offset(-16)`，更不易写错符号。
- 不要把 `make` 重命名（保持 SnapKit 习惯，方便 grep）。

## 3. 优先级与抗压

文本省略、自适应高度时常用：

```swift
label.setContentHuggingPriority(.required, for: .vertical)
label.setContentCompressionResistancePriority(.defaultLow, for: .horizontal)
```

约束级 priority：

```swift
make.width.equalTo(120).priority(.high)
```

不要全用 `.required`，否则一冲突就 unsatisfiable。**调试 unsatisfiable 时**，先把所有非必须约束降到 `.high`，再看哪个真正必须。

## 4. ScrollView 标准三层模型

```swift
view.addSubview(scrollView)
scrollView.addSubview(contentView)   // 唯一直接子视图
contentView.addSubview(...)          // 其余视图加在 contentView 内

scrollView.snp.makeConstraints { $0.edges.equalToSuperview() }
contentView.snp.makeConstraints {
    $0.edges.equalToSuperview()              // 决定 contentSize
    $0.width.equalTo(scrollView.snp.width)   // 仅竖向滚动时锁宽
}
```

横向滚动则锁高度。**忘了锁宽/高 = ScrollView 不滚或滚错方向**。

## 5. 键盘与安全区

- 输入框页面优先用 `IQKeyboardManagerSwift`/`IQKeyboardManager`，避免每个 VC 重复处理。
- 自定义键盘适配：监听 `UIResponder.keyboardWillChangeFrameNotification`，在回调里 `view.layoutIfNeeded()` 配合 `update` 修改底部约束。

## 6. 多尺寸防漂移

- **绝对数值要分场景**：`width.equalTo(80)` 这种小尺寸 OK；大块布局用比例 `width.equalTo(superview).multipliedBy(0.5).inset(8)`。
- **iPad/Mac Catalyst**：`UIDevice.current.userInterfaceIdiom` 不能信，用 `traitCollection.horizontalSizeClass`。
- **小屏（iPhone SE）**：避免硬编码 `.height.equalTo(280)` 这种大块固定高度，让内部内容撑高度并 `scrollView` 兜底。
- **横竖屏**：使用 `viewWillTransition(to:with:)` 触发 `remakeConstraints` 而不是 `setNeedsLayout`。

## 7. 动态字体（Dynamic Type）

- `label.font = .preferredFont(forTextStyle: .body)` + `label.adjustsFontForContentSizeCategory = true`。
- 高度不要写死，让 intrinsic content size 撑开。
- 只在产品明确不允许放大字体的页面才用固定 font。

## 8. 暗黑模式

- 颜色全部用 `UIColor(named:)` 走 Asset Catalog 的 Any/Dark；**禁止**散落的 `UIColor(red: g: b: a:)`。
- 图标如需深浅色变体，走 Asset 的 Render as Template + `tintColor`，或两套 image。
- 自定义视图覆盖 `traitCollectionDidChange(_:)` 时只重画颜色，不重排约束。

## 9. 性能与排错

- 复杂列表用 `UITableView`/`UICollectionView` + cell 复用，不要全部塞 `UIStackView`（>20 行明显卡顿）。
- 看到日志 `Unable to simultaneously satisfy constraints`：把整段约束贴回，逐条注释定位**最早**冲突的那条。
- 多用 `view.setNeedsLayout(); view.layoutIfNeeded()` 强制刷新，少用 `DispatchQueue.main.async` 兜底。

## 10. Agent 编写步骤

1. `addSubview`（先父后子）。
2. 一个视图一段 `makeConstraints`，按 `top/leading/trailing/bottom/width/height` 顺序写。
3. 顶部/底部默认 `safeAreaLayoutGuide`。
4. 颜色用命名色；字体优先 `preferredFont`。
5. 改完跑 `ios-xcode-build-loop` 确认无 unsatisfiable 警告再交付。

## 11. 禁止项

- 在 `init` / `viewDidLoad` 之外（如 `viewWillAppear` 每次回来）重复 `makeConstraints`。
- `frame = ...` 与 `snp.makeConstraints` 混用。
- 自动布局视图同时设置 `translatesAutoresizingMaskIntoConstraints = true`（SnapKit 默认置 false，别再改回去）。
- 通过 `layoutSubviews` 手算 frame 来「修正」AutoLayout——这是死循环征兆。
