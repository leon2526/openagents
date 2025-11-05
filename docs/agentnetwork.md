# AgentNetwork 类 Wiki

## 概览
`AgentNetwork` 是 OpenAgents 的核心网络协调类，负责根据配置初始化网络拓扑、管理代理注册和认证、调度事件总线以及提供工作区访问入口。类定义位于 `src/openagents/core/network.py` 中。其主要职责包括：

- 依据 `NetworkConfig` 加载网络参数与拓扑实现。
- 管理网络生命周期（初始化、关闭）。
- 为代理提供注册、注销与身份验证能力。
- 提供事件网关以统一处理外部与内部事件。
- 暴露工作区接口以支持线程式协作能力。

## 初始化流程
创建 `AgentNetwork` 实例时，需要传入 `NetworkConfig` 配置对象以及可选的 `workspace_path`。构造函数会：

1. 保存网络名称与节点 ID，若未提供节点 ID 则自动生成。
2. 按需初始化工作区管理器，支持持久化工作区或临时工作区。
3. 根据配置模式选择中心化或去中心化拓扑，并调用 `create_topology` 创建拓扑实例。
4. 初始化模块、元数据、身份管理器、密钥管理器以及事件网关。

若需从配置文件或配置对象创建实例，可使用：

- `AgentNetwork.create_from_config(config, port=None, workspace_path=None)`：接受 `NetworkConfig` 对象，负责加载网络 Mod 并执行构造流程。
- `AgentNetwork.load(config=None, port=None, workspace_path=None)`：从 YAML 配置文件加载网络。支持传入路径字符串、`Path` 对象或在 `workspace_path` 中自动发现 `network.yaml`。

## 关键属性
- `topology`：由 `create_topology` 返回的拓扑对象，实现代理连接管理与消息路由。
- `mods`：`OrderedDict`，存储已加载的网络 Mod 实例。
- `event_gateway`：事件总线接口，负责事件订阅、分发和处理。
- `identity_manager`：代理身份认证组件。
- `secret_manager`：生成与验证代理密钥，用于事件认证。

## 核心方法
| 方法 | 功能摘要 |
| --- | --- |
| `initialize()` | 异步初始化拓扑、注册内部事件处理器并标记网络为运行状态。 |
| `shutdown()` | 异步关闭拓扑并清理运行状态。 |
| `register_agent(agent_id, transport_type, metadata, certificate, force_reconnect=False, password_hash=None)` | 注册代理，生成密钥，通知 Mod 并返回 `EventResponse`。 |
| `unregister_agent(agent_id)` | 注销代理，清理密钥与事件队列并广播通知。 |
| `get_agent_registry()` / `get_agent(agent_id)` | 查询当前已注册的代理信息。 |
| `get_network_stats()` | 汇总网络运行时间、代理分组等统计信息。 |
| `process_external_event(event)` / `process_event(event)` | 处理外部或内部事件，请求通过 `event_gateway` 统一调度。 |
| `workspace(client_id=None)` | 创建并返回绑定网络的 `Workspace` 实例，自动配置连接参数。 |

此外，模块还提供 `create_network(config)` 工具函数，用于根据配置类型自动调用 `create_from_config` 或 `load`。

## 事件与认证
- `AgentNetwork` 通过 `EventGateway` 统一对接系统事件，如代理注册、注销、消息轮询等。
- `_validate_event_authentication(event)` 利用 `secret_manager` 校验事件携带的密钥，确保来源可信。
- `emit_to_event_bus(event)` 为未来的统一事件系统预留接口，实现事件广播。

## 工作区支持
调用 `workspace(client_id=None)` 时：

1. 检查网络是否启用了 `workspace.default` Mod，否则抛出异常提示配置要求。
2. 创建 `AgentClient` 与 `Workspace` 实例，并为调用方准备自动连接配置（复用网络 host/port）。
3. 返回 `Workspace` 对象，供上层在异步环境中建立连接并开展协作。

## 兼容性别名
为兼容旧代码，模块末尾暴露以下别名：

- `AgentNetworkServer = AgentNetwork`
- `EnhancedAgentNetwork = AgentNetwork`
- `create_enhanced_network = create_network`

这些别名允许旧版引用继续工作，同时引导迁移到统一的 `AgentNetwork` 接口。
