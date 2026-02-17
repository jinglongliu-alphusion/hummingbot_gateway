# zquant_suite 项目规范

## 1. 组件生态总览

### 核心组件

| 组件 | 语言 | 定位 | 核心能力 |
|------|------|------|----------|
| **zquantmq** | C++17 | IPC 消息队列 | 无锁环形缓冲区、零拷贝消息、异步日志、Prometheus 指标 |
| **zquant_gateway** | C++20 | 交易网关 | 多交易所连接器、订单路由、持仓管理、风控服务 |
| **zquant_strategy** | C++/Python/Rust | 策略执行框架 | 策略接口、回测/实盘模式、多语言绑定 |
| **zquant_dashboard** | Rust + React | 监控仪表板 | 指标聚合、REST API、实时监控 |
| **hummingbot_gateway** | TypeScript | DEX 网关 | 区块链交互、DEX 连接器、钱包管理 |

### 组件依赖关系

```
zquantmq (基础库，无依赖)
    ↓
zquant_gateway (依赖 zquantmq)
    ↓
zquant_strategy (依赖 zquantmq + zquant_gateway)

zquant_dashboard (独立，通过 HTTP/RPC 集成)
hummingbot_gateway (独立，通过 REST API 集成)
```

### 数据流架构

```
交易所 ──► 连接器 ──► zquantmq ──► 策略
                         │
                         ├──► 日志系统 ──► log_dumper ──► 文件
                         │
                         └──► 指标系统 ──► metrics_emitter ──► Prometheus
```

---

## 2. 技术栈与系统约束

### C++ 标准

| 模块 | C++ 标准 | 编译器要求 |
|------|----------|-----------|
| zquantmq | C++17 | GCC 9+ / Clang 12+ |
| zquant_gateway | C++20 | GCC 11+ / Clang 14+ |
| zquant_strategy | C++17/20 | 同 gateway |

### 平台支持

| 平台 | 状态 | 最低版本 | 备注 |
|------|------|----------|------|
| Linux x86_64 | ✓ 生产 | 5.15.0+ | 主要目标平台 |
| macOS | ✓ 开发 | 12.0+ | 支持 Apple Silicon |
| Windows | ✗ 计划 | - | 未来支持 |

### 构建系统

- **主构建工具**: Bazel 6.0+ (Bzlmod 模式)
- **Rust 构建**: Cargo (dashboard)
- **TypeScript 构建**: pnpm (hummingbot_gateway)
- **Python**: 3.10+, Pybind11 绑定

### 核心依赖

| 依赖 | 版本 | 用途 |
|------|------|------|
| yaml-cpp | 0.8.0 | YAML 配置解析 |
| googletest | 1.14.0 | C++ 单元测试 |
| google_benchmark | 1.8.3 | 性能基准测试 |
| nlohmann_json | 3.11.3 | JSON 序列化 |
| protobuf | 21.7 | Protocol Buffers |
| SBE | 1.28.0 | 二进制消息编码 |

### 供应商库 (Vendor)

| 库 | 路径 | 说明 |
|----|------|------|
| IB API | vendor/ibapi | Interactive Brokers |
| QuickFIX | vendor/fix/quickfix | FIX 协议 |
| CTP | vendor/domestic/ctp | 国内期货 (仅动态库) |
| CCAPI | vendor/crypto/ccapi | 加密货币交易所 |

---

## 3. 构建规范

### Bazel 构建配置

```bash
# 发布构建
bazel build --config=release //...

# 调试构建
bazel build --config=debug //...

# 打包构建 (DEB)
bazel build --config=package //...

# 运行测试
bazel test //tests/... --test_tag_filters=-integration,-stress,-fuzz,-chaos
```

### 编译优化标志 (生产)

```
-O3 -march=native -mtune=native -flto -DNDEBUG
-falign-functions=64  # 缓存行对齐 (HFT 优化)
```

### 测试配置预设

| 配置 | 用途 |
|------|------|
| `--config=asan` | AddressSanitizer |
| `--config=tsan` | ThreadSanitizer |
| `--config=ubsan` | UndefinedBehaviorSanitizer |
| `--config=coverage` | 代码覆盖率 |

---

## 4. 代码规范

### 命名约定

- **函数/变量**: snake_case (`get_position`, `order_count`)
- **类/类型**: PascalCase (`OrderBook`, `MarketData`)
- **常量/枚举值**: UPPER_SNAKE_CASE (`MAX_BUFFER_SIZE`)
- **文件名**: snake_case.cpp, snake_case.h

### HFT 代码约束

1. **热路径禁止**:
   - 动态内存分配 (malloc/new)
   - 系统调用 (除非必要)
   - 异常抛出
   - 虚函数调用 (尽量)
   - 日志 I/O (使用异步日志)

2. **推荐做法**:
   - 预分配缓冲区
   - 使用 `claim()/commit()` 零拷贝 API
   - 原子操作使用 acquire/release 语义
   - 64 字节缓存行对齐

### 错误处理

- 使用 `std::expected` 或 `std::optional` (C++23/17)
- 避免异常用于控制流
- 返回错误码或状态枚举

---

## 5. 测试规范

### 测试分类

| 类型 | 标签 | CI 执行 | 说明 |
|------|------|---------|------|
| 单元测试 | (无) | ✓ 必需 | 快速、隔离 |
| 集成测试 | integration | ✓ 可选 | 跨组件交互 |
| 压力测试 | stress | ✗ 手动 | 性能验证 |
| 模糊测试 | fuzz | ✗ 手动 | 边界条件 |

### 测试执行

```bash
# 仅单元测试 (CI 默认)
bazel test //tests/... --test_tag_filters=-integration,-stress,-fuzz,-chaos

# 包含集成测试
bazel test //tests/... --test_tag_filters=-stress,-fuzz,-chaos

# 全部测试
bazel test //tests:all_tests
```

---

## 6. zquantmq Channel 路径规范

### 命名约定

Channel 路径采用分层路径式命名，遵循 Unix 文件系统风格：

```
/<namespace>/<component_type>/<specifics>
```

**规则：**
- 使用前导斜杠 `/` 作为根
- 命名空间：`zquant` 作为全局前缀
- 多级路径用 `/` 分隔
- 变量部分用 `{placeholder}` 表示

### Channel 路径层级

```
/zquant/
├── md/                          # Market Data (SPMC)
│   └── {connector}/out          # 行情输出
│
├── strategy/                    # 策略相关
│   └── {strategy_id}/
│       ├── {connector}/
│       │   ├── order_req        # 订单请求 (SPSC, 策略→连接器)
│       │   ├── order_status     # 订单状态 (SPMC, 连接器→策略/PM)
│       │   └── exec_status      # 成交回报 (SPMC, 连接器→策略/PM)
│       └── position             # 持仓更新 (SPMC)
│
├── log/                         # 日志 (SPMC)
│   └── {name}                   # Connector: connector-{cid}, Service: {service_name}
│       示例:
│       ├── connector-fix_lmax_baggioss
│       ├── gateway
│       └── position_manager
│
├── metrics/                     # 指标 (SPMC)
│   └── {name}                   # Connector: connector-{cid}, Service: {service_name}
│       示例:
│       ├── connector-fix_lmax_baggioss
│       └── gateway
│
├── {connector}/                 # 连接器事件
│   ├── event
│   └── error
│
└── overview/                    # 聚合数据 (SPMC)
    ├── position/out
    └── risk/out
```

### Channel 类型与模式

| 类型 | 路径模式 | 通信模式 | 方向 |
|------|---------|---------|------|
| Market Data | `/zquant/md/{connector}/out` | SPMC | 连接器→策略 |
| Order Request | `/zquant/strategy/{sid}/{conn}/order_req` | SPSC | 策略→连接器 |
| Order Status | `/zquant/strategy/{sid}/{conn}/order_status` | SPMC | 连接器→策略/PM |
| Exec Status | `/zquant/strategy/{sid}/{conn}/exec_status` | SPMC | 连接器→策略/PM |
| Position | `/zquant/strategy/{sid}/position` | SPMC | 策略→消费者 |
| Log | `/zquant/log/{name}` | SPMC | 进程→dumper/consumers |
| Metrics | `/zquant/metrics/{name}` | SPMC | 进程→emitter |
| Connector Event | `/zquant/{connector}/event` | SPMC | 连接器→策略 |

### 占位符定义

| 占位符 | 说明 | 示例 |
|--------|------|------|
| `{strategy_id}` | 策略唯一标识 | `arb_v1`, `mm_v2` |
| `{connector}` | 交易所连接器 | `ctp`, `fix`, `ibapi`, `ccapi` |
| `{name}` | 日志/指标通道名 | Connector: `connector-{cid}`, Service: `gateway`, `position_manager` |

**已废弃占位符：** `{process_name}` (被 `{name}` 替代)

### 支持的 Connectors

```yaml
connectors: [ibapi, fix, ctp, ccapi, mock]
```

### 共享内存地址转换

Channel 名到 SHM 名转换规则：
- 输入：`/zquant/md/ctp/out`
- IPC 地址：`ipc:///zquant/md/ctp/out`
- SHM 名称：`/zqmq_zquant_md_ctp_out`

### 配置参考

完整定义见 `/zquant_strategy/conf/channels.yaml`

---

## 7. zquantmq API 使用规范

### 原则：仅使用高层 API

所有组件必须通过高层 ZMQ 风格的 pub/sub API 使用 zquantmq，禁止直接访问底层 core 模块。

### 允许使用的 API (zquantmq/zmq/include/zmq/)

| API | 用途 |
|-----|------|
| `context_t` / `default_context_t` | Socket 上下文管理 |
| `push_socket` / `pull_socket` | SPSC 管道模式 |
| `pub_socket` / `sub_socket` | SPMC 广播模式 |
| `pair_socket` | 双向 1:1 通信 |
| `req_socket` / `rep_socket` | 请求-应答模式 |
| `poller_t` | I/O 多路复用 |
| `message_ref` | 零拷贝消息视图 |
| `bind()`, `connect()`, `close()` | Socket 生命周期 |
| `send()`, `recv()` | 消息操作 |
| `claim()`, `commit()` | 零拷贝消息操作 |

### 禁止直接使用的 API (zquantmq/core/)

以下内部 API 禁止在 zquantmq 模块外直接使用：

- `RingBufferSPSC` / `RingBufferSPMC` - 内部环形缓冲区实现
- `MmapRegion` - 原始共享内存管理
- `claim_slot()`, `commit_slot()` - 直接环形缓冲区操作
- `init_in_memory()`, `attach_to_memory()` - 手动内存生命周期管理

### 适用范围

本规范适用于：

- **zquant_gateway** - 所有核心代码、连接器及工具 (md_receiver, position_manager, risk_manager, dashboard_agent, metrics_emitter)
- **zquant_strategy** - 所有策略上下文和执行代码
- **zquant_dashboard** - 任何未来的 C++ 集成
- **zquantmq/logging** - 内部实现必须使用 zmq sockets
- **zquantmq/metrics** - 内部实现必须使用 zmq sockets
- **所有测试和示例** - 必须展示正确的 API 使用方式

### 设计理由

1. **可维护性** - 高层 API 提供稳定接口，内部实现可独立演进
2. **安全性** - 防止直接操作环形缓冲区导致的内存管理错误
3. **一致性** - 所有组件采用统一的消息传递模式
4. **可测试性** - 标准 socket 接口便于 mock 和测试

---

## 8. 性能目标

### IPC 延迟

| 指标 | 目标值 | 备注 |
|------|--------|------|
| SPSC p50 | <200ns | 生产者-消费者单通道 |
| SPSC p99 | <1µs | |
| SPMC p99 | <2µs | 广播模式 |
| 日志调用 | <200ns | 异步写入 |

### 吞吐量

| 指标 | 目标值 |
|------|--------|
| SPSC | >10M msg/sec |
| SPMC | >5M msg/sec |

### 连接器延迟

| 连接器 | 订单延迟目标 |
|--------|-------------|
| CTP | <15µs |
| FIX | <100µs |
| IB API | <1ms |
| CCAPI | <10ms |

---

## 9. 配置文件规范

### 配置文件位置

| 配置类型 | 路径 |
|----------|------|
| Gateway 主配置 | `zquant_gateway/conf/gateway.yaml` |
| 连接器配置 | `zquant_gateway/conf/connectors/*.yaml` |
| 服务配置 | `zquant_gateway/conf/services/*.yaml` |
| Strategy 配置 | `zquant_strategy/conf/live.yaml`, `paper.yaml` |
| Channel 配置 | `zquant_strategy/conf/channels.yaml` |
| 指标配置 | `zquantmq/conf/zquantmq_metrics.yaml` |

### 共享内存路径

| 环境 | 路径 |
|------|------|
| 生产 (Linux) | `/dev/shm/zquant/` |
| 开发 (macOS) | `/tmp/zquant/` |
| 自定义 | `$ZQUANT_SHM_DIR` |

---

## 10. CI/CD 规范

### CI 执行顺序

```
zquantmq (build + test)
    ↓
zquant_gateway (build + test)
    ↓
zquant_strategy (build + test)
zquant_dashboard (parallel, Rust + Node)
```

### CI 脚本

```bash
# 完整 CI
./ci/zquant-ci.sh

# 仅测试
./ci/zquant-ci.sh --stages test

# 单项目
./ci/zquant-ci.sh --projects zquant_gateway

# 部署
./ci/zquant-ci.sh --deploy aws-sg
```

### 部署产物

- DEB 包: `dist/*.deb`
- systemd 服务文件
- 配置模板
