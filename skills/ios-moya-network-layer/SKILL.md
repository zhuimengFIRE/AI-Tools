---
name: ios-moya-network-layer
description: >-
  基于 Moya TargetType、RSA 包裹的 JSON 请求体、SmartCodable 响应、多环境 baseURL
  与可选 Alamofire 分块上传，搭建 iOS Swift 网络层；命名按项目模块前缀自定义，
  JSON 与上传合并为单一网络入口类。适用于新增 API 封装或迁移到该技术栈时。
---

# iOS Moya 网络层

按下列结构生成网络封装，**总体分层与数据流不变**。**依赖**：Moya、Alamofire、SmartCodable；若需加解密则加 SwiftyRSA。

## 0. 命名：使用项目前缀，禁止写死 `FF` / `LL`

- 由用户或仓库约定一个**模块前缀**（如 `App`、`AC`、`Storage`），所有对外类型统一带此前缀；无前缀项目可用 `API`、`Network` 等中性名。
- 下表以占位符 **`{P}`** 表示前缀；示例：`{P}BaseRequest` 在具体项目中可为 `AppBaseRequest`、`ACBaseRequest`。
- `ApiService`、`ApiEnvironment`、第三方类型（Moya、Alamofire）保持通用名即可。

| 占位符 | 职责 |
|--------|------|
| `{P}BaseRequest` | 协议：`Codable` + `apiName() -> String` |
| `{P}BaseResponse<T>` | `SmartCodable`：`code` / `msg` / `data`，`success()`（通常 `code == 200`） |
| `{P}Network` | **唯一对外网络入口**（见下文合并说明） |
| `ApiEnvironment` | 多环境 `baseURL` |
| `ApiService` | `TargetType`：`postRequest(params: {P}BaseRequest)` 等 |
| `RSA`（可选） | 请求加密 / 响应解密；密钥走配置注入，勿写死生产密钥 |

## 1. 合并 `{P}Network`：原「JSON 客户端 + 上传」合一

将原先拆成两个类（例如历史上的 JSON 专用类与上传类）**合并为一个** `{P}Network`（单例或依赖注入均可，与项目风格一致），内部可同时持有：

- **`MoyaProvider<ApiService>`**：所有走 TargetType 的 JSON API（RSA、header、统一解析）。
- **上传**：OSS / `multipart/form-data` 等仍用 **`Alamofire.upload`**；先通过 `{P}Network.request` 拉取 policy/host，再在同一类中封装 `performUpload` / `uploadImage` 等方法，**不要**再建第二个「全局 JSON 入口」类。

可选：`endpointClosure` / `requestClosure` / `NetworkActivityPlugin` 等与超时、日志相关的逻辑，全部放在该类的 `Provider` 初始化处，避免一套 Provider 在 A 类、另一套在 B 类。

**禁止**：长期并存两个并列的「主网络单例」（除非短期迁移且文档标明废弃时间表）。兼容旧 `MultiTarget` 时，优先逐步收口到单一 `ApiService`，上传与非 Moya 请求仍归 `{P}Network` 管理。

## 2. TargetType 约定

- **method**：默认 `POST`；若需 GET 等，用**新增** `ApiService` case 表达，勿随意改现有 case 语义。
- **path**：`params.apiName()`；前导 `/` 与 base 重复片段按后端约定。
- **task**：`JSONEncoder` →（可选）`RSA.encrypt` 去空白 → `.requestData`；编码失败时统一错误策略（如 `.requestPlain` + 日志）。
- **headers**：设备、渠道、版本、时间戳、Token 等**只在此处**维护。

## 3. `request` 流程（合并后仍适用）

1. `provider.request(.postRequest(params: params))`。
2. **成功**：`Data` → `String`，校验非空、非 `"null"`、非 HTML 错误页、非 `"404"` 前缀等（可按业务增删）。
3. （可选）`RSA.decrypt` → `{P}BaseResponse<T>.deserialize(from:)`。
4. **业务码**：如 `code == 402` 清空登录态，与产品约定一致。
5. **失败**：映射为 `{P}BaseResponse`（含 `MoyaError` 的 `errorCode` 扩展或统一占位码与用户文案）。
6. **超时**：`URLSessionConfiguration` 或 `requestClosure` 内 `timeoutInterval`（如 60）。

上传分支：Alamofire 回调里对响应字符串仍用 **`{P}BaseResponse<上传结果模型>`** 解析，与 JSON 主流程保持一致；主线程回调、失败 Toast 等 UI 策略放在 `{P}Network` 私有方法中集中处理，避免调用方重复。

## 4. 新增接口清单（Agent）

1. 定义请求 struct，遵循 `{P}BaseRequest`，实现 `apiName()`。
2. 定义 `T: SmartCodable`（空数据用占位模型）。
3. 不拆 `ApiService` 的既有 case 语义；新 HTTP 语义用新 case。
4. 调用：`{P}Network.shared.request(..., modelType: T.self) { result in … }`，用 `result.success()` 判断业务成功。
5. 若含上传：在同一 `{P}Network` 增加方法，内部先 `request` 拿 OSS 参数再 `AF.upload`。

## 5. 安全与日志

- RSA、baseURL：**xcconfig / CI / 环境变量**，勿提交生产密钥。
- 日志封装（如项目内 `Log`）：生产环境对接口路径、密文、原文做脱敏或关闭。

## 6. 禁止项

- 未要求不得将 `SmartCodable` 换成 SwiftyJSON 等。
- 不另起与 `{P}Network` 平级的第二套全局 JSON Provider（迁移期除外且需标注废弃）。
- 不改变 `{P}BaseResponse` 字段语义与 `success()` 规则，除非后端契约变更。

## 7. 最小骨架（占位符版）

```swift
protocol {P}BaseRequest: Codable {
    func apiName() -> String
}

struct {P}BaseResponse<T: SmartCodable>: SmartCodable {
    var code: Int = 0
    var msg: String?
    @SmartAny var data: T?
    func success() -> Bool { code == 200 }
}

final class {P}Network {
    static let shared = {P}Network()
    private let provider: MoyaProvider<ApiService>
    // private let session: Session  // 按需
    private init() { /* Provider + 超时 + plugins */ }

    func request<T: SmartCodable>(
        _ params: {P}BaseRequest,
        modelType: T.Type, // 无 data 时可约定 `T` 为项目内的空占位 `SmartCodable`
        completion: @escaping ({P}BaseResponse<T>) -> Void
    ) { /* Moya + 校验 + RSA + deserialize + 业务码 */ }

    // func uploadImage(...) { /* AF.upload，内部可调用 request 取 OSS */ }
}

// ApiService: TargetType — POST + requestData（RSA 密文）
```

生成时将 `{P}` 替换为当前项目前缀；加密、header 键名、特殊业务码以**当前仓库**为准。本 skill 固定的是**分层、单入口、TargetType 数据流与合并原则**。
