# 自动回复接口 v1.0

[返回产品首页](../README.md)

每个租户可以配置自己的地址和密钥，请求与响应结构保持一致。外部知识库只返回处理决定，发送对象由 Signal Hub 从原消息确定。

## 连接要求

- HTTP POST，公网 HTTPS、443 端口，可解析到公网 IPv4。
- `Authorization: Bearer <租户接口密钥>`；`Content-Type: application/json`。
- 不支持内网、自签名证书、跳转、URL 查询参数或流式响应。
- 总调用超时 15 秒，响应体最多 64 KiB；成功处理返回 HTTP 200。
- 所有示例字段均必填，不接受额外字段。

## 请求示例

```json
{
  "version": "1.0",
  "request_id": "example_request_001",
  "session_id": "example_session_001",
  "account_id": "example_account_001",
  "customer_id": "example_customer_001",
  "message": {
    "id": "example_message_001",
    "type": "text",
    "text": "我们有20人使用，请问如何收费？",
    "sent_at": "2026-09-10T10:00:00+08:00"
  }
}
```

ID 均为不透明标识，最长 128 字符；同一实际会话的 session_id 保持稳定。问题非空且最多 4,000 个 Unicode 码点，时间需包含时区。v1 不附带历史消息；后台接口测试使用虚拟会话标识。

## 响应示例

有答案，或需要追问时：

```json
{
  "version": "1.0",
  "request_id": "example_request_001",
  "action": "reply",
  "reply": { "type": "text", "text": "请问您需要按月还是按年使用？" },
  "reason_code": null
}
```

答案为非空纯文本，最多 2,000 个 Unicode 码点。request_id 必须原样返回；不允许指定接收人。

不回复本条消息：

```json
{
  "version": "1.0",
  "request_id": "example_request_001",
  "action": "ignore",
  "reply": null,
  "reason_code": "NO_MATCH"
}
```

ignore 的 reason_code 只能是 NO_MATCH 或 NOT_APPLICABLE。

需要人工处理：

```json
{
  "version": "1.0",
  "request_id": "example_request_001",
  "action": "handoff",
  "reply": null,
  "reason_code": "NEEDS_HUMAN"
}
```

handoff 的 reason_code 只能是 NEEDS_HUMAN 或 SENSITIVE_TOPIC。正式模式下会暂停会话，直到人工恢复；试运行只记录建议。reason_code 不是发送给客户的文案。

## 启用与运行边界

先以关闭模式保存接口及密钥，进行接口测试，再选择账号并开启试运行。正式回复需要当前配置版本的接口测试通过，并且平台发送策略允许目标联系人。

当前每 5 秒扫描已入库消息，只处理启用后、两分钟内、入库满两秒的新私聊文字。账号需在线，联系人需已同步；免打扰、撤回、自己的消息及接管会话不触发。

当前连续消息只处理最新一条，不自动合并。同一原消息只建立一个任务；接口结果返回及发送前会复核会话，配置变化、人工回复或出现更新消息时放弃旧答案。每次保存重新划定消息起点，不补发旧消息。

基础版每轮最多收集 100 条、串行处理 10 条；设有会话与租户频率保护，超限跳过。慢接口及积压可能导致消息过期，不能视作高并发客服方案。

网络失败、非 200、格式不符或 request_id 不匹配均不发送答案，不自动重试。发送结果未知时阻止会话后续自动处理，须人工核对后恢复。已提交的发送不能通过关闭功能撤回。

后台可下载 JSON Schema 和完整协议。问答数据会传至租户配置的知识库，应事先确认数据处理权限与服务方隐私安排。试运行处理记录包含客户问题和候选答案，仅供有权限的管理员查看。
