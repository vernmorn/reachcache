# ReachCache 风险修复实施计划

本文档对应 `PROJECT_RISKS.md`，基于当前工作区代码审查结果编写。它是实施计划，不是本轮代码修复记录。

## 审查结论

清单中的 44 个条目均能在当前工作区找到对应的生产代码缺陷、API 契约缺口或不安全默认行为，因此满足“逐项制定修复计划”的条件：

- P0：9 项，全部确认。
- P1：18 项，全部确认。
- P2：13 项，全部确认。
- P3：4 项，全部确认。

“确认”表示代码路径或接口行为确实存在风险，不表示每次运行都会立即触发。当前基线验证结果如下：

- `go test ./...`：通过。
- `go test -race ./...`：通过。
- `go vet ./...`：通过。
- `examples/single` 和 `examples/shortlink` 的 `go test ./...`：通过。

现有测试没有覆盖多数特定交错时序、服务发现恢复、恶意配置和安全默认场景，因此测试通过不构成风险不存在的证据。

两个条目需要保留审查边界：

- P2-03 的 `nil` 语义不一致已确认；负 TTL 在 LRU 和 LRU-2 中目前都被当作永不过期，因此“负 TTL 在两者之间不一致”这一子表述不成立，但负 TTL 被静默接受为永久缓存的语义风险成立。
- P2-09 的 `Get` 客户端确实没有检查响应 `Code`；当前内置 Server 在主要错误路径同时返回 gRPC error，所以该缺陷在内置 Server/Client 配对下通常不触发，但协议防御缺口仍然存在。

## 共同实施原则

1. 先为每个并发和协议缺陷增加可稳定复现的回归测试，再修改实现；涉及时序的测试使用 channel/barrier 控制交错，不依赖随机 sleep。
2. 所有用户回调、网络调用、etcd 调用和生命周期等待都不得在全局注册表锁、缓存外层锁或 Store 内部数据锁中执行。
3. 跨节点写入统一携带可比较的版本元数据和绝对过期时间；没有版本元数据的旧协议请求必须按明确的兼容策略处理，而不是静默覆盖新值。
4. 安全配置按失败关闭原则设计。开发用明文或无认证模式必须显式开启，不能由零值配置隐式启用。
5. 每个阶段完成后运行 `go test ./...`、`go test -race ./...`、`go vet ./...`，并为涉及的 nested module 单独运行测试。

# P0：阻断上线

## P0-01 哈希环实际上不包含本节点

现状证据：`peers.go` 的服务发现和全量拉取路径跳过 `selfAddr`，`PickPeer` 只有在环返回自身地址时才会报告 `self=true`。

实施计划：

1. 将本节点以“环成员”加入一致性哈希环，但不为本节点创建 gRPC `Client`；本地节点地址应统一使用实际 advertise 地址，避免监听地址和注册地址不一致。
2. 修改 `PickPeer`：命中自身时返回 `ok=true, self=true`，允许 `peer` 为 nil；远端节点仍返回已建立的 Client。
3. 保证服务发现的 PUT、DELETE、全量校准不会误删本节点，也不会为本节点创建或关闭远程连接。
4. 在两节点和三节点集成测试中验证本地归属直接调用本地 Getter，远端归属只发生一次 RPC，不形成 A→B→A 递归。

验收条件：自节点在环中有虚拟节点；`PickPeer` 能稳定返回 `self=true`；两节点 miss 不再互相递归；节点增删后本地和远程归属结果一致。

## P0-02 gRPC 默认明文且无认证

现状证据：`DefaultServerOptions` 未启用 TLS 和 Token，`NewClient` 默认使用 `insecure.NewCredentials()`，Server 直接监听传入 TCP 地址。

实施计划：

1. 增加明确的安全模式配置，生产模式要求服务端 TLS、客户端证书校验和节点认证 Token；缺少任一必需凭据时 `NewServer`、`NewClient` 和 `NewClientPicker` 返回错误。
2. 保留开发模式，但必须使用显式的 `WithInsecureDevelopmentMode` 之类选项，并在日志中明确标记不安全模式；默认监听地址改为回环地址或要求调用方显式提供暴露地址。
3. Token 认证要求传输安全，禁止在明文 gRPC 上发送 Token；校验 TLS ServerName、CA 和证书错误路径。
4. 增加未认证、错误 Token、TLS 证书不匹配、开发模式显式开启和正常节点间调用的集成测试。

验收条件：默认生产配置无法启动不安全服务；没有有效认证的 Get/Set/Delete 均被拒绝；只有显式开发配置才允许明文和空 Token。

## P0-03 LRU-2 完全不使用 `MaxBytes`

现状证据：`store.Options.MaxBytes` 只被 LRU 使用，`lru2Store` 没有字节计量字段，默认 LRU-2 只按桶和节点数量淘汰。

实施计划：

1. 为 LRU-2 增加统一的字节计量，至少计入 key 长度和 `Value.Len()`，并定义全局 `MaxBytes` 语义，而不是把预算错误地解释为每桶预算。
2. 在新增、更新、L1 晋升、L2 写入、逻辑删除和物理回收路径中原子地调整 used bytes；更新已有 key 时只计算新旧值差额。
3. 超出预算时从明确的冷数据端淘汰，直到预算满足；单个 value 超过预算时定义并测试“拒绝写入”或“写入后立即淘汰”的行为，不能静默绕过限制。
4. 将字节使用量和淘汰原因加入统计，并增加大 value、更新放大、跨 L1/L2 晋升和并发写入测试。

验收条件：LRU-2 的 `MaxBytes` 对默认和显式配置都有效；任何可观察时刻的有效缓存字节数遵守约定；超大 value 行为有明确错误或淘汰结果。

## P0-04 动态重平衡会死锁一致性哈希 Map

现状证据：`consistenthash.Map.rebalanceNodes` 持有 `m.mu.Lock()` 时调用再次获取同一把锁的 `m.Remove()`。

实施计划：

1. 抽出只允许在写锁内调用的 `removeNodeLocked`，由公开 `Remove` 获取锁后调用；重平衡路径只调用锁内版本。
2. 在重平衡前处理空环、空节点和平均负载为零的情况；所有副本数变更、计数重置和 keys 排序保持在同一写锁协议内。
3. 用可注入阈值或直接调用测试钩子稳定触发“请求数达到 1000 且负载失衡”的路径，验证重平衡完成后仍可 Get/Add/Remove。
4. 增加 goroutine 泄漏和超时测试，并用 `go test -race` 覆盖后台 balancer 与请求并发。

验收条件：重平衡在有限时间内返回；不会永久持有 Map 写锁；触发后所有公开路由操作仍可完成。

## P0-05 SingleFlight 注册过程存在竞态

现状证据：`singleflight/singleflight.go` 先 `Load` 再 `Store`，完成时无条件 `Delete`。

实施计划：

1. 使用 `LoadOrStore` 原子注册 call，只有实际存入 map 的调用方执行 fn，其余调用方等待同一个完成结果。
2. 清理时使用按实例校验的删除操作，例如 `CompareAndDelete(key, c)`；如果运行环境不支持，则在互斥锁保护下校验当前值后删除。
3. 将完成通知抽象成一次性 channel，确保值、错误和 panic 恢复结果在唤醒等待者前完全写入。
4. 增加两个首次并发注册、旧 call 清理期间新 call 注册、fn panic 和高并发同 key 测试。

验收条件：同一个 key 的 fn 在任意首次并发窗口最多执行一次；旧 call 不会删除后续 call；所有等待者都能收到同一结果或错误。

## P0-06 回源结果会覆盖并发写操作

现状证据：`Group.load` 在回源完成后无条件向 `mainCache` 写入，`Set`、`Delete` 和 `Clear` 没有版本或 epoch 校验。

实施计划：

1. 在 Group 中维护全局 cache epoch 和按 key 的 mutation version；Set/Delete 更新 key 版本，Clear 增加 epoch。
2. 回源开始时记录 `(epoch, keyVersion)`，回源完成后在写缓存前重新校验；版本变化时禁止旧结果写入。
3. 明确定义版本冲突时调用方的返回语义：优先读取并返回当前缓存值；若当前操作是删除或清空且没有新值，则返回 miss 或重新回源，不能把旧结果作为当前缓存结果写回。
4. Group 关闭时停止新的回源写入，并为 Set→load、Delete→load、Clear→load 和同名 Group 替换增加 barrier 测试。

验收条件：任何回源完成后，后续 Get 不会看到被并发 Delete/Clear 删除的数据；并发 Set 的新值不会被旧回源覆盖；版本冲突可观测且无数据竞争。

## P0-07 异步节点同步导致数据不一致和乱序覆盖

现状证据：`Group.Set` 和 `Delete` 每次独立启动 goroutine，只向当前哈希目标发送不带版本的操作，远端没有新旧操作比较。

实施计划：

1. 为每个 Group/key 生成单调逻辑版本，Set、Delete 都生成操作记录；Delete 必须携带墓碑版本，防止旧 Set 在删除后重新出现。
2. 扩展 peer RPC 请求携带版本、操作类型和绝对过期时间；远端只应用版本高于本地记录的操作，并持久化或至少在缓存生命周期内保留墓碑。
3. 对同一 key 使用串行发送队列；队列应与有界 worker 机制共用，确保 Set(v1)、Set(v2)、Delete 按版本应用。
4. 明确副本模型：只同步归属节点还是广播失效；若其他节点允许本地副本，写入和删除必须发送失效/版本信息，不能依赖 TTL 偶然收敛。
5. 增加延迟、乱序、重复投递、节点重启和多写者冲突测试，并记录最终一致性边界。

验收条件：旧 Set 不能覆盖新 Set；旧 Set 不能在 Delete 后复活；重复和乱序消息最终收敛到最高版本；同步失败不会无限积压。

## P0-08 LRU-2 更新 key 后可能返回旧值

现状证据：key 晋升到 L2 后再次 `Set` 只更新 L1，旧值仍可能留在 L2；L1 淘汰后 L2 可返回旧值。

实施计划：

1. 优先为 LRU-2 建立统一 key 索引，保证同一个 key 在 L1/L2 只有一个有效版本；不能立即重构时，Set 更新必须同步更新或删除 L2 副本。
2. L1→L2 晋升、Set、Delete、过期和容量复用都校验版本，禁止旧层级覆盖新层级。
3. 修正 Len 和 OnEvicted，使同 key 双层残留不会重复计数或重复回调。
4. 增加“访问晋升→更新→淘汰 L1→读取”“更新与晋升并发”“删除后更新”测试。

验收条件：任意时刻 Get 返回该 key 的最新有效值；L1 淘汰不会暴露旧 L2 值；每个逻辑条目最多产生一次删除/淘汰回调。

## P0-09 LRU 过期 Get 可能删除新值

现状证据：`lruCache.Get` 释放读锁后按 key 调用 `Delete`，未验证期间发现的链表元素仍是当前元素。

实施计划：

1. 过期 Get 记录原始 `*list.Element` 和过期时间；获取写锁后仅当 `items[key]` 仍指向同一 element 且过期时间未被更新时才删除。
2. 抽出 `removeExpiredIfUnchanged`，让删除、usedBytes 更新和回调事件收集在同一写锁内完成。
3. 增加过期 Get 与 Set、过期 Get 与 Delete、过期 Get 与更新 TTL 的确定性交错测试，并运行 race detector。

验收条件：旧 Get 不能删除并发写入的新值；更新 TTL 后旧过期判断无效；usedBytes、Len 和淘汰回调保持一致。

# P1：高优先级

## P1-01 `OnEvicted` 在锁内执行，容易死锁

现状证据：LRU 的 `removeElement` 和 `Clear` 在 Store 锁内调用回调；LRU-2 即使释放桶锁后回调，仍可能被 Cache 外层锁包住。

实施计划：

1. 将淘汰操作拆为“锁内修改并收集事件”和“完全解锁后分发事件”两步；Store 锁和 Cache 外层锁都必须在分发前释放。
2. 统一覆盖 Set、更新淘汰、Delete、Clear、过期清理、Close 清理等路径；规定回调执行顺序和回调 panic 隔离策略。
3. 增加回调中重入 Get/Set/Delete/Clear/Close、慢回调和回调 panic 测试；使用超时断言没有死锁。

验收条件：回调可安全重入缓存和 Group API；慢回调不阻塞缓存内部锁；回调事件不丢失、不重复且顺序符合文档。

## P1-02 `GetWithExpiration` 在读锁下修改链表

现状证据：`store/lru.go:GetWithExpiration` 持有 `RLock` 时调用 `list.MoveToBack`。

实施计划：

1. 将该方法改为写锁，或采用与 Get 相同的两阶段策略并在写阶段重新确认 element 身份；优先选择更容易证明正确的写锁方案。
2. 统一过期删除和 LRU 位置更新的锁协议，禁止任何读锁下修改 list、items 或 expires。
3. 增加多个 GetWithExpiration 与 Get/Set/Delete 并发测试，并用 `go test -race` 验证。

验收条件：所有链表写操作都在写锁内；并发读取不会破坏链表结构、items 映射或淘汰顺序。

## P1-03 LRU 更新已有 key 后不执行容量淘汰

现状证据：`lruCache.SetWithExpiration` 更新已有 key 后直接返回，未执行 `evict()`。

实施计划：

1. 更新值、过期时间和 usedBytes 后统一执行过期清理及容量淘汰，不区分新 key 和旧 key。
2. 处理更新本身导致超限、更新后变小、更新为过期值和更新触发回调的情况。
3. 增加更新大 value 超过预算的测试，验证淘汰顺序、usedBytes 和回调结果。

验收条件：每次成功更新完成后都满足 `usedBytes <= maxBytes`（按约定处理单值超限）；不需要等待下一次插入或 ticker 才收敛。

## P1-04 LRU-2 过期和逻辑删除会长期保留对象

现状证据：LRU-2 的 `del` 只将 `expireAt` 设为 0，保留 hmap、key、value 和节点；Len/清理依赖惰性路径。

实施计划：

1. 区分逻辑删除标记和可复用空闲节点；删除或过期时从 hmap 移除 key，清空 key/value 引用，并正确维护链表和 free list。
2. 过期 Get、后台清理、Delete、Clear 和容量复用都走统一的物理回收流程；回调只对有效旧值触发一次。
3. 更新 Len、字节统计和 L1/L2 索引，使已过期项目不会被计入有效条目。
4. 增加大对象 Delete/Clear 后 GC 可回收、TTL 后 Len、无后续写入的过期项和双层残留测试。

验收条件：删除或过期对象不再长期持有 key/value 引用；Len、统计和回调符合有效条目语义；Clear 后内部索引为空。

## P1-05 远端 Get 不继承 Context，也没有转发跳数限制

现状证据：`Peer.Get` 不接收 Context，`Client.Get` 创建新的 Background context 和固定 3 秒超时，Server Get 没有转发元数据。

实施计划：

1. 将 `Peer.Get` 改为接收调用方 Context，`Group.getFromPeer` 直接传入上游 deadline/cancellation；Client 仅在没有 deadline 时补充最大调用超时。
2. 在请求 metadata 或协议字段中加入 hop count、request ID 和必要的 origin 信息；每次转发递增 hop，超过上限直接失败。
3. 让服务端拒绝超过 hop 上限或重复 request ID 的递归请求，并保留错误原因和 trace 信息。
4. 增加上游取消传播、短 deadline、A/B 循环拓扑和 hop 上限测试。

验收条件：上游取消后远端 RPC 尽快结束；循环路由在有限跳数内失败；单次请求不会产生无限递归链。

## P1-06 etcd 服务前缀匹配过宽

现状证据：`peers.go` 的 Watch 和 Get 使用 `/services/` 加服务名但没有尾部 `/`，会匹配具有前缀关系的服务名。

实施计划：

1. 统一服务 key 格式为 `/services/{validated-service-name}/{address}`，查询和 Watch 前缀统一追加 `/`。
2. 校验服务名不为空且不包含 `/`、控制字符或路径穿越语义；集中封装 key 构造和解析逻辑。
3. 对旧 key 是否迁移做版本化处理，避免上线期间同时把旧服务错误加入环。
4. 增加 `cache` 与 `cache-admin`、相同地址不同服务和 PUT/DELETE 事件测试。

验收条件：一个服务只能发现自身前缀下的节点；全量查询、Watch、注册和删除使用同一 key 规范。

## P1-07 服务发现存在事件丢失且 Watch 失败后不恢复

现状证据：全量 Get 的 revision 未用于启动 Watch；Watch 关闭或出错后直接退出，没有重新全量同步。

实施计划：

1. 使用全量响应的 `Header.Revision + 1` 作为 Watch 起点，避免全量读取和 Watch 建立之间遗漏事件。
2. 封装可恢复的 discovery loop：Watch 错误、channel 关闭、compacted revision 或连接故障时关闭旧 watcher，退避后重新全量拉取并建立新 Watch。
3. 全量校准时对比当前 clients，补齐新增节点并关闭删除节点；处理重复 PUT 的幂等性。
4. 增加事件窗口、revision 压缩、channel 关闭、短暂 etcd 断连和恢复后的拓扑校准测试。

验收条件：Watch 发生任何可恢复故障后拓扑能自动收敛；不会永久连接已下线节点；重连过程不会丢失节点变更。

## P1-08 服务注册失败不会让 Server 启动失败

现状证据：`Server.Start` 在后台 goroutine 调用 `registry.Register`，失败只记录日志，gRPC 仍继续 Serve。

实施计划：

1. 将注册结果纳入启动状态：注册成功后才报告 ready/开始对外服务，或者让 `Start` 返回注册错误并关闭已创建 listener。
2. 分离“监听成功”和“集群就绪”状态，使用 gRPC health/readiness 明确反映注册状态；未注册节点不能宣称 serving。
3. 将注册成功、租约创建失败、Put 失败和 etcd 超时纳入启动测试。

验收条件：etcd 注册失败时调用方能获得明确启动失败或 not-ready 状态；不会出现端口可访问但节点不可发现的假可用。

## P1-09 Server 停止时不等待注册注销

现状证据：`registry.Register` 内部创建并持有独立 etcd Client，`Server.Stop` 关闭 `stopCh` 后不等待 Revoke goroutine 完成。

实施计划：

1. 让 Register 返回可关闭、可等待的 registration handle，或接受 `context.Context` 并提供完成 channel；统一 etcd Client 所有权。
2. 将 `stopCh` 改成明确的 `chan struct{}` 或 context cancel，Stop 只触发一次，等待 Revoke 和注册 goroutine 完成后再关闭共享资源。
3. 为 Revoke 超时定义兜底行为并记录结果；不能在注册 goroutine 仍使用 Client 时关闭该 Client。
4. 增加优雅停止、快速停止、重复 Stop、Revoke 超时和租约 key 消失时间测试。

验收条件：Stop 返回前已完成注销等待或明确记录不可完成原因；停止后不会因旧租约继续向该节点路由。

## P1-10 etcd 连接无法配置 TLS 和认证

现状证据：Server、ClientPicker 和 registry 只设置 etcd Endpoints/DialTimeout，没有 TLS、用户名、密码或认证配置。

实施计划：

1. 扩展独立的 etcd 安全配置，支持 CA、客户端证书/私钥、ServerName、是否跳过校验、用户名和密码，并禁止复用 gRPC TLS 配置中的隐含假设。
2. 将配置传递到 Server 注册、ClientPicker Watch、Client 创建和 registry Register 的每一条 etcd 连接路径。
3. 对证书缺失、证书不匹配、认证失败和明文开发模式做显式错误与日志脱敏。
4. 使用 TLS etcd 测试实例或可替换 client 工厂验证加密和认证配置确实生效。

验收条件：生产 etcd 连接可强制 TLS 和认证；任何未配置安全参数的远程 etcd 连接不会被误认为安全。

## P1-11 异步 Set/Delete 会无限制创建 goroutine

现状证据：每次 Set/Delete 直接启动 `go g.syncToPeers`，没有队列、worker、并发上限或背压。

实施计划：

1. 为 Group 建立有界同步队列和固定数量 worker，队列项包含 key、版本、操作、value、expiry 和重试信息。
2. 对同 key 保证串行，对不同 key 使用有限并行；队列满时定义阻塞、拒绝、丢弃或降级策略，并将结果暴露给监控。
3. 为远端故障增加有限重试、指数退避、最大重试次数和死信/丢弃统计；不能创建一个 goroutine 等待一个故障 RPC。
4. 将队列纳入 Group Close，停止接收新任务并等待或取消已有 worker。
5. 增加高写入、慢远端、队列满、重试上限和 goroutine 数量上限测试。

验收条件：并发 goroutine 数和待发送数据量有上限；远端故障下系统有背压或明确丢弃；同 key 操作顺序不被 worker 调度破坏。

## P1-12 Group 销毁不会等待进行中的任务

现状证据：Group 没有生命周期 Context/WaitGroup，`close` 只设置 closed、清理缓存并关闭底层 Cache。

实施计划：

1. 为 Group 增加生命周期 context、cancel、WaitGroup 和关闭状态机；每个 Getter、peer RPC、同步任务在启动时登记。
2. Close 先阻止新任务，再 cancel 生命周期 context，等待回源、同步 worker 和 peer 请求退出，最后清理 Cache。
3. 为同名替换增加 Group generation/instance ID，防止旧任务向新 Group 或远端同名组写入旧数据。
4. 定义等待超时和强制关闭策略，确保进程退出不会无限等待不可控 Getter。
5. 增加 DestroyGroup 与回源、Set/Delete 同时发生、同名重建和 Close 超时测试。

验收条件：DestroyGroup 返回后旧任务全部停止或被明确取消；旧 Group 不会写入新 Group；网络和 goroutine 资源最终释放。

## P1-13 全局注册表锁覆盖 Group 关闭和用户回调

现状证据：`NewGroup`、`DestroyGroup` 和 `DestroyAllGroups` 持有 `groupsMu` 调用 `g.close`，而关闭会 Clear Cache 并可能执行回调。

实施计划：

1. 在 `groupsMu` 内只完成摘除、替换或快照；保存待关闭 Group 后立即释放锁，再执行 close 和用户回调。
2. NewGroup 同名替换先把新 Group 放入注册表，再在锁外关闭旧 Group；Destroy 操作先删除后关闭；DestroyAll 先复制并清空 map。
3. 配合 P1-12，保证摘除后旧 Group 仍可安全完成生命周期关闭但不能重新注册自己。
4. 增加回调中 GetGroup/NewGroup、并发 List/Get/Destroy 和大缓存关闭耗时测试。

验收条件：回调重入注册表不会死锁；Group 关闭耗时不阻塞所有注册表读写；替换和销毁的可见性顺序有文档和测试保证。

## P1-14 `from_peer` Context Key 可被业务调用方伪造

现状证据：Group 和 Server 使用字符串 key `"from_peer"`，任意嵌入式调用方可以构造同名 context value。

实施计划：

1. 使用包级不可导出的强类型 context key，避免外部包伪造；只在受认证的 peer RPC 入口设置该内部标记。
2. 不以 `Value(...) != nil` 判断布尔语义，使用私有类型和明确 `ok && value == true` 检查。
3. 将“是否来自 peer”和“是否允许传播”从业务 context 与传输认证中分离，避免业务调用方能绕过复制策略。
4. 增加外部同名字符串 key、false 值、未认证 RPC 和合法 peer RPC 测试。

验收条件：业务 context 无法抑制节点同步；只有经过认证且由 Server 设置的内部标记才能进入 peer 写入路径。

## P1-15 跨节点传播会重置 TTL

现状证据：同步只传 value，远端 Set 和 peer Get 回填都以当前时间重新计算 `g.expiration`。

实施计划：

1. 将绝对过期时间或剩余 TTL 加入 Set、Get response 和内部 peer 结果；0 明确表示永不过期。
2. 本地写入时生成唯一 expiry；远端应用和本地回填使用收到的 expiry，不再使用接收时间重算。
3. 处理网络延迟导致 expiry 已过期的消息：直接丢弃并执行必要的删除/墓碑处理；同时防止负剩余 TTL 被解释为永久缓存。
4. 增加多跳传播、延迟传播、远端回填、无限 TTL 和过期消息测试。

验收条件：数据的总生存时间不因节点传播次数增加；远端副本与源数据共享同一过期语义。

## P1-16 未命中结果没有负缓存

现状证据：`loadData` 的 Getter 错误直接返回，`Group.load` 不缓存“确实不存在”的结果；SingleFlight 只合并同一时刻请求。

实施计划：

1. 引入明确的 typed not-found 错误或 Getter 结果类型，区分 key 不存在、后端暂时故障和参数错误。
2. 增加可配置的负缓存 TTL 和内部 not-found sentinel；负缓存命中返回稳定的 not-found 错误，不把 sentinel 暴露为正常 ByteView。
3. 后端故障、超时和取消不得写入负缓存；对空 key 和异常 key 继续做校验、限流或请求保护。
4. 增加不存在 key 的并发、负 TTL 到期、后端故障不缓存和负缓存清理测试。

验收条件：短时间内重复不存在 key 不重复打穿后端；真实后端故障不会被负缓存掩盖；负缓存可按 Group 配置和监控。

## P1-17 一致性哈希重复节点和哈希碰撞会破坏环

现状证据：`Map.Add` 未做节点幂等检查，`hashMap[int]string` 会覆盖碰撞，Remove 后可能留下 keys 与映射不一致。

实施计划：

1. Add 对已有节点做幂等处理，或先执行受保护的完整替换；nodeReplicas、keys 和 nodeCounts 必须保持一致。
2. 将环 token 从单一 `hash -> node` 改为带真实节点/副本序号的碰撞安全结构，例如排序后的 token 记录；Remove 只删除自己的 token。
3. 提升哈希空间或支持稳定的 tie-breaker，但不能假设 CRC32 不碰撞；Get 对空映射和 nil counter 做防御性检查并返回 miss/错误而不是 panic。
4. 增加重复 Add/Remove、强制哈希碰撞、碰撞节点增删和并发 Get 测试。

验收条件：重复添加不会复制环；哈希碰撞不会覆盖其他节点；删除后不存在悬空 token；任何坏环状态都不会导致 panic。

## P1-18 重平衡路径混用原子和非原子读写

现状证据：请求路径原子递增 `totalRequests`，但 `checkAndRebalance` 和 `rebalanceNodes` 存在直接读取。

实施计划：

1. 所有 totalRequests 读取、递增和重置统一使用 atomic；nodeCounts 继续使用 atomic 或在同一锁协议下读写。
2. 修正平均负载计算，保证节点数量和请求计数来自可接受的一致采样；避免除零和重平衡期间计数丢失。
3. 用 race detector 和高并发请求持续触发后台 balancer；检查负载统计和副本变更没有数据竞争。

验收条件：`go test -race` 无报告；所有访问遵守同一原子/锁协议；重平衡不会因非原子读取产生错误阈值或丢计数。

# P2：中优先级

## P2-01 多个 Close API 不幂等

现状证据：LRU、consistenthash Map、ClientPicker 和 Server 直接关闭 channel，没有统一的 `sync.Once`/状态保护；其他部分实现已有幂等语义。

实施计划：

1. 为每个拥有 channel、ticker、watcher、连接或注册句柄的组件增加统一 Close 状态和 `sync.Once`。
2. Close 内部按照 cancel→停止后台任务→等待→关闭连接的顺序执行；重复和并发调用返回同一结果，不重复关闭资源。
3. 补充 LRU、Map、Picker、Server、Client 和 Group 的重复/并发 Close 测试，并检查关闭后的 API 语义。

验收条件：任意公开 Close 可重复、可并发调用而不 panic；资源只关闭一次；关闭后的调用不会重新启动后台资源。

## P2-02 `WithCacheOptions` 会覆盖 `cacheBytes`

现状证据：`NewGroup` 先把 `cacheBytes` 写入 `MaxBytes`，`WithCacheOptions` 随后直接用 `NewCache(opts)` 替换整个缓存配置。

实施计划：

1. 统一配置来源并明确优先级：建议 `NewGroup` 的 `cacheBytes` 始终是 Group 字节预算，`WithCacheOptions` 只覆盖其余字段；若允许覆盖预算，提供显式的 `WithMaxBytes`。
2. 将选项改为字段级合并，不再整体替换 `mainCache`；保留未指定字段的默认值和已计算的预算。
3. 增加只设置 CacheType、只设置清理间隔、显式覆盖预算和默认 LRU-2 的配置测试。

验收条件：使用 `WithCacheOptions` 不会意外把 MaxBytes 变成 0；所有配置项的优先级在 API 文档中唯一且可测试。

## P2-03 LRU 和 LRU-2 的 nil、负 TTL 语义不一致

现状证据：LRU 对 nil value 调用 Delete，LRU-2 可保存 nil value；两者都把负 TTL 静默解释为永不过期。

实施计划：

1. 统一 Store 契约，建议 nil value 直接返回明确的 invalid-value 错误，不把 nil 隐式解释为删除；删除只通过 Delete API 表达。
2. 规定只有 `expiration == 0` 表示永不过期，`expiration < 0` 返回错误；所有 Store、Cache 和 Group 入口执行相同校验。
3. 统一过期计算、sentinel 和错误类型，并增加 LRU/LRU-2 对照测试；同步修订注释和 README。

验收条件：同一个输入在两种 Store 下行为一致；负 TTL 不会产生永久脏数据；nil 不会成为可命中的正常缓存值。

## P2-04 LRU-2 和一致性哈希配置缺少边界校验

现状证据：LRU-2 使用 uint16 计算 `cap+1`/`mask+1`，一致性哈希配置可传入 nil HashFunc、0 副本或非法范围而不在构造期报错。

实施计划：

1. 在构造期校验 BucketCount、CapPerBucket、Level2Cap 不会导致 uint16 溢出、空数组或索引超界；内部计算改用 int，并限制最大可接受容量。
2. 校验 HashFunc 非 nil、DefaultReplicas/MinReplicas/MaxReplicas 为正且范围有序，阈值为有限值并处于允许区间。
3. 让构造 API 返回配置错误，或提供 checked constructor 并让 Group/Cache 传播错误；禁止构造成功后首次读写才 panic。
4. 增加极端 uint16、空哈希、零副本、反向范围、NaN/Inf 阈值和普通合法配置测试。

验收条件：非法配置在构造阶段失败并说明原因；合法最大配置不会溢出；首次 Get/Set 不会因构造参数导致 panic。

## P2-05 Store 的 Len 和统计 size 可能包含过期数据

现状证据：LRU `Len` 直接返回 list 长度，LRU-2 `walk` 只检查逻辑删除标记，不比较当前时间；Cache/Group Stats 直接使用 Len。

实施计划：

1. 定义 Len 是“物理条目数”还是“当前有效条目数”；建议统一为当前有效条目数，并在锁内过滤/回收已过期项。
2. 让 LRU 和 LRU-2 的 Len、Stats、清理路径共享过期判断和删除逻辑；避免 Len 为统计而执行用户回调时持锁。
3. 增加 TTL 到期后立即 Len、后台清理前 Stats、并发过期和过期回收测试。

验收条件：文档定义与两个 Store 一致；过期条目不计入有效 size；统计读取不会观察到明显违反定义的过期项目。

## P2-06 SingleFlight 等待者无法响应自己的 Context 取消

现状证据：等待者直接调用 `WaitGroup.Wait()`，SingleFlight API 没有 Context 或完成 channel。

实施计划：

1. 增加 `DoContext(ctx, key, fn)`，call 使用完成 channel；等待者用 `select` 同时等待完成和自己的 ctx.Done。
2. 保留 leader 与 waiter 的错误语义：waiter 取消只影响自身，不取消仍被其他请求使用的 leader；Group 回源 context 需定义 leader 取消时的处理。
3. 让 Group.Get 将请求 context 传给等待路径，增加 waiter 取消、leader 慢回源和 leader 失败后的重试测试。

验收条件：等待者取消后在 deadline 内返回 context error；不会遗留等待 goroutine；leader 结果完成后其他等待者仍能正常收到结果。

## P2-07 Peer 返回的 byte slice 未复制

现状证据：`group.go:getFromPeer` 直接用 Peer 返回的 slice 构造 ByteView，随后可能写入本地缓存。

实施计划：

1. 在进入 ByteView 前始终复制 Peer 返回的数据；同时检查 nil/空值及所有权约定。
2. 在 Peer 接口文档中明确返回 slice 的所有权，内部实现也不要复用会被异步修改的 buffer。
3. 增加自定义 Peer 返回可复用 buffer、返回后立即修改和并发修改测试，并运行 race detector。

验收条件：Peer 后续修改原 slice 不会改变本地缓存；跨节点回填与 Getter/Set 路径具有相同的复制语义。

## P2-08 ByteView 暴露可修改内部数据的 `Bytes`

现状证据：`ByteView.Bytes` 直接返回内部 `b`，只通过注释要求调用方不要修改。

实施计划：

1. 将公开 `Bytes` 改为返回副本，或将其改名为明确的安全 API；零拷贝访问只保留给包内私有 helper。
2. 更新 Server、示例和序列化调用点，统一使用复制后的只读数据；评估额外复制成本并在必要处提供受限内部接口。
3. 增加修改返回 slice 后缓存值不变、并发读写 race 和空 ByteView 测试。

验收条件：外部调用方无法通过公开 API修改缓存内部数组；“只读视图”注释、实现和测试一致。

## P2-09 客户端 Get 没有检查协议响应 Code

现状证据：`Client.Get` 在 gRPC 无 error 时直接返回 `resp.Value`，没有检查 `resp.Code`；Set/Delete 已检查。

实施计划：

1. Get 与 Set/Delete 统一检查响应 Code，并将非零 Code 转为稳定的 typed error 或标准 gRPC status；未知 Code 也必须失败。
2. 同步修订 Server，使协议 Code 和 gRPC status 只表达一套明确语义，避免 response/error 双重且相互矛盾。
3. 增加伪造非零 Code 且 gRPC error 为 nil 的 Client 单测，以及内置 Server 的 group-not-found、key-not-found、backend-error 测试。

验收条件：任何非零 Get Code 都不会被当成成功空值；客户端错误可被调用方可靠区分和处理。

## P2-10 统计数据不是一致快照

现状证据：Cache Stats 分别读取 initialized、closed、hits、misses 和 Len；Group.load 为每个等待者递增 loads，统计字段来自不同时间点。

实施计划：

1. 定义统计口径，至少区分请求数、实际 loader 执行数、SingleFlight 等待数、命中数和加载耗时。
2. 为 Cache/Group 提供一致快照方法：在统一锁或 epoch 方案下读取状态和 size，原子计数使用同一采样规则。
3. 只让 leader 计入实际加载次数；等待者单独计入 waiters，并明确平均耗时的分母。
4. 增加 Stats 与 Close/Clear/读写并发、SingleFlight 多等待者和快照内部一致性测试。

验收条件：指标名称和分母有文档；一次快照不会混合明显不一致的生命周期状态；监控可以区分请求压力和后端加载压力。

## P2-11 Stats HTTP 接口无认证、无 TLS 和请求超时

现状证据：Stats HTTP Server 只设置 Addr/Handler，无认证、TLS 或 Read/Write/Idle timeout，`:port` 会监听所有接口。

实施计划：

1. 默认只绑定回环或管理网地址；暴露到其他网络时要求显式启用认证和 TLS，并支持独立的 stats 凭据。
2. 配置 `ReadHeaderTimeout`、`ReadTimeout`、`WriteTimeout`、`IdleTimeout` 和合理的请求/响应限制。
3. 对 `/stats` 和 `/stats/all` 做最小权限校验，避免未授权暴露 Group 名称、节点地址和内部指标。
4. 增加本机默认绑定、未授权、错误凭据、TLS、慢请求和 Stop 超时测试。

验收条件：Stats 默认不向公网暴露；没有凭据不能读取统计；慢连接不会无限占用 HTTP 资源。

## P2-12 NewServer 部分初始化失败时泄漏 etcd Client

现状证据：`NewServer` 创建 etcd Client 后加载 TLS 失败直接返回，没有关闭已创建的 Client。

实施计划：

1. etcd Client 创建成功后立即登记清理责任，在所有后续失败路径执行 Close；成功构造后再转移所有权给 Server。
2. 统一 NewServer 的 listener、TLS、gRPC 和 etcd 初始化错误清理顺序，避免遗漏未来新增资源。
3. 增加无效证书、证书路径不存在、gRPC 初始化失败和重复构造失败测试；必要时通过 client 工厂检查 Close 被调用。

验收条件：任一部分初始化失败都释放已创建资源；重复失败构造不会累积 etcd 连接或后台 goroutine。

## P2-13 全局默认配置使用可变指针

现状证据：`DefaultServerOptions`、`registry.DefaultConfig` 和 `consistenthash.DefaultConfig` 是导出的可变指针，Map 还直接保存 Config 指针。

实施计划：

1. 将默认配置改为返回值函数或不可变内部值，调用方获取的是副本；Endpoints 等 slice 也必须深复制。
2. 构造 Map、Server、Picker 和 registry client 时复制完整配置，不保存调用方可变指针。
3. 对旧导出符号做版本化处理，明确迁移方式；禁止运行中修改已构造实例配置，动态配置必须使用受锁保护的专用 API。
4. 增加修改默认配置不影响既有实例、并发构造/读取和 race detector 测试。

验收条件：一个实例的配置不会被另一个实例或全局变量修改；默认配置读取无数据竞争；slice 不发生外部别名污染。

# P3：低优先级和 API 改进

## P3-01 外部包无法方便构造 ByteView

现状证据：`ByteView.b` 未导出，没有构造函数，而导出的 `Cache.Add` 要求调用方提供 ByteView。

实施计划：

1. 增加 `NewByteView([]byte) ByteView`，构造时复制输入；空 slice 的行为与 Group/Store 契约统一。
2. 在公共文档中说明 ByteView 的只读和复制语义，并给出外部包直接使用 Cache 的示例。
3. 增加构造后修改输入不影响 ByteView、Cache.Add 能写入非空值和空输入测试。

验收条件：外部包可以不依赖未导出字段构造安全的非空 ByteView；构造函数不会引入底层 slice 别名。

## P3-02 错误类型和协议错误码不统一

现状证据：Server 对多种 Get 错误复用 Code 2，Set/Delete 多数错误复用 Code 3，同时返回 response 和 gRPC error；Client 不能稳定区分错误。

实施计划：

1. 建立错误分类表：参数错误、Group 不存在、key 不存在、后端失败、关闭状态、认证失败分别映射到标准 gRPC codes。
2. 优先使用标准 gRPC status 作为传输错误；若保留 Code 字段，限定其为兼容展示字段并保证与 status 一致。
3. 提供客户端 typed error/helper，统一处理 Get/Set/Delete，删除“返回空值即成功”的歧义。
4. 更新 proto 注释、Server、Client、README 和各示例，增加每类错误的协议兼容测试。

验收条件：调用方能可靠区分 not found、参数错误和后端故障；response Code、Msg 和 gRPC status 不互相矛盾。

## P3-03 Protobuf `go_package` 使用相对路径

现状证据：`proto/reachcache.proto` 使用 `option go_package = "./"`，已生成代码包名为 `__`。

实施计划：

1. 将 `go_package` 改为稳定完整路径，例如 `github.com/vernmorn/reachcache/proto;proto`，并固定 protoc/protoc-gen-go 版本。
2. 使用 source-relative 规则重新生成 `reachcache.pb.go` 和 `reachcache_grpc.pb.go`，检查包名、导入路径和 generated header。
3. 更新根模块、示例和下游引用，禁止手工修改生成文件；在 CI 中加入 proto regeneration diff 检查。
4. 增加独立模块导入 proto、重新生成后编译和 gRPC 客户端/服务端互操作测试。

验收条件：外部模块可用稳定 import path 导入 proto；重新生成不会产生异常包名或无关路径变化。

## P3-04 示例代码包含不安全或不严谨实现

现状证据：shortlink 示例硬编码 HMAC key、手工拼接 JSON、允许 GET 执行写操作、默认无认证监听并忽略多处数据库错误；single 示例也忽略部分 DB、JSON 和缓存错误。

实施计划：

1. 将短链接密钥改为必需环境变量或启动参数，缺失时拒绝启动；README 明确示例仅供本地演示，不能直接公网部署。
2. 全部 JSON 响应使用 `encoding/json`，只允许 POST 创建短链接，加入 URL/路径输入校验、请求大小限制和 HTTP 超时。
3. 处理所有数据库、事务、缓存 Set、JSON marshal 和 HTTP 写入错误；区分 `sql.ErrNoRows` 与真实数据库故障。
4. 示例默认绑定回环地址，必要时提供显式认证配置；补充安全响应头、优雅关闭和日志脱敏。
5. 为 shortcode、JSON 特殊字符、数据库失败、重复码、未授权请求和超时增加示例测试，并从发布产物中排除不必要的二进制和本地数据库文件。

验收条件：示例不包含可复用密钥；响应始终是合法 JSON；写操作不会通过 GET 触发；数据库和缓存失败不会被静默忽略；README 明确安全边界。

## 建议实施批次

1. 第一批修复 P0-01 至 P0-09，并先完成安全默认、环路、内存预算、锁死和并发一致性回归测试。
2. 第二批修复 P1-01 至 P1-10，完成回调锁、服务发现、注册生命周期、Context 传播和 etcd 安全配置。
3. 第三批修复 P1-11 至 P1-18，完成有界同步队列、Group 生命周期、版本/TTL 传播、哈希环完整性和原子访问统一。
4. 第四批修复 P2-01 至 P2-13，完成关闭、配置、Store 语义、统计、API 数据所有权和管理接口治理。
5. 第五批修复 P3-01 至 P3-04，完成公共 API、协议错误语义、Protobuf 生成配置和示例安全整理。

每批次的完成标准是：对应回归测试通过，`go test -race ./...` 和 `go vet ./...` 通过，公开 API/协议变化已更新文档，并且没有把尚未修复的高优先级风险标记为已解决。
