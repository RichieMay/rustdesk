# TCP 打洞方向无关修复设计

## 背景

当前 RustDesk 官方 TCP 打洞逻辑是一个有方向偏好的模型:

- 发起方: 使用连 hbbs 时的本地地址 `local_addr`,主动 `connect` 被连方的公网地址。
- 被连方: 使用连 hbbs 时的本地地址 `local_addr`,先发一个短暂的反向 `connect` 探测包,但丢弃这条连接结果;随后关闭 hbbs socket,在同一个 `local_addr` 上 `listen/accept`。

也就是说,最终官方业务链路只真正使用:

```text
发起方 active connect -> 被连方 passive listen
```

被连方的反向 `connect` 只是 NAT 探测,返回的 socket 不进入业务通信。

这会导致方向敏感:

- 被连方是锥形 NAT 时,发起方 connect 被连方 hbbs 公网端口通常可达,TCP 打洞成功。
- 被连方是对称 NAT/CGNAT 时,发起方 connect 被连方 hbbs 公网端口会被端点相关过滤丢弃,TCP 打洞失败。
- 反过来,如果对称 NAT 一侧作为发起方,它主动 connect 锥形 NAT 被连方时可达,所以反向能成功。

## 目标

修复 TCP 打洞方向相关性,使“至少一侧为锥形 NAT”的场景中,无论谁作为发起方,都能优先直连;双方对称 NAT 时仍应回退 relay。

同时尽量保持与官方旧客户端兼容,不修改 hbbs 协议。

## 核心方案

把 TCP 打洞从单一官方路径:

```text
发起方 active connect -> 被连方 passive listen
```

扩展为双候选路径:

```text
候选 A: 发起方 active connect -> 被连方 passive listen
候选 B: 被连方 active connect -> 发起方 passive listen
```

双方都使用各自连接 hbbs 时得到的 `local_addr`:

```text
listen(local_addr)
connect(peer_public_addr, local_addr)
```

任意一条 TCP stream 成功后,就可以进入 RustDesk 原有业务握手:

- 发起方使用 `Client::secure_connection(...)`
- 被连方使用 `server::create_tcp_connection(...)`

## 去重与优先级

为了兼容官方原行为,不使用 ID 仲裁,而使用角色优先级 + 短去重窗口。

核心原则:优先链路成功立即定案(不等,保持官方原延迟);只有兜底链路先成功时,才给优先链路一个 300ms 观察窗口,不再等优先链路的完整 connect 超时。

### 发起方

```text
active connect 优先,passive listen 兜底
```

时序:

```text
1. 同时启动 active connect 与 passive listen。
2. active connect 成功 -> 立即选中 active,丢弃 listen,不等。
3. passive listen 先成功 -> 暂存为兜底,给 active connect 最多 300ms 观察窗口。
   - 300ms 内 active 也成功 -> 仍选 active(官方链路),丢弃 listen。
   - 300ms 到 active 还没成功 -> 使用 listen 兜底。
4. 两条都失败 -> 回退 relay。
```

规则汇总:

```text
active 成功 + listen 成功 -> 使用 active,丢弃 listen
active 成功 + listen 失败 -> 使用 active
active 失败 + listen 成功 -> 使用 listen
active 失败 + listen 失败 -> 回退 relay
```

### 被连方

```text
passive listen 优先,active connect 兜底
```

时序:

```text
1. 同时启动 passive listen 与 active connect。
2. passive listen 成功 -> 立即选中 listen,丢弃 active,不等。
3. active connect 先成功 -> 暂存为兜底,给 passive listen 最多 300ms 观察窗口。
   - 300ms 内 listen 也成功 -> 仍选 listen(官方链路),丢弃 active。
   - 300ms 到 listen 还没成功 -> 使用 active 兜底。
4. 两条都失败 -> 等待发起方 relay / 失败回退。
```

规则汇总:

```text
listen 成功 + active 成功 -> 使用 listen,丢弃 active
listen 成功 + active 失败 -> 使用 listen
listen 失败 + active 成功 -> 使用 active
listen 失败 + active 失败 -> 等待发起方 relay / 失败回退
```

这样双方锥形 NAT 同时建立两条连接时,双方会共同保留官方原链路:

```text
发起方 active connect -> 被连方 passive listen
```

另一条反向链路会被双方丢弃。

### 300ms 观察窗口的设计依据

- 优先链路成功时不等,是为了双方锥形这种最常见场景保持官方原延迟。
- 兜底链路先成功时只等 300ms 而不等完整 connect 超时(最差可达 18s),是为了让"锥形发起 -> 对称被连"这种反向链路场景尽快用上反向连接,而不是空等 active 超时。
- 300ms 足以覆盖两条链路几乎同时打通的时序差;超过 300ms 优先链路还没成功,基本可判定优先链路无法打通。

## 兼容性分析

### 新发起方 -> 旧被连方

旧被连方仍只使用官方 listen 路径。新发起方 active connect 优先,因此仍命中官方原链路。新增 listen 只是兜底,不会破坏旧客户端。

### 旧发起方 -> 新被连方

旧发起方仍只 active connect。新被连方 listen 优先,因此仍命中官方原链路。新被连方 active connect 只是兜底,不会优先破坏旧客户端。

### 新 -> 新,双方锥形

两条链路都可能成功。按角色优先级:

- 发起方保留 active
- 被连方保留 listen

双方选择同一条官方原链路,避免重复会话。

### 新 -> 新,锥形发起 -> 对称被连

官方路径失败:

```text
发起方 active -> 对称被连方 listen
```

反向路径成功:

```text
对称被连方 active -> 锥形发起方 listen
```

按角色优先级:

- 发起方 active 失败,listen 成功 -> 使用 listen
- 被连方 listen 失败,active 成功 -> 使用 active

双方选择同一条反向链路,实现方向无关。

### 双方对称

两条路径都大概率失败,最终回退 relay。

## 实现要点

### 1. 发起方 client.rs

在当前 `connect_futures` 中,除原有 active connect future 外,增加 passive listen future:

```text
active: connect_tcp_local(peer_addr, Some(local_addr), connect_timeout)
passive: new_listener(local_addr, true).accept()
```

不能简单 `select_ok` 谁先成功就立即使用,否则双方锥形时可能选中不同连接。应使用角色优先级 + 短去重窗口:

1. 同时启动 active/listen。
2. active(connect,优先链路)成功 -> 立即选中 active,不等 listen,保持官方原延迟。
3. listen(兜底链路)先成功 -> 只给 active 最多 300ms 宽限,不再等 active 的完整 connect 超时。
4. 300ms 内 active 也成功 -> 仍选 active(官方链路);300ms 到了 active 还没成功 -> 使用 listen 兜底。
5. 两条都失败,进入原 relay fallback。

### 2. 被连方 rendezvous_mediator.rs

原逻辑中:

```rust
allow_err!(socket_client::connect_tcp_local(peer_addr, Some(local_addr), 30).await);
```

只是探测并丢弃结果。需要改为双候选:

```text
passive: new_listener(local_addr, true).accept()
active: connect_tcp_local(peer_addr, Some(local_addr), timeout)
```

同样使用角色优先级 + 短去重窗口:

1. 两个候选并发。
2. listen(passive,优先链路)成功 -> 立即选中 listen,不等 active。
3. active(connect,兜底链路)先成功 -> 只给 listen 最多 300ms 宽限,不再等 listen 的完整超时。
4. 300ms 内 listen 也成功 -> 仍选 listen(官方链路);300ms 到了 listen 还没成功 -> 使用 active 兜底。
5. 两条都失败,按原逻辑失败/relay。

### 3. 业务流使用

发起方候选返回后继续走原来的:

```rust
Client::secure_connection(...)
```

被连方候选返回后走:

```rust
crate::server::create_tcp_connection(server, stream, peer_addr, true, meta).await
```

### 4. 不改 hbbs 协议

现有 PunchHoleRequest/PunchHoleResponse 已经提供双方公网地址和 local_addr 复用所需条件。该方案不需要新增字段,不需要改 hbbs。

## 注意事项

- listen 和 connect 都要绑定同一个 `local_addr`,依赖当前已有的 `new_listener(local_addr, true)` 与 `connect_tcp_local(peer, Some(local_addr), ...)` 的地址复用能力。
- 如果平台不允许同端口同时 listen/connect,不能让整个 TCP 打洞直接失败;需要保留已有成功候选,并在必要时回退到单候选路径。
- 第一版应保留充分日志:
  - active connect 成功/失败
  - passive listen 成功/失败
  - 双候选仲裁结果
  - 选中 active 还是 passive

## 验证计划

1. 双方锥形:应仍走官方原链路,日志显示发起方 active / 被连方 passive 被选中。
2. 对称方发起 -> 锥形方应仍直连成功,不回归。
3. 锥形方发起 -> 对称方应使用反向链路(被连方 active -> 发起方 passive)直连成功。
4. 双方对称:应超时后回退 relay,不能卡死。
5. 与官方旧客户端互通:
   - 新发起方 -> 旧被连方:仍可走官方原链路。
   - 旧发起方 -> 新被连方:仍可走官方原链路。
