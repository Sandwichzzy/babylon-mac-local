# babylon Mac 环境本地网络启动

## 项目简介

babylon-mac-local 是一个用于在 macOS 上运行完整 Babylon 区块链测试网络的本地开发编排框架。它通过 git submodules 管理 13+ 个服务组件，包括 Babylon 节点、Bitcoin regtest 网络、vigilante 服务、finality providers、BTC staking 和 IBC relayer。

## 1. 依赖安装

### 1.1 系统工具安装

安装依赖的工具最小集合到你系统 brew：
```bash
HOMEBREW_NO_AUTO_UPDATE=1 brew install jq yq sha3sum go-task supervisor gvm bitcoin
```

**需要特别注意的版本要求：**
- supervisord 需要 4.2.5 版本
- bitcoind v28.0.0

### 1.2 Go 版本管理

建议使用 gvm 管理 go 版本：
```bash
gvm install go1.23.3
gvm use go1.23.3
```

### 1.3 配置 CODE_BASE 路径

**重要：** 在首次运行前，必须修改 `lib/utils.sh` 文件第 3 行的 `CODE_BASE` 变量为你的本地仓库路径：
```bash
export CODE_BASE=/your/path/to/babylon-mac-local
```

## 2. 项目启动

### 2.1 下载项目
```bash
git clone git@github.com:dapplink-labs/babylon-mac-local.git
cd babylon-mac-local
```

### 2.2 拉取依赖项目
```bash
git submodule update --init --recursive
```

### 2.3 完整启动流程

按照以下顺序执行命令：

1. **构建所有组件**
```bash
task build-all
```

2. **初始化网络配置**
```bash
task init-network-conf
```

3. **启动整个网络**
```bash
task up-babylon-network
```

4. **初始化比特币网络**
```bash
task init-bitcoin
```

5. **部署 Babylon 合约**（如需使用 IBC）
```bash
task deploy_babylon_contract
```

6. **初始化 IBC Relayer**（如需使用 IBC）
```bash
task init-relayer
```

7. **启动 Relayer**（如需使用 IBC）
```bash
task start-relayer
```

## 3. 可用命令详解

### 3.1 构建相关

#### `task build-all`
**作用：** 构建所有 Babylon 网络组件的二进制文件

**构建的组件：**
- babylon - Babylon 区块链主节点
- btc-staker - BTC 质押服务
- covenant-emulator - Covenant 签名模拟器
- finality-gadget - Finality 机制
- finality-provider - Finality 提供者
- vigilante - Bitcoin 监控服务

**执行时间：** 首次构建可能需要 5-10 分钟

**别名：** `task build-all`

### 3.2 网络管理

#### `task init-network-conf`
**作用：** 初始化 Babylon 网络配置文件

**执行内容：**
- 创建 `testnets/` 目录结构
- 初始化 2 个 Babylon 节点（node0, node1）
- 生成测试用的 keyring
- 配置所有服务的配置文件
- 设置 Bitcoin regtest 网络参数
- 配置 vigilante 的 4 个角色（reporter, submitter, monitor, bstracker）
- 初始化 EOTS managers 和 finality providers

**重要参数：**
- Chain ID: `chain-test`
- Epoch interval: 10 blocks
- BTC confirmation depth: 1
- BTC finalization timeout: 2

**别名：** `task init-babylon-config`

#### `task up-babylon-network`
**作用：** 启动整个 Babylon 网络的所有服务

**启动的服务（13个）：**
1. **babylondnode0** - Babylon 节点 0 (RPC: 26657)
2. **babylondnode1** - Babylon 节点 1 (RPC: 26667)
3. **bitcoindsim** - Bitcoin regtest 网络 (RPC: 18443)
4. **vigilante-reporter** - 向 Babylon 报告 Bitcoin 数据 (gRPC: 8080)
5. **vigilante-submitter** - 向 Bitcoin 提交检查点 (gRPC: 8081)
6. **vigilante-monitor** - 监控检查点最终确认 (gRPC: 8082)
7. **vigilante-bstracker** - 追踪 BTC 质押交易 (gRPC: 8083)
8. **btc-staker** - BTC 质押服务 (RPC: 15812)
9. **finality-provider** - 主 Finality 提供者 (RPC: 12581)
10. **consumer-fp** - Consumer 链 Finality 提供者 (RPC: 12582)
11. **eotsmanager** - EOTS 签名管理器 (RPC: 15813)
12. **consumer-eotsmanager** - Consumer EOTS 管理器 (RPC: 15814)
13. **covenant-emulator** - Covenant 模拟器 (Metrics: 4116)

**进程管理：** 使用 supervisord 管理所有服务，支持自动重启

**Web 界面：** http://127.0.0.1:39001

**别名：** `task start-babylon`, `task up-babylon`

#### `task ps`
**作用：** 查看所有服务的运行状态

**输出信息：**
- 服务名称
- 运行状态（RUNNING/STOPPED/FATAL）
- 进程 PID
- 运行时长

**别名：** `task status`

#### `task clean`
**作用：** 停止所有服务并删除所有配置和数据

**执行内容：**
1. 停止 supervisord 和所有子服务
2. 删除整个 `testnets/` 目录

**警告：** 此操作会删除所有链上数据和配置，无法恢复

**别名：** `task clean all data and services`

#### `task rm-all`
**作用：** 仅删除 testnets 目录中的配置和数据，不停止服务

**使用场景：** 当服务已经停止，只需要清理数据时使用

**别名：** `task clean all data in testnet dir`

### 3.3 Supervisord 管理

#### `task up-supervisord`
**作用：** 单独启动 supervisord 进程管理器

**使用场景：** 通常不需要单独调用，`up-babylon-network` 会自动启动

**配置文件：** `conf/supervisord/supervisord.ini`

**别名：** `task start-supervisord`

#### `task down-supervisord`
**作用：** 停止 supervisord 和所有管理的服务

**别名：** `task stop-supervisord`

### 3.4 Bitcoin 网络

#### `task init-bitcoin`
**作用：** 初始化 Bitcoin regtest 网络

**执行内容：**
- 创建默认钱包（wallet name: `default`）
- 创建 btcstaker 钱包（wallet name: `btcstaker`）
- 生成 3 个 btcstaker 地址
- 挖掘初始区块（生成测试 BTC）
- 向 btcstaker 地址转账

**前置条件：** 必须先执行 `task up-babylon-network` 启动 Bitcoin 节点

**RPC 配置：**
- 端口: 18443
- 用户: rpcuser
- 密码: rpcpass

**别名：** `task init-bitcoin`

### 3.5 IBC Relayer 相关

#### `task deploy_babylon_contract`
**作用：** 部署 Babylon 智能合约到 consumer 链

**部署的合约：**
1. babylon_contract.wasm - Babylon 主合约
2. btc_staking.wasm - BTC 质押合约

**配置参数：**
- Network: regtest
- BTC confirmation depth: 1
- Checkpoint finalization timeout: 2
- Consumer chain ID: bcd-test

**前置条件：**
- 网络已启动（`task up-babylon-network`）
- Bitcoin 已初始化（`task init-bitcoin`）

**别名：** `task deploy-babylon-contract`

#### `task init-relayer`
**作用：** 初始化 Cosmos IBC relayer 配置

**执行内容：**
- 配置 Babylon 链连接（chain-test）
- 配置 Consumer 链连接（bcd-test）
- 创建 IBC 通道
- 配置密钥和 RPC 端点

**前置条件：** 必须先执行 `task deploy_babylon_contract`

**别名：** `task init-relayer`

#### `task start-relayer`
**作用：** 启动 IBC relayer 进行跨链中继

**功能：**
- 在 Babylon 链和 Consumer 链之间中继 IBC 数据包
- 更新客户端状态（间隔: 20s）
- 处理跨链消息传递

**前置条件：** 必须先执行 `task init-relayer`

**别名：** `task start-relayer`

### 3.6 其他命令

#### `task list`
**作用：** 列出所有可用的 task 命令及其描述

## 4. 服务端口映射

### Babylon 节点
- node0 RPC: 26657
- node0 P2P: 26656
- node1 RPC: 26667
- node1 P2P: 26666

### Bitcoin
- RPC: 18443
- ZMQ Sequence: 29000
- ZMQ Raw Block: 29001
- ZMQ Raw Transaction: 29002

### Vigilante 服务
- Reporter gRPC: 8080, Metrics: 2112
- Submitter gRPC: 8081, Metrics: 2113
- Monitor gRPC: 8082, Metrics: 2114
- BStracker gRPC: 8083, Metrics: 2115

### Finality & Staking
- BTC Staker RPC: 15812
- Finality Provider RPC: 12581, Metrics: 4112
- Consumer FP RPC: 12582, Metrics: 4113
- EOTS Manager RPC: 15813, Metrics: 4114
- Consumer EOTS RPC: 15814, Metrics: 4115
- Covenant Emulator Metrics: 4116

### IBC Consumer Chain
- RPC: 36657
- P2P: 36656
- gRPC: 19090
- Profiling: 16060

### 监控
- Supervisord Web: 39001
- Script Exporter: 9469

## 5. 常见问题

### 5.1 服务无法启动
**检查项：**
1. 确认 `lib/utils.sh` 中的 `CODE_BASE` 路径正确
2. 确认所有端口未被占用
3. 确认 Go 版本为 1.23.3
4. 查看日志：`testnets/logs/`

### 5.2 构建失败
**解决方案：**
```bash
# 确认 Go 版本
gvm use go1.23.3
go version

# 清理后重新构建
task clean
task build-all
```

### 5.3 Relayer 无法启动
**检查顺序：**
1. 确认网络已启动：`task ps`
2. 确认合约已部署：`task deploy_babylon_contract`
3. 然后初始化：`task init-relayer`
4. 最后启动：`task start-relayer`

### 5.4 Bitcoin 节点无响应
**解决方案：**
```bash
# 确认 Bitcoin 服务运行中
task ps | grep bitcoindsim

# 重新初始化
task init-bitcoin
```

## 6. 开发提示

### 6.1 查看日志
所有服务日志位于：`testnets/logs/`
```bash
# 查看特定服务日志
tail -f testnets/logs/babylondnode0.log
tail -f testnets/logs/vigilante-reporter.log
```

### 6.2 重启单个服务
```bash
# 使用 supervisorctl
supervisorctl -c conf/supervisord/supervisord.ini restart babylondnode0
```

### 6.3 配置文件位置
- 模板配置：`conf/`
- 运行时配置：`testnets/`（由 init-network-conf 生成）

### 6.4 测试账户
所有服务使用 `--keyring-backend=test` 进行开发测试



