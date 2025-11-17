# 升级Vditor CDN加载的mermaid从8.8.0到11.12.1

## Vditor CDN Mermaid升级方案

### 方案1: 修改CDN加载路径（推荐）

修改 `vditor/src/ts/markdown/mermaidRender.ts`，将CDN路径从vscode-vditor改为直接加载mermaid 11.12.1：

```typescript
// 当前代码：
addScript(`${cdn}/dist/js/mermaid/mermaid.min.js`, "vditorMermaidScript")

// 修改为：
addScript(`https://unpkg.com/mermaid@11.12.1/dist/mermaid.min.js`, "vditorMermaidScript")
```




```mermaid
graph TB
    subgraph "mio 0.6.23 (问题版本)"
        A1[Poll::new] --> A2[poll.register]
        A2 --> A3[事件处理]
        A3 --> A4[poll.deregister]
        A4 -.->|可能失败| A5[资源泄漏]
        A5 --> A6[FD计数增长]
        A6 --> A7[EMFILE错误]
        
        style A5 fill:#FFB6C1
        style A6 fill:#FFB6C1
        style A7 fill:#FF6B6B
    end
    
    subgraph "mio 0.8.12 (修复版本)"
        B1[Poll::new] --> B2[registry.register]
        B2 --> B3[事件处理]
        B3 --> B4[registry.deregister]
        B4 --> B5[正确资源清理]
        B5 --> B6[FD计数稳定]
        B6 --> B7[长期稳定运行]
        
        style B5 fill:#90EE90
        style B6 fill:#90EE90
        style B7 fill:#98FB98
    end
    
    C[升级决策] --> A1
    C --> B1
    C --> D[API适配]
    D --> E[features: tcp,udp → net]
    D --> F[Ready::readable → Interest::READABLE]
    D --> G[PollOpt → 简化API]
```
