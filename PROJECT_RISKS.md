# ReachCache 问题清单

本文档整理当前工作区对 ReachCache 生产代码进行审查后发现的问题。

审查范围不包含测试覆盖率、测试缺失、测试设计和测试稳定性问题，仅记录可能影响正确性、安全性、可用性、资源管理和 API 语义的生产风险。

## 总体结论

当前项目不适合直接用于生产级分布式部署。最严重的问题集中在：

- 分布式哈希环没有正确包含本节点，可能造成请求递归和错误路由。
- 默认 gRPC 服务为明文且无认证，端口暴露后可被任意读写缓存。
- 默认 LRU-2 不执行 `MaxBytes` 限制，存在内存失控风险。
- 动态重平衡路径存在确定性死锁。
- SingleFlight、回源回写和异步节点同步存在并发一致性问题。

## 优先级定义

- **P0：阻断上线**。可能造成数据错误、服务冻结、内存耗尽或严重安全事件，应优先修复。
- **P1：高优先级**。在正常生产负载或故障场景下可能造成明显功能异常，应尽快修复。
- **P2：中优先级**。会造成资源、运维或 API 语义风险，应纳入近期治理计划。
- **P3：低优先级**。不直接阻断核心功能，但会影响可维护性、易用性或示例质量。

## P0：阻断上线

### P0-01 哈希环实际上不包含本节点

- **位置**：`peers.go:193-197`、`peers.go:227-230`、`peers.go:263-278`
- **触发条件**：使用 `ClientPicker` 进行真实分布式部署。
- **问题**：服务发现时跳过 `selfAddr`，一致性哈希环只包含远端节点，因此 `self=true` 在实际运行中基本不会出现。
- **影响**：本节点没有真正的 key 归属；两节点场景下，A 的 miss 可能路由到 B，B 又路由回 A，形成递归 gRPC 请求和超时。不同节点的 key 归属也不一致。
- **建议**：将本节点加入哈希环，但不创建本地 gRPC Client；命中本节点时由 `PickPeer` 返回 `self=true`，由 `Group` 直接执行本地 Getter。

### P0-02 gRPC 默认明文且无认证

- **位置**：`server.go:68-73`、`server.go:141-156`、`client.go:101-111`
- **触发条件**：未显式配置 TLS 和 Token，且 gRPC 端口可被其他主机访问。
- **问题**：默认没有 TLS，也没有 Token 认证。
- **影响**：任何可访问端口的客户端都可以读取、覆盖或删除缓存，造成数据泄露、缓存投毒和批量删除。
- **建议**：生产模式强制 TLS 和认证；或者默认只监听回环地址，并提供显式的不安全开发模式。

### P0-03 LRU-2 完全不使用 `MaxBytes`

- **位置**：`cache.go:78-86`、`store/store.go:89-91`、`store/lru2.go:72-85`
- **触发条件**：使用默认 LRU-2，或显式设置 `CacheOptions.MaxBytes`。
- **问题**：`NewGroup` 的 `cacheBytes` 最终写入 `MaxBytes`，但 LRU-2 只按条目数限制，不读取该字段。
- **影响**：单个大 value 和总内存都不受字节预算控制，可能造成内存持续增长甚至 OOM。
- **建议**：为 LRU-2 增加字节计量和淘汰机制，或把 API 改为明确的 entry capacity，避免继续宣称字节上限。

### P0-04 动态重平衡会死锁一致性哈希 Map

- **位置**：`consistenthash/consistenthash.go:245-279`
- **触发条件**：总请求数达到 1000 且负载不均衡超过阈值，触发 `rebalanceNodes`。
- **问题**：`rebalanceNodes` 持有 `m.mu.Lock()` 时调用 `m.Remove()`，而 `Remove()` 会再次获取同一把锁。
- **影响**：后台重平衡 goroutine 永久阻塞，并阻塞后续路由操作。
- **建议**：拆分出只允许在锁内调用的 `removeNodeLocked`，禁止持锁函数调用公开加锁方法。

### P0-05 SingleFlight 注册过程存在竞态

- **位置**：`singleflight/singleflight.go:64-75`
- **触发条件**：两个 goroutine 同时对同一个 key 首次调用 `Do`。
- **问题**：`Load` 和 `Store` 不是原子操作，两个调用可能同时创建不同的 call。前一个 call 的 `Delete` 还可能删除后一个仍在执行的 call。
- **影响**：同一个 key 执行多次回源，无法保证请求合并，并可能造成结果覆盖和后端压力放大。
- **建议**：使用 `LoadOrStore` 原子注册；清理时校验当前 map 中仍是同一个 call。

### P0-06 回源结果会覆盖并发写操作

- **位置**：`group.go:245-277`、`group.go:313-337`
- **触发条件**：Getter 回源期间并发执行同 key 的 `Set`、`Delete` 或 `Clear`。
- **问题**：回源完成后无条件将旧结果写入缓存。
- **影响**：删除后 key 重新出现、更新值被旧值覆盖、`Clear` 返回后数据再次出现。
- **建议**：为 key 增加版本号或缓存 epoch；回源完成后仅当版本未变化时允许回写。

### P0-07 异步节点同步导致数据不一致和乱序覆盖

- **位置**：`group.go:206-239`、`group.go:244-265`、`group.go:379-411`
- **触发条件**：启用 peers，并在多个节点执行 Set/Delete 或存在高并发写入。
- **问题**：Set/Delete 只更新本地节点和一个哈希目标节点；每个操作还会单独启动 goroutine，操作顺序不受保证。
- **影响**：其他节点的本地副本在 TTL 内持续过期；`Set(v1)`、`Set(v2)`、`Delete` 可能乱序到达，产生旧写覆盖新写或删除后重新写入。
- **建议**：使用版本号、时间戳或墓碑记录，并对同一 key 串行同步；明确系统提供的最终一致性边界。

### P0-08 LRU-2 更新 key 后可能返回旧值

- **位置**：`store/lru2.go:174-196`、`store/lru2.go:380-400`
- **触发条件**：key 已从 L1 晋升到 L2，随后执行 `Set` 更新该 key，且新值之后从 L1 淘汰。
- **问题**：新的 `Set` 只写入 L1，旧值仍留在 L2。
- **影响**：后续 `Get` 可能返回 L2 中的旧值；同一个 key 还可能同时存在于 L1 和 L2，导致 `Len` 和淘汰回调语义错误。
- **建议**：更新时同时更新或失效 L2 副本，最好维护统一的 key 索引。

### P0-09 LRU 过期 Get 可能删除新值

- **位置**：`store/lru.go:91-106`
- **触发条件**：Get 发现旧值过期、释放读锁后，另一个 goroutine 对同 key 执行 Set。
- **问题**：旧 Get 随后按 key 执行 Delete，无法区分旧元素和新元素。
- **影响**：刚写入的新值被旧 Get 删除。
- **建议**：在写锁内按链表元素身份和原始过期时间执行条件删除，不能只按 key 删除。

## P1：高优先级

### P1-01 `OnEvicted` 在锁内执行，容易死锁

- **位置**：`store/lru.go:177-188`、`store/lru.go:225-243`、`cache.go:199-208`、`cache.go:228-238`
- **触发条件**：配置 `OnEvicted`，且回调执行较慢或重新访问缓存。
- **问题**：LRU 的回调在 Store 锁内执行；即使 LRU-2 将回调放到 Store 锁外，也仍可能被 Cache 外层锁包住。
- **影响**：回调调用 `Get/Set/Clear/Close` 时可能死锁，慢回调还会阻塞所有缓存操作。
- **建议**：锁内只收集回调参数，完全释放 Store 和 Cache 锁后执行用户回调。

### P1-02 `GetWithExpiration` 在读锁下修改链表

- **位置**：`store/lru.go:278-303`
- **触发条件**：并发调用 `GetWithExpiration`，或与 Get/Set/Delete 并发。
- **问题**：函数持有 `RLock`，却调用 `list.MoveToBack` 修改链表。
- **影响**：多个读者可以同时修改链表，可能造成数据竞争和链表结构损坏。
- **建议**：改用写锁，或采用与 `Get` 相同的两阶段锁策略。

### P1-03 LRU 更新已有 key 后不执行容量淘汰

- **位置**：`store/lru.go:156-163`
- **触发条件**：已有 key 被更新为更大的 value。
- **问题**：更新 `usedBytes` 后直接返回，不执行 `evict()`。
- **影响**：缓存可能长期超过 `maxBytes`，直到下一次插入或定时清理。
- **建议**：更新后统一执行容量检查和淘汰。

### P1-04 LRU-2 过期和逻辑删除会长期保留对象

- **位置**：`store/lru2.go:101-163`、`store/lru2.go:249-267`、`store/lru2.go:432-443`、`store/lru2.go:479-489`
- **触发条件**：LRU-2 中的 key 过期、被 Delete、被 Clear，或长期没有后续写入。
- **问题**：部分路径只返回 miss，部分路径只把 `expireAt` 设为 0，key、value 和 map 条目仍然保留。
- **影响**：Len 统计不准确，大对象无法及时回收，`OnEvicted` 可能不触发，Clear 后仍可能残留内部对象。
- **建议**：过期访问时在桶锁内完成删除；删除时清理 key/value 引用并维护空闲节点。

### P1-05 远端 Get 不继承 Context，也没有转发跳数限制

- **位置**：`group.go:343-376`、`client.go:186-199`、`server.go:248-264`
- **触发条件**：上游请求取消、节点视图暂时不一致或出现循环路由。
- **问题**：`Peer.Get` 不接收 Context，`Client.Get` 使用新的 `context.Background()` 和固定 3 秒超时；Server 的 Get 也没有注入转发标记。
- **影响**：上游取消后远端请求仍继续执行；节点之间可能递归转发，形成请求链和额外延迟。
- **建议**：让 `Peer.Get` 接收 Context，传播 deadline，并增加最大转发跳数或请求 ID。

### P1-06 etcd 服务前缀匹配过宽

- **位置**：`peers.go:164`、`peers.go:219`、`registry/register.go:86-89`
- **触发条件**：同一 etcd 中存在具有前缀关系的多个服务名，例如 `cache` 和 `cache-admin`。
- **问题**：Watch 和全量查询使用 `/services/` 加服务名，没有追加结尾 `/`。
- **影响**：ClientPicker 可能发现并连接到其他服务的节点，造成错误路由和跨服务污染。
- **建议**：统一使用 `/services/{svcName}/` 作为前缀，并校验服务名。

### P1-07 服务发现存在事件丢失且 Watch 失败后不恢复

- **位置**：`peers.go:149-183`、`peers.go:214-234`
- **触发条件**：全量拉取和 Watch 建立之间发生节点变更，或 etcd Watch 通道关闭、被压缩或永久失败。
- **问题**：忽略全量查询返回的 revision；Watch 失败后直接退出 goroutine，不重连、不重新全量同步。
- **影响**：节点拓扑长期过期，错误路由到已下线节点或无法发现新节点。
- **建议**：使用 `Header.Revision + 1` 启动 Watch；遇到错误时重连并执行全量校准。

### P1-08 服务注册失败不会让 Server 启动失败

- **位置**：`server.go:219-225`
- **触发条件**：etcd 不可用、租约创建失败或注册写入失败。
- **问题**：注册 goroutine 只记录日志，gRPC Server 仍继续运行。
- **影响**：调用方以为节点已经加入集群，但其他节点无法发现它。
- **建议**：增加明确的 readiness 状态，或让启动流程返回注册失败。

### P1-09 Server 停止时不等待注册注销

- **位置**：`server.go:234-245`、`registry/register.go:103-122`
- **触发条件**：Server 优雅退出或进程快速退出。
- **问题**：注册函数内部创建独立 etcd Client，Server.Stop 没有等待注册 goroutine 完成 Revoke。
- **影响**：节点可能在 etcd 中保留到租约超时，其他节点继续向已停止节点发起请求。
- **建议**：统一 etcd Client 所有权，Stop 时等待注册注销完成。

### P1-10 etcd 连接无法配置 TLS 和认证

- **位置**：`server.go:132-139`、`peers.go:129-137`、`registry/register.go:56-59`
- **触发条件**：使用远程或生产 etcd 集群。
- **问题**：gRPC 支持 TLS，但 etcd Client 没有 CA、客户端证书、用户名密码或认证选项。
- **影响**：服务发现流量可能为明文，且 etcd 中的服务注册信息可能被伪造或篡改。
- **建议**：为 etcd 单独增加 TLS、CA、客户端证书和认证配置。

### P1-11 异步 Set/Delete 会无限制创建 goroutine

- **位置**：`group.go:237-239`、`group.go:263-265`
- **触发条件**：高写入量、远端节点故障或网络长时间阻塞。
- **问题**：每次同步操作都创建独立 goroutine，没有队列上限、并发上限或背压。
- **影响**：goroutine、上下文和待发送数据大量积累，最终造成资源耗尽。
- **建议**：使用有界队列和固定 worker，并设计重试、丢弃和背压策略。

### P1-12 Group 销毁不会等待进行中的任务

- **位置**：`group.go:237-239`、`group.go:284-308`
- **触发条件**：DestroyGroup 后仍存在回源或异步同步任务，或随后创建同名 Group。
- **问题**：旧 Group 的任务继续运行。
- **影响**：旧任务可能把旧数据写入同名的新 Group 或远端节点；回源和网络资源也不能及时释放。
- **建议**：为 Group 增加生命周期 Context 和 WaitGroup，关闭时停止新任务并等待旧任务退出。

### P1-13 全局注册表锁覆盖 Group 关闭和用户回调

- **位置**：`group.go:155-164`、`group.go:466-489`
- **触发条件**：同名替换、DestroyGroup 或 DestroyAllGroups，且缓存回调访问注册表。
- **问题**：持有 `groupsMu` 调用 `g.close()`，关闭过程可能 Clear 缓存并执行用户回调。
- **影响**：回调调用 `GetGroup` 或创建 Group 时死锁；销毁大缓存时所有注册表操作都会被阻塞。
- **建议**：先在注册表锁内摘除或替换 Group，释放锁后再关闭旧实例。

### P1-14 `from_peer` Context Key 可被业务调用方伪造

- **位置**：`group.go:222-239`、`server.go:274-276`
- **触发条件**：嵌入式业务代码构造 `context.WithValue(ctx, "from_peer", true)`。
- **问题**：字符串 key 没有包级私有性，业务调用方可绕过节点同步逻辑。
- **影响**：业务请求可能只修改一个节点，导致集群状态不一致。
- **建议**：使用不可导出的强类型 Context Key，或使用受认证保护的 gRPC metadata。

### P1-15 跨节点传播会重置 TTL

- **位置**：`group.go:228-238`、`group.go:332-337`、`group.go:395-403`
- **触发条件**：启用 peers，存在异步 Set 同步或远端 Get 回填。
- **问题**：远端收到数据或本地从 peer 回填时，以当前时间重新计算 TTL。
- **影响**：数据可能在多次传播后存活远超原始 TTL。
- **建议**：传播绝对过期时间或剩余 TTL。

### P1-16 未命中结果没有负缓存

- **位置**：`group.go:342-367`
- **触发条件**：外部请求大量随机、不存在的 key。
- **问题**：Getter 返回“不存在”后不会缓存负结果。
- **影响**：每次请求都访问后端，容易形成缓存穿透和后端压力放大。
- **建议**：支持可配置的短 TTL 负缓存，并增加 key 校验、限流和后端保护。

### P1-17 一致性哈希重复节点和哈希碰撞会破坏环

- **位置**：`consistenthash/consistenthash.go:96-103`、`consistenthash/consistenthash.go:124-138`、`consistenthash/consistenthash.go:174-188`
- **触发条件**：重复调用 Add，或多个虚拟节点产生相同哈希值。
- **问题**：重复节点会留下重复虚拟节点；CRC32 碰撞会覆盖 `hashMap` 映射。Remove 后可能留下没有对应 node 的 key。
- **影响**：路由结果错误，`Get` 访问空 `nodeCounts` 指针时可能 panic。
- **建议**：Add 做幂等检查；使用碰撞安全的映射结构或更大的哈希空间。

### P1-18 重平衡路径混用原子和非原子读写

- **位置**：`consistenthash/consistenthash.go:175-176`、`consistenthash/consistenthash.go:217`、`consistenthash/consistenthash.go:249`、`consistenthash/consistenthash.go:286`
- **触发条件**：进入动态重平衡路径并发处理请求。
- **问题**：`totalRequests` 的写入使用原子操作，但部分读取直接访问字段。
- **影响**：存在数据竞争和不一致读取风险。
- **建议**：所有访问统一使用 `atomic.LoadInt64`、`atomic.StoreInt64` 或在同一锁协议下完成。

## P2：中优先级

### P2-01 多个 Close API 不幂等

- **位置**：`store/lru.go:266-273`、`consistenthash/consistenthash.go:328-330`、`peers.go:281-303`、`server.go:235-245`
- **触发条件**：重复关闭、并发关闭，或多个组件共同负责清理。
- **问题**：部分实现重复关闭 channel 或重复停止资源，和 LRU-2 的 `sync.Once` 语义不一致。
- **影响**：可能触发 panic 或产生不一致的关闭行为。
- **建议**：所有 Close 使用 `sync.Once`，并明确 Close 后的并发调用语义。

### P2-02 `WithCacheOptions` 会覆盖 `cacheBytes`

- **位置**：`group.go:120-123`、`group.go:139-146`
- **触发条件**：调用 `WithCacheOptions`，但没有在选项中重新设置 `MaxBytes`。
- **问题**：选项直接替换 `mainCache`，原始 `cacheBytes` 被丢弃。
- **影响**：LRU 可能因为 `MaxBytes==0` 变成无界缓存；其他默认参数也可能被重置。
- **建议**：合并配置而不是整体替换，或移除重复的 `cacheBytes` 参数。

### P2-03 LRU 和 LRU-2 的 nil、负 TTL 语义不一致

- **位置**：`store/lru.go:138-154`、`store/lru2.go:174-196`、`group.go:102-106`
- **触发条件**：调用 `SetWithExpiration(key, nil, ...)` 或传入负 TTL。
- **问题**：LRU 将 nil value 解释为删除，LRU-2 可能保存 nil value；负 TTL 在多个路径中被解释为永不过期。
- **影响**：相同 Store 接口在不同淘汰策略下行为不同，配置错误可能产生长期脏数据。
- **建议**：统一 nil 和负 TTL 语义。建议仅允许 `0` 表示永不过期，负值返回错误。

### P2-04 LRU-2 和一致性哈希配置缺少边界校验

- **位置**：`store/lru2.go:52-85`、`store/lru2.go:317-365`、`consistenthash/config.go:29-60`
- **触发条件**：使用极端容量、无效 replica、空 HashFunc 或不合法负载均衡参数。
- **问题**：`uint16` 容量在 `cap+1`、`mask+1` 时可能溢出；自定义配置没有统一校验。
- **影响**：构造成功后，首次访问或写入时可能切片越界 panic，或创建出空环。
- **建议**：内部容量使用 `int`，构造阶段验证所有配置并返回错误。

### P2-05 Store 的 Len 和统计 size 可能包含过期数据

- **位置**：`store/lru.go:245-250`、`store/lru2.go:249-267`
- **触发条件**：TTL 已到期，但尚未发生 Get 或后台清理。
- **问题**：Len 依赖惰性删除或 ticker 清理，不能即时过滤过期项。
- **影响**：`Len` 和 `Cache.Stats()["size"]` 可能大于真实有效条目数。
- **建议**：Len 时过滤过期项，或维护严格的有效条目计数。

### P2-06 SingleFlight 等待者无法响应自己的 Context 取消

- **位置**：`singleflight/singleflight.go:63-69`
- **触发条件**：leader 的回源操作长时间阻塞，等待者的请求已取消或超时。
- **问题**：等待者只能阻塞在 `Wait()`。
- **影响**：HTTP/gRPC 请求 goroutine 和相关资源持续占用。
- **建议**：使用完成 channel 和 `select`，支持等待者独立取消。

### P2-07 Peer 返回的 byte slice 未复制

- **位置**：`group.go:370-376`
- **触发条件**：自定义 Peer 返回可复用或后续会修改的 slice。
- **问题**：ByteView 直接引用 Peer 返回的底层数组。
- **影响**：缓存数据可能被外部 buffer 修改，产生数据污染或数据竞争。
- **建议**：构造 ByteView 时始终复制数据，并在 Peer 接口中明确 slice 所有权。

### P2-08 ByteView 暴露可修改内部数据的 `Bytes`

- **位置**：`byteview.go:40-44`
- **触发条件**：调用方使用 `Bytes()` 返回值并修改切片。
- **问题**：`Bytes()` 返回内部切片引用，与“只读视图”语义冲突。
- **影响**：直接污染缓存，且并发读写时可能产生数据竞争。
- **建议**：公开 API 只返回副本；零拷贝接口限制为内部 API。

### P2-09 客户端 Get 没有检查协议响应 Code

- **位置**：`client.go:186-199`
- **触发条件**：服务端返回非零 `Code` 但没有同时返回 gRPC error。
- **问题**：客户端直接返回 `resp.GetValue()`。
- **影响**：错误可能被当作成功的空值。
- **建议**：统一使用标准 gRPC status，或严格检查响应 Code。

### P2-10 统计数据不是一致快照

- **位置**：`cache.go:320-342`、`group.go:414-448`
- **触发条件**：Stats 与 Close、Clear、读写请求并发执行。
- **问题**：统计字段来自不同时间点；`Group.loads` 统计所有等待者，而不是实际回源次数。
- **影响**：可能看到互相矛盾的状态，监控指标和告警判断失真。
- **建议**：定义统计口径，区分请求数、实际加载数和等待数；必要时生成一致快照。

### P2-11 Stats HTTP 接口无认证、无 TLS 和请求超时

- **位置**：`server.go:318-377`
- **触发条件**：StatsAddr 绑定到非本机或公网地址。
- **问题**：接口没有访问控制，也没有完整的 HTTP 超时配置。
- **影响**：暴露 Group 名称和运行状态，并增加连接资源耗尽风险。
- **建议**：默认绑定管理网或回环地址，增加认证、TLS 和 HTTP 超时。

### P2-12 NewServer 部分初始化失败时泄漏 etcd Client

- **位置**：`server.go:132-154`
- **触发条件**：etcd Client 创建成功，但 TLS 证书加载失败。
- **问题**：构造函数直接返回，没有关闭已经创建的 etcd Client。
- **影响**：反复创建失败时累积连接和后台资源。
- **建议**：所有初始化失败路径统一释放已创建资源。

### P2-13 全局默认配置使用可变指针

- **位置**：`server.go:68-73`、`registry/register.go:35-39`、`consistenthash/consistenthash.go:59-69`
- **触发条件**：业务代码修改导出的默认配置，或运行中并发读取和修改配置。
- **问题**：默认配置以可变全局指针暴露。
- **影响**：不同实例之间互相影响，甚至产生数据竞争。
- **建议**：返回配置副本，或将默认配置改为不可变值。

## P3：低优先级和 API 改进

### P3-01 外部包无法方便构造 ByteView

- **位置**：`byteview.go:26-28`
- **问题**：`ByteView.b` 未导出，也没有构造函数。外部包直接使用 `NewCache` 和 `Cache.Add` 时无法方便创建非空 ByteView。
- **建议**：提供 `NewByteView([]byte)`，并在构造时复制输入。

### P3-02 错误类型和协议错误码不统一

- **位置**：`server.go:248-315`、`client.go:186-233`
- **问题**：Server 多数错误同时返回 response 和 gRPC error，客户端通常只能拿到 gRPC error；错误码 `2` 也被用于 Getter 失败等非 key-not-found 场景。
- **建议**：统一使用标准 gRPC status code，减少重复的自定义 Code 字段。

### P3-03 Protobuf `go_package` 使用相对路径

- **位置**：`proto/reachcache.proto:22`
- **问题**：`go_package` 为 `"./"`，生成代码包名为 `__`。当前可以编译，但不利于下游重新生成、导入和 API 维护。
- **建议**：使用稳定的完整 Go import path，例如 `github.com/vernmorn/reachcache/proto`。

### P3-04 示例代码包含不安全或不严谨实现

- **位置**：`examples/shortlink/shortcode.go`、`examples/shortlink/main.go`、`examples/shortlink/db.go`、`examples/single/main.go`
- **问题**：示例使用硬编码短链接密钥、无认证 HTTP 接口、非严格 JSON 转义，并忽略多处数据库错误。
- **影响**：示例容易被误认为生产模板，导致密钥泄露、接口暴露或错误处理缺失。
- **建议**：明确标注仅供演示，改用环境变量密钥、严格 JSON 编码、输入校验和完整错误处理。

## 建议修复顺序

1. 修复本节点不入环、默认 gRPC 暴露、LRU-2 内存上限和一致性哈希重平衡死锁。
2. 修复 SingleFlight 竞态、回源覆盖、LRU-2 双版本和异步同步乱序问题。
3. 修复 Context 传播、Get 转发跳数、Watch revision/重连和 etcd 安全配置。
4. 增加 Group 任务生命周期、异步队列、统一关闭流程和注册注销等待。
5. 统一配置校验、TTL/nil 语义、错误协议、统计口径和 ByteView API。
6. 最后整理 Protobuf 生成配置和示例代码的安全性。
