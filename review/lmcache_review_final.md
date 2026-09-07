# LMCache 代码评审复查最终报告

复查日期：2026-09-07。本文面向维护者及后续修复人员，是评审结论与问题去重记录，不是用户使用文档或实现设计。

## 1. 最终结论

三份初始报告包含真实缺陷，但不足以支持“全部问题均已确认”“均没有对应 issue”或性能收益表中的具体数字。

- **存储报告最具可操作性**：磁盘锁泄漏、磁盘读取竞态、部分 layer 命中后的 pin 生命周期等有源码依据；前两项已有直接对应的 issue 和修复 PR。
- **协议与控制器报告需要修正部分推导**：共享 set 的迭代异常、重复创建 manager 必然导致指标重复注册等结论不成立；控制面生命周期、消息发送及 TCP 读包问题值得保留。
- **性能报告应作为待测量的优化清单**：逐 chunk 同步、线性扫描和部分绑定持有 GIL 的现象存在，但锁创建频率、vLLM token 输入类型、PCIe 流量、指针初始化缓存等描述有事实错误。
- **未搜索到不等于不存在**：只能表述为“截至本次检索，未发现同根因的公开 issue/PR”。关闭的 PR 也不能等同于已合并或已修复。

本报告不沿用初始报告的“确认问题总数”。下文区分源码可证事实、隔离复现、触发条件不足和优化观察项，避免把它们合计为已复现缺陷。

## 2. 范围、版本与验证方法

复查输入为本地的以下三份报告；本文包含独立结论，不要求读者同时取得原报告：

| 原报告 | ID 前缀 | 内容 |
| --- | --- | --- |
| `lmcache_review_storage.md` | storage | 存储后端、引用计数、pin 生命周期及关闭路径 |
| `lmcache_review_protocol.md` | protocol | 协议、控制器、REST、异步任务及可观测性 |
| `lmcache_review_perf.md` | perf | GPU 传输、分配器、哈希及 Python 热路径 |

初始报告基线为 `7de9c79e892104f75108cab84ba7dd95dcc8e7c3`，本次核对源码为 `16df1bbb35c958e8353ddeeac66b4e6f89c64800`。两者仅有 `.buildkite/scripts/amd-vllm-bench-container.sh` 变化，相关 `lmcache/`、`csrc/` 源码相同。因此本次发现的报告错误不是版本漂移造成的。

验证包括：

1. 对照源码及实际调用方核查触发条件，尤其区分公共方法默认参数与集成调用时显式传入的参数。
2. 使用 GitHub Search API 搜索上游 `LMCache/LMCache` 的公开 issue 和 PR，不限制 open/closed；阅读关键候选正文并核查 `merged_at`。
3. 使用从源码提取的方法配合轻量替身，隔离验证锁泄漏、缺 key 行为和 TCP GET 交错；另验证报告建议的 struct 格式。

验证没有覆盖真实 CUDA/RDMA 传输、GPU 基准测试、完整 pytest 套件或生产负载。隔离复现验证的是方法行为，不代表完成了端到端故障复现。

下文源码链接固定到复查 commit，便于后续代码变化后仍能核对证据。

## 3. 已有 issue/PR：直接对应与相关记录

### 3.1 直接对应

| 条目 | 已有记录及检索时状态 | 对应关系 |
| --- | --- | --- |
| storage C1：CPU staging 分配失败泄漏 `disk_lock` | [Issue #3855](https://github.com/LMCache/LMCache/issues/3855) open；[PR #4036](https://github.com/LMCache/LMCache/pull/4036) open；[PR #3555](https://github.com/LMCache/LMCache/pull/3555) closed、未合并 | 正文明确指向同一方法的分配失败提前返回路径。当前源码仍有该缺陷。 |
| storage H2：磁盘读取后访问 `cached_positions` 抛 KeyError | [Issue #2420](https://github.com/LMCache/LMCache/issues/2420) closed；[PR #4405](https://github.com/LMCache/LMCache/pull/4405) open | Issue 给出同一行调用栈，PR 修复同一锁外查字典的竞争窗口。 |
| protocol M3：TCP GET 事务拆锁与消息头短读 | [PR #3569](https://github.com/LMCache/LMCache/pull/3569) closed、未合并，关联 [Issue #3565](https://github.com/LMCache/LMCache/issues/3565) | PR 已包含完整消息头读取和整个 GET 事务加锁。Issue 主诉是断连重连，不能把该 issue 的全部内容当作拆锁竞态复现。 |
| protocol L11：健康检查与默认关闭的 worker 心跳不一致 | [PR #4808](https://github.com/LMCache/LMCache/pull/4808) open | 修复同一配置组合风险。但其文案使用 30 秒超时，复查源码的 controller 默认超时为 300 秒，健康检查默认关闭；必须说明启用条件。 |

### 3.2 相关，但不能据此认定重复或已修复

| 条目 | 相关记录 | 区别 |
| --- | --- | --- |
| storage H1：lookup 部分 layer 命中时 pin 未回滚 | [#2954](https://github.com/LMCache/LMCache/issues/2954)、[#4133](https://github.com/LMCache/LMCache/issues/4133)、[PR #4346](https://github.com/LMCache/LMCache/pull/4346) | 这些主要讨论 retrieve 重复 unpin、提前逐出或结果不足；不同于 lookup 未接受该 chunk 时遗留 pin。#4346 已关闭但未合并。 |
| storage M1：sync PD remove 的引用计数职责 | [PR #2934](https://github.com/LMCache/LMCache/pull/2934) | 主要处理 APC 跳过的 PD buffer 清理，不能替代 remove 公共契约的审计。 |
| storage M2：abort 后仍在运行的异步传输 | 已合并 [PR #3444](https://github.com/LMCache/LMCache/pull/3444) | 处理共享 key 去重、引用计数和 CancelNotif key 跟踪，不等同于禁止 aborted request 的在途任务继续完成。 |
| protocol H4：旧二进制协议缺少版本标记 | [Issue #4719](https://github.com/LMCache/LMCache/issues/4719) | 讨论 MP 协议和接口兼容性，与 `v1/protocol.py` 的旧 wire format 不是同一实现。 |
| perf：Python 控制路径开销 | [RFC #4344](https://github.com/LMCache/LMCache/issues/4344) | 关注 MP 模式，不能直接作为旧 connector 每项性能问题的重复记录。 |

检索使用了方法名、字段名、错误文本和场景组合，例如 `disk_lock`、`cached_positions`、`prefetch_all_done_callback`、`hot_cache KeyError`、`lookup_pins partial`、`send_multipart`、`run_script`、`PDBackendAsync abort`、`p2p_max_retry_count`、`single_layer_kv_transfer gil`、`store_stream synchronize`、`token_database cpu` 等。部分宽泛查询命中数超过单页上限，随后使用窄查询补查；这不是公开历史的全量穷举，也无法覆盖私有安全报告。

## 4. 存储报告复查

| ID | 最终判断 | 保留或修正内容 |
| --- | --- | --- |
| C1 | 成立；隔离复现；已有跟踪 | `allocate()` 返回 None 后锁保持占用。建议按高优先级可用性缺陷处理。修复还需处理此前分配但尚未读入的 buffer、disk pin 和 staging pin/ref；不能只补一次 release。 |
| C2 | 实现缺陷成立；默认场景和 Critical 评级证据不足 | 缺 key 确实抛 KeyError，完成回调裸调 `task.result()` 也缺异常兜底。但实际 async lookup 客户端显式传 `pin=True`，不能用 engine 方法的 `pin=False` 默认值证明常规 LRU 即可触发。应覆盖 pin=False、强制删除或 pin 生命周期失效等明确场景。 |
| H1 | 源码支持；值得独立修复 | 部分 layer 命中后提前返回，已 pin 的 key 没有进入 `lookup_pins`，也没有回滚。还应覆盖所有 layer 分散在多个 backend、因 `len(block_mapping) != 1` 被拒绝的情况。回滚需按 backend 分组。 |
| H2 | 成立；已有跟踪 | 锁外 IO 后无保护访问 `self.dict[key]`。缺文件路径也会删 key 后返回，后续同样可能 KeyError。修复需释放已分配对象并返回 miss。 |
| H3 | 部分成立，拆分处理 | `close()` 等待 future 无超时成立；None 连接使用 assert 不合理。构造器抛错时 `self.connection = CreateConnector(...)` 尚未完成赋值，仅凭 except 中置 None 不能证明半成品泄漏，需给出具体构造器及资源证据。 |
| M1 | 契约不一致成立；泄漏范围需补验证 | sync PD remove 依赖 engine 补 refcount，与其他 backend 不一致。但是否延迟或永久占用槽位取决于对象剩余引用、allocator 和调用路径；不能统一归为所有 controller clear 都泄漏。 |
| M2 | 生命周期风险保留；原时序必须改写 | 适配器首次计算后保存 `total_chunks`，新批次不必然携带 0。更有根据的风险是已进入传输的任务在 abort 清空状态后继续完成，完成检查读取缺失 total 的默认 0。需确定性调度测试验证 notification 与 receiver 状态。 |
| M3 | write-back 门禁问题成立；UAF 推断不足 | 单条 put 没有 `use_hot` 门禁，write-back 可插入对象而 clear 直接跳过。close 顺序也值得整理；但 allocator close 后 free 若仅操作记账，不自动构成 UAF，必须证明实际访问已释放内存。 |
| M4 | 配置边界问题成立，收窄场景 | `max_retry_count <= 0` 使循环不执行，随后使用未绑定的 `ret_msg`。反复收到 REMOTE_XFER_HANDLER_NOT_INITIALIZED 时变量已经赋值，不是未绑定异常的触发条件。 |
| L1 | 关闭生命周期观察项 | loop.stop 不等待全部任务完成；具体丢失的引用和回调需结合 backend 自身的 drain/close 行为验证，不应描述为全部剩余回调必然被截断。 |
| L2 | 不列为当前运行缺陷 | AsyncMultiSerializer 当前未启用；未来启用前需要并发契约测试，不能据此计入活跃 bug 数量。 |
| L3 | 低优先级写法改进 | 时间戳为 0 时真值判断不同于存在性判断，但正常 `time.time()` 输入下未建立实际故障场景。 |

关键源码：

- [磁盘异步读取与失败返回](https://github.com/LMCache/LMCache/blob/16df1bbb35c958e8353ddeeac66b4e6f89c64800/lmcache/v1/storage_backend/local_disk_backend.py#L544-L608)。
- [CPU 异步读取](https://github.com/LMCache/LMCache/blob/16df1bbb35c958e8353ddeeac66b4e6f89c64800/lmcache/v1/storage_backend/local_cpu_backend.py#L225-L238)及[实际客户端传入 pin=True](https://github.com/LMCache/LMCache/blob/16df1bbb35c958e8353ddeeac66b4e6f89c64800/lmcache/v1/lookup_client/lmcache_async_lookup_client.py#L369-L374)。
- [layerwise lookup 接受条件与提前返回](https://github.com/LMCache/LMCache/blob/16df1bbb35c958e8353ddeeac66b4e6f89c64800/lmcache/v1/cache_engine.py#L1190-L1220)。
- [适配器保存 total_chunks](https://github.com/LMCache/LMCache/blob/16df1bbb35c958e8353ddeeac66b4e6f89c64800/lmcache/integration/vllm/vllm_v1_adapter.py#L413-L428)。

## 5. 协议、控制器与监控报告复查

| ID | 最终判断 | 保留或修正内容 |
| --- | --- | --- |
| C1 | 高优先级暴露风险；未进行利用验证 | controller 注册 common API，`/run_script` 执行上传脚本并注入 app，受限 builtins 不能视为可靠隔离边界。源码路由链成立；实际严重度取决于服务可达性和外围访问控制。未发现同根因公开修复记录不代表没有私有安全报告。 |
| H1 | 缺 await 成立 | PyZMQ 返回 Future；未 await 不等于完全不发送，但会丢失错误处理和背压。补 await 的方向合理，故障时的消息保留与重试需要另行定义。 |
| H2 | 跨 loop 使用风险成立 | P2P 与 full sync 使用同一 DEALER，无统一 loop 归属及请求响应配对机制。具体错配时序需并发测试；不能笼统断言所有 PyZMQ 版本必然同样报错。 |
| H3 | 无超时成立；gather 描述错误 | controller→worker 的 REQ 未设收发超时。`asyncio.gather` 默认传播首个异常，但不自动取消其他任务。超时后还需处理 REQ socket 状态，不能只增加 timeout 参数。 |
| H4 | 兼容性设计缺口 | 旧 struct 协议缺少版本及自描述信息属实；“升级即静默损坏”过度概括，应给出实际不兼容版本或畸形输入。与 MP 协议分开处理。 |
| H5 | 撤回所述崩溃结论 | `for key in hashes` 迭代请求列表，对共享 set 做成员查询。不能因此产生“set 迭代期间大小变化”的 RuntimeError。活引用的快照一致性可另作设计讨论。 |
| H6 | PULL handler 任务退出风险成立 | PULL 消息解码/处理没有逐消息保护，gather 保留异常结果但无消费。REQ、heartbeat handler 已有部分异常保护，不能声称三个 handler 完全相同。 |
| M1 | 部分成立 | 心跳计入 REPLY 指标确实混淆来源。使用 Gauge 作为单调计数不规范，但不能说 PromQL `rate()` 因类型而无法执行；数值解释取决于实际序列和重置行为。 |
| M2 | 并发遍历风险成立；缓存影响需场景化 | instances/workers 未快照遍历，与 health 线程删除有竞争。30 秒缓存影响 worker 成员集合，但缓存里是活对象，不能把所有 sync 状态都说成冻结了 30 秒。 |
| M3 | 成立；隔离复现；已有相关修复 PR | 拆锁在已有等待者时可交错，下一 GET 将前一 GET 的 body 当 header。单个 recv 也不保证完整头。无竞争时未必让出控制权，复现必须包含锁等待条件。 |
| M4 | no-op 成立，原修复不完整 | 默认 `[]` 与消息中“None 表示全部”的意图不一致；engine 没有实现全部压缩，单改 None 会到 tokens/hashes 缺失异常。应明确实现全量操作或要求非空 tokens。 |
| M5 | 关闭竞态成立，概率措辞撤回 | deregister 仅入队就停 loop，不能保证发送完成；“几乎必然”没有测量支持。应验证 flush/确认语义。 |
| M6 | 注册失败后缺少重试成立 | register 抛错后 start_all 仅记录错误，后续循环未启动。应定义重试或明确失败退出，避免长期无控制面注册。 |
| M7 | 重复注册清空 metadata 成立 | 是否导致持久 miss 取决于 full sync 成功与否。来源认证属于独立的信任边界议题，不应与普通重注册可靠性混成一个修复。 |
| M8 | 忽略完成状态成立 | `_poll_for_completion()` 返回 False 后仍记录成功并返回 True；应传递结果。任务引用保留属于另一项生命周期改进。 |
| M9 | 观察项，需建立事件时间线 | full sync 丢弃增量是已有设计行为；需要结合 freeze 生效、快照、在途 store/evict 证明遗漏，不能仅凭 debug 日志级别判定数据发散。 |
| M10 | shutdown 清理不完整成立 | lifespan 未调用 manager.close，health loop/thread 清理不完整。不能据此断言进程退出后必然持续占端口或重启失败。 |
| M11 | handler 错误路径保护不足 | 异常响应发送再次失败可退出请求任务；“对端断开就一定导致 REP send 抛错”不是普遍成立的触发条件。 |
| L1 | 低优先级协议边界项 | 空白填充与 strip 无法保留合法 key 的边界空白；需要明确 key 字符契约。 |
| L2 | 可移植性改进；示例错误 | 本地字节序应显式化，但常见 x86/ARM 都是小端。`<!i` 是非法格式，应选择 `<i` 或 `!i` 等单一前缀。 |
| L3 | 未实现功能 | LIST 枚举与服务端缺少对应处理不一致；常规客户端 list 本身未实现，不能当作默认路径挂死。 |
| L4 | 输入限制与错误响应观察项 | 缺少长度限制和线程上限值得检查；连接关闭后客户端可以立即读到 EOF，并非只能等超时。 |
| L5 | 错误处理改进 | assert 用作运行期响应校验不合适，HTTP 错误映射应明确；信息泄露严重度取决于响应内容和访问边界。 |
| L6 | 聚合契约待明确 | ErrorMsg 与 num_tokens 解包不匹配可发生；worker 数量是否必须一致需按操作、TP/MLA 语义定义，不能统一改成求和。 |
| L7 | 与 M1/M2 合并 | 指标缓存延迟是可观测性权衡；Gauge 计数建议与 M1 合并，避免重复计数。 |
| L8 | 降级语义与监控改进 | 请求失败退化为 miss 可以是容错策略；应区分失败指标与正常 miss，但不直接认定返回 miss 本身错误。 |
| L9 | 撤回必然重复注册结论 | manager 使用 GetOrCreate 复用 singleton，第二次创建不必然注册相同 collector；不同 labels 仍沿用旧实例的问题保留。 |
| L10 | 初始化条件限制成立，原描述不精确 | heartbeat 是否运行只判断一次；条件不满足时直接结束，并不是必然进入 warning 空转。重配置/重注册场景需要明确支持契约。 |
| L11 | 条件成立；已有 PR | 仅在启用健康检查、未启用有效心跳等配置组合下发生，不是所有默认启动都会清除 worker。 |

关键证据：

- [脚本 API](https://github.com/LMCache/LMCache/blob/16df1bbb35c958e8353ddeeac66b4e6f89c64800/lmcache/v1/internal_api_server/common/run_script_api.py)及[controller 路由注册](https://github.com/LMCache/LMCache/blob/16df1bbb35c958e8353ddeeac66b4e6f89c64800/lmcache/v1/api_server/__main__.py#L97-L116)。
- [P2P lookup 的实际迭代对象](https://github.com/LMCache/LMCache/blob/16df1bbb35c958e8353ddeeac66b4e6f89c64800/lmcache/v1/cache_controller/controllers/kv_controller.py#L404-L439)。
- [TCP GET 拆锁](https://github.com/LMCache/LMCache/blob/16df1bbb35c958e8353ddeeac66b4e6f89c64800/lmcache/v1/storage_backend/connector/lm_connector.py#L138-L168)。
- [engine compress 的 tokens 使用](https://github.com/LMCache/LMCache/blob/16df1bbb35c958e8353ddeeac66b4e6f89c64800/lmcache/v1/cache_engine.py#L1424-L1455)。
- [Python gather 行为](https://docs.python.org/3/library/asyncio-task.html#asyncio.gather)、[struct 字节序前缀](https://docs.python.org/3/library/struct.html#byte-order-size-and-alignment)、[PyZMQ asyncio Future 接口](https://pyzmq.readthedocs.io/en/latest/api/zmq.asyncio.html)。

## 6. 性能报告复查

| ID | 最终判断 | 保留或修正内容 |
| --- | --- | --- |
| H1 | 性能候选，需基准 | 非 CUDA 目标逐对象同步、batch 循环属实。删除同步前必须保持 buffer 完成与复用契约；异步 serde 或完成回调不会自动建立 CUDA 依赖。撤回 30–60% 损失/收益。 |
| H2 | 核心描述错误，去锁建议不可直接采用 | 每个对象构造一把锁，并非每次 ref/pin 操作创建新锁。锁保护计数修改、归零判断和 allocator 释放等复合过程，不能用 GIL 或 itertools.count 替代整个契约。撤回 5–15% CPU 收益。 |
| H3 | 输入前提错误 | `_hash_tokens` 支持 Tensor，GPU Tensor 时确有逐 chunk `.cpu()` 成本；实际 vLLM 适配器主路径传 list[int]，不能称默认每请求多次 D2H。撤回 1–5ms。直接改 bytes 哈希还会改变缓存 key 兼容性。 |
| H4 | 算法事实成立，评级待测 | AddressManager first-fit 线性扫描存在；PagedTensorMemoryAllocator 稳态是预分配对象的 deque，不能混为同一分配路径。上千空闲块达到毫秒、优化数量级收益没有实测。 |
| H5 | 部分绑定持有 GIL 属实 | 补 release 可以评估，但必须检查 Python API 访问、回调、对象生命周期及锁顺序；“返回 void，所以零风险”不成立。 |
| M1 | PCIe 流量论断错误 | 路径是 GPU KV→GPU staging→CPU，新增设备内搬运，不是两次跨 PCIe D2H。应比较 gather 后 DMA 与直接写 pinned host 的真实吞吐。 |
| M2 | 指针重复 H2D 论断错误 | V2 `_initialize_pointers()` 已按设备缓存并提前返回。外围 contiguity/布局检查可能重复，但不能据此声称每 chunk 重填指针并 H2D。 |
| M3 | 可测量的 CPU 优化候选 | 重复请求处理和 key 字符串生成存在；prefix hash 已是滚动链式计算，不是每 chunk 重新扫描全部前缀。缓存 hash/string 要处理 key 可变字段和生命周期。 |
| M4 | 监控锁开销候选 | 单对象路径的多次更新存在；batch 路径不能直接按单对象次数外推。去锁前需证明 monitor 数据及采样一致性不依赖该锁。 |
| M5 | 定向同步优化候选 | layerwise 的全设备同步存在；event 替代需覆盖 load、compute、buffer 复用等完整依赖，不能未经测量认定层间重叠收益。 |
| M6 | 设计权衡观察项 | lazy pin 降低启动阻塞，但后台注册可能增加运行期成本；chunk 增大也可能拉长单次注册停顿，应测量后选择。 |
| L1 | 非默认路径观察项 | separator 滑窗比较复杂度可分析，通常短 separator 下不构成已证瓶颈；searchsorted 不是任意子序列匹配的直接替代。 |
| L2 | CPU 拷贝候选 | 串行 memmove 是否适合并行取决于对象尺寸和带宽；cudaMemcpyAsync 不是所有 CPU/SHM 拷贝的直接替代。 |
| L3 | 启动成本观察项 | paged 对象预创建属实，稳态复用；“GB buffer / 64MB 页即数千对象”数量级不普遍成立，数千页需要约百 GB 容量。 |
| L4 | 小开销候选 | key hash 可考虑缓存，但应先固定或正确失效相关字段，不能直接套用 frozen/memo。 |
| L5 | 限定正面结论的范围 | 已看到向量化、融合和异步传输设计；静态抽查不能证明全部 native 内核正确，也不能担保全部序列化路径零多余拷贝。 |
| L6 | 参考实现，不是等价替换方案 | MP blake3 的批量打包可作参考；迁移旧 token_database 时须保留或迁移哈希兼容契约。 |

关键源码：

- [V2 指针缓存提前返回](https://github.com/LMCache/LMCache/blob/16df1bbb35c958e8353ddeeac66b4e6f89c64800/lmcache/v1/gpu_connector/gpu_connectors.py#L246-L273)。
- [GPU staging 与同步](https://github.com/LMCache/LMCache/blob/16df1bbb35c958e8353ddeeac66b4e6f89c64800/lmcache/v1/gpu_connector/gpu_connectors.py#L375-L428)。
- [MemoryObj 锁初始化](https://github.com/LMCache/LMCache/blob/16df1bbb35c958e8353ddeeac66b4e6f89c64800/lmcache/v1/memory_management.py#L636-L654)及[计数/释放临界区](https://github.com/LMCache/LMCache/blob/16df1bbb35c958e8353ddeeac66b4e6f89c64800/lmcache/v1/memory_management.py#L753-L820)。
- [vLLM 保存路径明确要求 list](https://github.com/LMCache/LMCache/blob/16df1bbb35c958e8353ddeeac66b4e6f89c64800/lmcache/integration/vllm/vllm_v1_adapter.py#L1041-L1046)。

## 7. 已执行的隔离验证

以下为复查时在本地执行的实际结果。方法从源码 AST 提取，去除装饰器，并注入轻量依赖替身；没有加载 GPU 后端或连接真实服务。

| 验证 | 构造条件 | 观察结果 |
| --- | --- | --- |
| 磁盘分配失败锁生命周期 | 一个合法 disk key；CPU allocate 返回 None | 返回 `[]` 后 `disk_lock.locked()` 为 True。验证后手工释放锁，未遗留阻塞任务。 |
| CPU 缺 key 行为 | 空 hot_cache；请求一个缺失 key | 原方法抛 KeyError。未模拟 pin=True 的完整 lookup 链，因此不证明默认集成路径可被普通逐出触发。 |
| TCP GET 交错 | 同一 asyncio.Lock 上已有两个 GET 等待者；FIFO 假 socket 按 header/body 顺序返回 | 第二个 GET 的 metadata 输入为第一个 GET 的 body；首个 GET 还可能消费第二个 header。 |
| struct 修复示例 | 执行 `struct.calcsize('<!i')` | 抛 `struct.error: bad char in struct format`。 |

TCP 交错的关键顺序为：

```text
锁当前被占用，GET A 与 GET B 均排队
释放锁 → A 获取锁并读取 header A
A 释放锁 → 唤醒 B
A 再次申请锁时因已有等待者而等待
B 获取锁，把 body A 作为自己的 header 读取
```

这说明“方法中没有显式业务 await”不足以排除交错：有等待者时，第二次 async lock acquire 本身就是让出执行权的位置。

## 8. 后续处理优先级与验收要求

1. **先处理可用性和控制面风险**：磁盘 staging 失败、磁盘读取竞态沿用已有 issue/PR；脚本端点暴露、controller handler 生命周期、REQ 超时和共享 socket 的问题按明确触发条件分别跟踪。
2. **补引用与 pin 契约测试**：覆盖部分 layer 命中、多 backend 分散命中、abort 后在途传输完成，以及 local_cpu=False write-back。检查返回值、可再次分配容量、可逐出性和通知是否正确，避免仅断言内部计数变化。
3. **修订不完整建议再实现**：compress 全量语义需贯通 REST 与 engine；disk 失败要回滚全部已取得资源；REQ 超时后要恢复 socket 状态；关闭流程要明确任务 drain/cancel 与 allocator 生命周期。
4. **性能项先测量再排序**：记录设备、传输模式、chunk 大小、层数、输入类型、批量大小、冷热状态与并发度，比较 TTFT、store/retrieve 时间、CPU profile、PCIe 流量和峰值内存。没有这些结果，不发布百分比或毫秒收益承诺。

对初始报告中“明确干净”“没有泄漏”等正面断言也应采用相同证据标准：它们只能表示本次已查看路径中未发现问题，不能作为模块级正确性保证。

本次交付为复查报告。它不包含运行时代码修复，也不代表已创建新 issue、向维护者发送消息或完成 GPU 回归验证。

## 9. 报告交付验证

- Markdown 已使用 CommonMark 解析器启用 table 扩展渲染，检查了 7 个表格、58 个逐项结论的 ID 顺序与完整性，以及链接均使用 HTTPS。
- 已检查报告文件无尾随空白等 diff 格式错误。
- 报告的 `codespell --toml pyproject.toml` 检查通过。
- 按仓库要求执行了 `make clean` 和 `make html SPHINXOPTS='-W --keep-going'`。站点 HTML 已生成，但严格构建未通过，汇总为 31 条 warning/error 诊断；包括原有 CLI 重复标签、controller 标题下划线、runtime plugin 示例引用缺失，以及 `chunk_statistics.rst` 的 code-block 指令错误。
- 本次没有修改 Sphinx 源文件。报告位于 `review/`，不属于 Sphinx 输入，因此站点构建不能验证报告本身，也不能将原有站点诊断记为本报告引入的回归。报告的格式使用前述独立 Markdown 渲染检查。
