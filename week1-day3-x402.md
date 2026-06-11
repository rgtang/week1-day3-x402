# Project Notes · [项目名]

> Source: https://github.com/coinbase/x402
> Read on: 2026-06-11

## 1. 项目在做什么（一句话）

x402面向 AI Agent 和 API 的互联网原生支付协议，主要解决如何让程序（尤其是 AI Agent）像访问网页一样，自动为 API、数据、内容或计算服务付费。

## 2. 顶层架构图

graph TD

A["MCP / AI Agent"] --> B["x402 Client SDK"]

B --> C["HTTP Middleware<br/>Express / Next / Fastify / Hono"]

C --> D["x402 Resource Server"]

D --> E["Payment Required (402)"]

B --> F["Wallet (MetaMask / Signer)"]

D --> G["Facilitator Service"]

G --> H["Blockchain (Base / EVM / SVM / Stellar)"]

B --> G


## 3. 核心模块表

| 模块 | 路径 | 职责 | 关键文件 |
|---|---|---|---|
| **协议核心** | `typescript/packages/core/` | 定义全部类型、三方角色引擎（Client / ResourceServer / Facilitator）、HTTP 编解码 | `src/client/x402Client.ts` · `src/server/x402ResourceServer.ts` · `src/facilitator/x402Facilitator.ts` |
| **区块链机制层** | `typescript/packages/mechanisms/` | EVM / SVM / Stellar 的签名、链上验证与结算具体实现 | `evm/src/exact/client/scheme.ts` · `evm/src/exact/facilitator/eip3009.ts` · `svm/src/settlement-cache.ts` |
| **HTTP 框架中间件** | `typescript/packages/http/` | 将核心引擎接入 Express / Next / Fastify / Hono（服务端）及 fetch / axios（客户端） | `express/src/index.ts` · `fetch/src/index.ts` · `core/src/http/x402HTTPResourceServer.ts` |
| **扩展模块** | `typescript/packages/extensions/` | Bazaar 服务发现、EIP-2612 Gas 赞助、Sign-in-with-x 等可插拔能力 | `src/bazaar/` · `src/eip2612-gas-sponsoring/` · `src/sign-in-with-x/` |
| **MCP AI 集成** | `typescript/packages/mcp/` | 包装 MCP SDK，让 AI Agent 的 `callTool()` 自动处理 402 支付 | `src/client/x402MCPClient.ts` |
| **官网 & 文档** | `typescript/site/` · `docs/` | x402.org 落地页、生态展示；Mintlify 协议文档 | `app/ecosystem/` · `docs/docs.json` · `docs/getting-started/` |

## 4. 关键路径示例

用户动作：客户端发起支付：AI Agent / fetch 拦截 → 签名 → 重发请求

主路径（EIP-3009 / EVM 默认流）
阶段一：首次请求 → 收到 402

1. 用户代码调用包装后的 fetch
   → typescript/packages/http/fetch/src/index.ts
     wrapFetchWithPayment() 返回的匿名 async 函数
2. 发出原始 HTTP 请求，收到 402 响应
   → typescript/packages/http/fetch/src/index.ts
     匿名函数内 fetch(request)  ← 网络 I/O，异步等待
3. 解析 PAYMENT-REQUIRED 响应头（v2）或 body（v1）
   → typescript/packages/http/fetch/src/index.ts
     httpClient.getPaymentRequiredResponse(getHeader, body)
   → typescript/packages/core/src/http/x402HTTPClient.ts
     x402HTTPClient.getPaymentRequiredResponse()
     └─ 调用 decodePaymentRequiredHeader(header) 解出 PaymentRequired 对象
阶段二：构造支付 Payload（纯本地，无网络）

4. 触发 beforePaymentCreation hooks（可中止），然后路由到对应 Scheme
   → typescript/packages/core/src/client/x402Client.ts
     x402Client.createPaymentPayload(paymentRequired)
     └─ selectPaymentRequirements()  ← 过滤 + policy 链 + selector
5. 调用链上 Scheme 客户端创建 payload
   → typescript/packages/mechanisms/evm/src/exact/client/scheme.ts
     ExactEvmScheme.createPaymentPayload(x402Version, requirements, context)
     └─ 判断 assetTransferMethod: 'eip3009' 走默认分支
6. 构建 EIP-3009 Authorization 结构体
   → typescript/packages/mechanisms/evm/src/exact/client/eip3009.ts
     createEIP3009Payload(signer, x402Version, paymentRequirements)
     创建 { from, to, value, validAfter, validBefore, nonce }
7. ★ EIP-712 离线签名（私钥操作，异步）
   → typescript/packages/mechanisms/evm/src/exact/client/eip3009.ts
     signEIP3009Authorization(signer, authorization, requirements)
     └─ signer.signTypedData({ domain, types, primaryType, message })
        ← viem WalletClient / 硬件钱包 / MPC — 本地密钥签名，返回 0x{string}
8. 组装最终 PaymentPayload，执行 afterPaymentCreation hooks
   → typescript/packages/core/src/client/x402Client.ts
     x402Client.createPaymentPayload() 后半段
     └─ mergeExtensions() + enrichPaymentPayloadWithExtensions()
阶段三：携带签名重发请求

9. 将 PaymentPayload 序列化进 HTTP header
   → typescript/packages/core/src/http/x402HTTPClient.ts
     x402HTTPClient.encodePaymentSignatureHeader(paymentPayload)
     → encodePaymentSignatureHeader()  ← base64-url JSON 编码
     写入 header: "PAYMENT-SIGNATURE: <encoded>"
10. 发出第二次 HTTP 请求（携带签名）
    → typescript/packages/http/fetch/src/index.ts
      fetch(clonedRequest)  ← 网络 I/O，异步，跨进程到服务端
阶段四：服务端中间件验证（服务端进程）

11. 框架中间件（Express/Next/Fastify/Hono）截获请求
    → typescript/packages/core/src/http/x402HTTPResourceServer.ts
      x402HTTPResourceServer.processHTTPRequest(context)
      └─ extractPayment() → decodePaymentSignatureHeader(header) 解出 PaymentPayload
12. 路由到核心资源服务端做验证
    → typescript/packages/core/src/server/x402ResourceServer.ts
      x402ResourceServer.verifyPayment(paymentPayload, matchingRequirements)
      └─ 触发 beforeVerify hooks → 调用 facilitatorClient.verify()
13. ★ 异步 HTTP POST 到 Facilitator 服务（跨进程 / 跨网络）
    → typescript/packages/core/src/http/httpFacilitatorClient.ts
      HTTPFacilitatorClient.verify(paymentPayload, paymentRequirements)
      POST https://x402.org/facilitator/verify
      ← 返回 VerifyResponse { isValid: true/false }
阶段五：Facilitator 内部验证（Facilitator 进程）

14. Facilitator 核心调度
    → typescript/packages/core/src/facilitator/x402Facilitator.ts
      x402Facilitator.verify(paymentPayload, requirements)
      └─ 按 (scheme, network) 路由到 ExactEvmScheme facilitator
15. EVM 机制层验证签名
    → typescript/packages/mechanisms/evm/src/exact/facilitator/scheme.ts
      ExactEvmScheme.verify(payload, requirements, context)
      └─ verifyEIP3009(signer, payload, requirements, eip3009Payload)
16. 链上 ECDSA 签名恢复 / ERC-1271 验证（异步 RPC 调用）
    → typescript/packages/mechanisms/evm/src/exact/facilitator/eip3009.ts
      verifyEIP3009() 内
      └─ signer.verifyTypedData({ address, domain, types, message, signature })
         ← viem publicClient.verifyTypedData → eth_call 链上验证
17. 链上模拟执行
    → typescript/packages/mechanisms/evm/src/exact/facilitator/eip3009.ts
      simulateEip3009Transfer(signer, erc20Address, eip3009Payload, ...)
      ← eth_call 模拟 transferWithAuthorization（不广播）
阶段六：验证通过 → 放行 → 结算 → 链上写入

18. verify 通过，中间件放行请求，业务 handler 执行并响应
19. 响应发出后触发结算流程
    → typescript/packages/core/src/http/x402HTTPResourceServer.ts
      x402HTTPResourceServer.processSettlement(paymentPayload, requirements, ...)
20. 异步 HTTP POST 到 Facilitator /settle（跨进程 / 跨网络）
    → typescript/packages/core/src/http/httpFacilitatorClient.ts
      HTTPFacilitatorClient.settle(paymentPayload, paymentRequirements)
      POST https://x402.org/facilitator/settle
21. Facilitator 结算调度 → 二次校验
    → typescript/packages/core/src/facilitator/x402Facilitator.ts
      x402Facilitator.settle() → ExactEvmScheme.settle()
    → typescript/packages/mechanisms/evm/src/exact/facilitator/eip3009.ts
      settleEIP3009() → 内部再调用一次 verifyEIP3009()（无模拟）
22. ★★ 广播链上交易（最终写入）
    → typescript/packages/mechanisms/evm/src/exact/facilitator/eip3009.ts
      executeTransferWithAuthorization(signer, erc20Address, eip3009Payload)
      └─ signer.writeContract({ address, abi, functionName: 'transferWithAuthorization', args })
         ← viem walletClient.writeContract → eth_sendRawTransaction → 链上广播
23. 等待交易确认
    → typescript/packages/mechanisms/evm/src/exact/facilitator/eip3009.ts
      settleEIP3009() 内
      └─ signer.waitForTransactionReceipt({ hash: tx })
         ← 轮询 eth_getTransactionReceipt 至 status: 'success'
         返回 SettleResponse { success: true, transaction: txHash }

## 5. 3 个可借鉴的设计点

1. 设计点 1：离线签名 + 延迟结算（Sign-Once, Settle-Later）
解决了什么问题
传统 Web3 支付要求用户在线等待链上确认，体验差且容易超时。更本质的问题是："授权行为"和"链上执行"被捆绑在一起，导致服务端必须等待用户交互完成才能放行资源。

x402 把这两件事彻底分开：用户只负责离线签一个 EIP-712 结构体（毫秒级），签名携带了所有约束（to/value/validBefore/nonce）；服务端收到签名即可放行，链上广播由独立的 Facilitator 负责，完全异步。

代码位置：eip3009.ts 24-43

2. 设计点 2：版本化三维注册表取代 if/else 多链分发
解决了什么问题
多链项目最常见的反模式是：

// 反模式：每加一条链就要改核心逻辑
if (network === "eip155:8453") { return evmHandler(payload); }
else if (network === "solana:mainnet") { return svmHandler(payload); }
else if (network === "stellar:...") { return stellarHandler(payload); }
这把链实现和调度逻辑耦合死了。x402 用 Map<version, Map<network, Map<scheme, impl>>> 三层嵌套 Map，加新链只需在外部 .register()，核心逻辑永不修改。支持通配符（eip155:* 匹配所有 EVM 链），且天然版本化（v1 和 v2 协议并存互不干扰）。

代码位置：x402Client.ts 488-502

3. 设计点 3：可中止 + 可恢复的生命周期 Hook 系统
解决了什么问题
Web3 业务中有大量横切需求：日志记录、风控拦截、gas 赞助、重试逻辑、审计追踪……如果都写进核心逻辑里，代码会迅速膨胀且无法拔插。

x402 在每个关键操作（创建支付、验证、结算）前后都插入了 Hook 链，每个 Hook 有三种能力：

中止：return { abort: true, reason } — 拦截这次操作
恢复：return { recovered: true, result } — 从失败中注入备用结果
透传：return void — 什么都不做，继续下一个 Hook
三种返回值构成了完整的"拦截器 + 降级"模式，且完全不需要修改核心代码。

代码位置：x402Facilitator.ts  43-61 303-394

## 6. 我的疑问 / 不确定的点

- [一两条留给周末直播问导师的问题]