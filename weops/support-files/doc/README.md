## 嘉为蓝鲸keepalived插件使用说明

## 使用说明

### 插件功能

Keepalived Exporter 通过 **Unix 信号（Signal）** 与 Keepalived 进程通信，并解析 Keepalived 生成的临时文件来采集监控指标数据。

### 版本支持

操作系统支持: linux

是否支持arm: 支持

**组件支持版本：**

keepalived: 1.2.13 及以上版本

**是否支持远程采集:**

否

### 参数说明

| **参数名**                 | **含义**                                     | **默认值**                 | **是否必填** | **使用举例**                |
|-------------------------|--------------------------------------------|-------------------------|----------|-------------------------|
| --ka.pid-path           | Keepalived进程PID文件路径（主机模式）                  | /var/run/keepalived.pid | 否        | /var/run/keepalived.pid |
| --ka.json               | 使用JSON模式采集（需要Keepalived编译时启用--enable-json） | false                   | 否        | true                    |
| --web.listen-address    | Exporter监听的IP地址及端口                         | :9165                   | 否        | 127.0.0.1:9165          |

**注意**
`--ka.json` 参数需要 Keepalived 在编译时启用 --enable-json 选项  
JSON 模式仅支持 Keepalived 2.0.0 及以上版本  
如果使用较旧版本的 Keepalived（< 2.0.0），请使用默认的文本解析模式（不添加 --ka.json 参数, 默认使用false） 

### 使用指引

#### 1. 快速开始（主机模式）
前置条件：
- 已安装并运行 keepalived，PID 文件位于 /var/run/keepalived.pid（默认）
- 具备可访问 /tmp 目录权限，并且 keepalived 能在收到信号后生成 /tmp/keepalived.data 和 /tmp/keepalived.stats


#### 主机模式
1. 确认 keepalived 已运行：`systemctl status keepalived` 或 `ps -ef | grep keepalived`
2. 确认 PID 文件存在：`ls -l /var/run/keepalived.pid`
3. 运行 exporter（无需 root，但需要读 PID 文件和 /tmp 文件权限）


#### JSON 模式使用说明
- 仅在 Keepalived 编译时加 --enable-json 并且版本 ≥ 2.0.0 才可用。
- 启用后只发送 JSON 信号生成 /tmp/keepalived.json，由 exporter 直接解析，不再解析文本。

失败场景：
- 不支持 JSON 时会日志提示并退出：请删除 --ka.json 参数改用文本模式。



#### 常见问题排查
| 问题表现            | 可能原因                 | 排查步骤                                     | 解决方案                               |
|-----------------|----------------------|------------------------------------------|------------------------------------|
| keepalived_up=0 | 无法读取 /tmp 文件         | 查看日志中 "No data found"                    | 确认信号可触发文件生成，检查权限                   |
| 指标缺失（无广告报文计数）   | keepalived.stats 未更新 | 查看文件时间戳                                  | 确认 SIGUSR2 信号发送成功                  |
| JSON 模式启动失败     | 版本或编译不支持             | `keepalived --version` 内容无 --enable-json | 改用文本模式                             |
| 进程信号发送失败        | PID 文件路径错误           | 检查 --ka.pid-path 与实际文件                   | 修正参数或建立软链接                         |
| exporter 频繁重启   | 信号执行超时或报错退出          | 查看 systemd / 日志                          | 降低采样频率（Prometheus scrape_interval） |



### 指标简介
| **指标分类**             | **指标ID**                                        | **指标中文名**          | **维度ID**                      | **维度含义**                   | **单位** | **指标类型** |
|----------------------|-------------------------------------------------|--------------------|-------------------------------|----------------------------|--------|----------|
| 基础(Base)             | keepalived_up                                   | 监控插件运行状态           | -                             | -                          | -      | gauge    |
| 状态(Status)           | keepalived_vrrp_state                           | VRRP当前状态           | iname, ip_address, intf, vrid | 实例名称, 虚拟IP地址, 网络接口, 虚拟路由ID | -      | gauge    |
| 状态(Status)           | keepalived_become_master_total                  | 切换为MASTER的次数       | intf, iname, vrid             | 网络接口, 实例名称, 虚拟路由ID         | -      | counter  |
| 状态(Status)           | keepalived_release_master_total                 | 释放MASTER状态次数       | intf, iname, vrid             | 网络接口, 实例名称, 虚拟路由ID         | -      | counter  |
| 广告报文(Advertisements) | keepalived_advertisements_interval_errors_total | 广告报文间隔错误次数         | intf, iname, vrid             | 网络接口, 实例名称, 虚拟路由ID         | -      | counter  |
| 广告报文(Advertisements) | keepalived_advertisements_received_total        | 接收的VRRP广告报文总数      | intf, iname, vrid             | 网络接口, 实例名称, 虚拟路由ID         | -      | counter  |
| 广告报文(Advertisements) | keepalived_advertisements_sent_total            | 发送的VRRP广告报文总数      | intf, iname, vrid             | 网络接口, 实例名称, 虚拟路由ID         | -      | counter  |
| 认证(Authentication)   | keepalived_authentication_failure_total         | 认证失败次数             | intf, iname, vrid             | 网络接口, 实例名称, 虚拟路由ID         | -      | counter  |
| 认证(Authentication)   | keepalived_authentication_invalid_total         | 无效认证包次数            | intf, iname, vrid             | 网络接口, 实例名称, 虚拟路由ID         | -      | counter  |
| 认证(Authentication)   | keepalived_authentication_mismatch_total        | 认证信息不匹配次数          | intf, iname, vrid             | 网络接口, 实例名称, 虚拟路由ID         | -      | counter  |
| 优先级(Priority)        | keepalived_priority_zero_received_total         | 接收到优先级为0的包次数       | intf, iname, vrid             | 网络接口, 实例名称, 虚拟路由ID         | -      | counter  |
| 优先级(Priority)        | keepalived_priority_zero_sent_total             | 发送优先级为0的包次数        | intf, iname, vrid             | 网络接口, 实例名称, 虚拟路由ID         | -      | counter  |
| 其他(Other)            | keepalived_gratuitous_arp_delay_total           | Gratuitous ARP延迟次数 | intf, iname, vrid             | 网络接口, 实例名称, 虚拟路由ID         | -      | counter  |
| 其他(Other)            | keepalived_invalid_type_received_total          | 接收到的无效类型包次数        | intf, iname, vrid             | 网络接口, 实例名称, 虚拟路由ID         | -      | counter  |
| 其他(Other)            | keepalived_ip_ttl_errors_total                  | TTL错误次数            | intf, iname, vrid             | 网络接口, 实例名称, 虚拟路由ID         | -      | counter  |
| 其他(Other)            | keepalived_packet_length_errors_total           | 报文长度错误次数           | intf, iname, vrid             | 网络接口, 实例名称, 虚拟路由ID         | -      | counter  |
| 其他(Other)            | keepalived_address_list_errors_total            | 地址列表错误次数           | intf, iname, vrid             | 网络接口, 实例名称, 虚拟路由ID         | -      | counter  |


### 版本日志

#### weops_keepalived_exporter 1.7.0

- weops调整
