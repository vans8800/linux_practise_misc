## 背景信息

借助公网IP的构建内网穿透的方法

## 主体方法

命令 ssh -gN -R 7800:127.0.0.1:22 -p 24893 testuser@114.242.206.180的详细分析：

### 一、命令结构解析
​​
该命令通过 SSH 建立**​​远程端口转发隧道**​​，核心参数如下：

ssh [选项] -R [远程端口]:[本地地址]:[本地端口] [用户]@[远程主机]


### 二、参数逐项解释​​

1. ​​-g​​

• ​​作用​​：允许远程主机（114.242.206.180）上的其他设备通过该 SSH 连接访问本地转发的端口（7800）。

• 默认行为​​：若不加 -g，远程端口仅绑定到 127.0.0.1（本地回环地址），仅远程主机自身可访问。

• 典型场景​​：需让局域网内其他设备通过跳板机访问本地服务时使用。

2. ​​-N​​

• ​​作用​​：仅建立 SSH 隧道，​​不执行远程命令​​。

• ​用途​​：避免启动远程 Shell 会话，专注于端口转发功能。

3. ​​-R 7800:127.0.0.1:22​​

• ​​功能​​：将远程主机（114.242.206.180）的 7800端口转发到本地（客户端）的 127.0.0.1:22（SSH 服务）。

• ​​流量路径​​：

• 外部请求 → 远程主机 7800端口 → 本地 22端口 → 本地 SSH 服务响应。

• 关键限制​​：

• 远程主机需开启 GatewayPorts yes（默认 no），否则仅允许本地访问 7800端口。需修改 /etc/ssh/sshd_config并重启 SSH 服务。

4. ​​-p 24893​​

•​​作用​​：指定 SSH 连接的远程主机端口为 24893（默认 SSH 端口为 22）。

5. ​​testuser@114.242.206.180​​

•用户与主机​​：以 testuser身份登录 IP 为 114.242.206.180的远程主机。

### ​​三、命令功能总结​​

1. ​​建立反向隧道​​：

通过远程主机 114.242.206.180的 7800端口，暴露本地 SSH 服务（22端口），实现 ​​内网穿透​​。

• 外部用户可通过 ssh -p 7800 testuser@114.242.206.180访问本地 SSH 服务。--有待验证

2. 安全场景​​：

• 远程维护内网服务器（如无公网 IP 的数据库、开发环境）。

• 绕过防火墙限制，仅暴露 SSH 端口到公网。

### ​​四、使用注意事项​​

1. ​​远程主机配置​​：

• 需修改远程主机的 /etc/ssh/sshd_config，设置 GatewayPorts yes并重启 SSH 服务，否则 -g无效。

2. 本地服务状态​​：

• 确保本地 22端口（SSH 服务）正在运行，否则隧道建立失败。

3. ​​安全性风险​​：

• 公开暴露端口可能被恶意访问，建议结合密钥认证、防火墙规则限制访问 IP。

4. 后台运行​​：

• 若需保持隧道持久化，可添加 -f参数（后台运行）或使用 **autossh**工具自动重连。

### ​​五、扩展应用示例​​

• ​​多级转发​​：

通过跳板机建立多层隧道，例如：

```bash
ssh -gN -R 7800:localhost:22 -p 24893 user@jump-host
ssh -gN -R 8080:localhost:7800 -p 22 user@internal-server
```

实现从公网到内网的多层穿透。

• 动态端口转发​​：

若需代理所有流量（如浏览器访问外网），可使用 -D参数创建 SOCKS 代理。

### ​​六、验证隧道是否生效​​

1. ​​远程主机检查​​：

netstat -tuln | grep 7800  # 确认端口已监听

2. ​​本地访问测试​​：

ssh -p 7800 testuser@114.242.206.180  # 从另一台机器测试连接

通过该命令，可灵活实现内网服务的外网暴露，是远程管理和安全通信的重要工具。

## 秘钥生成及公开公钥
---

### 一、核心原理​​

SSH 密钥对认证基于非对称加密：

1. ​​本地生成密钥对​​：包含私钥（id_rsa）和公钥（id_rsa.pub）。

2. ​公钥部署到远程服务器​​：将公钥内容添加到远程主机的 ~/.ssh/authorized_keys文件中。

3. 认证流程​​：

• 本地客户端使用私钥生成数字签名。

• 远程服务器用公钥验证签名，若匹配则允许登录。

### ​​二、操作步骤​​

1. ​​生成 SSH 密钥对（本地操作）​​
ssh-keygen -t ed25519 -C "your_email@example.com"

注意： ed25519 密钥固定长度为256bits，所以-b 选项对该类型密钥无效。
​​参数说明​​：

• -t ed25519：指定密钥类型（推荐 Ed25519，安全性高）。

• -C：添加注释（可选）。

• 交互提示​​：

• 直接按回车使用默认密钥路径（~/.ssh/id_ed25519）。

• 可设置密钥密码短语（可选，增强安全性）。

2. ​​将公钥复制到远程服务器​​

ssh-copy-id -p 24893 testuser@114.242.206.180

 ​​参数说明​​：

• -p 24893：指定远程 SSH 端口。

• 自动将本地公钥追加到远程主机的 ~/.ssh/authorized_keys。

3. ​​验证免密登录​​

ssh -p 24893 testuser@114.242.206.180

•若配置成功，直接进入远程 Shell，无需输入密码。

### ​​三、常见问题排查​​

1. ​​权限问题​​

• ​​远程主机权限要求​​：

• ~/.ssh目录权限必须为 700。

• ~/.ssh/authorized_keys文件权限必须为 600。

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

2. ​​SSH 服务配置​​

• ​​启用公钥认证​​：

编辑远程主机的 /etc/ssh/sshd_config，确保以下配置生效：

```bash
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
```

• ​​重启 SSH 服务​​：

```bash
systemctl restart sshd  # CentOS/RHEL
service ssh restart     # Ubuntu/Debian
```

3. ​​密钥格式兼容性​​

• 若使用非默认密钥类型（如 RSA），需在 SSH 命令中显式指定私钥路径：

```bash
ssh -i ~/.ssh/id_rsa -p 24893 testuser@114.242.206.180
```

### ​​四、高级场景​​

1. ​​多密钥管理​​

•​​为不同主机配置不同密钥​​：

编辑本地 ~/.ssh/config文件：

```bash
Host server1
  HostName 192.168.1.100
  User user1
  IdentityFile ~/.ssh/id_rsa_server1

Host server2
  HostName 10.0.0.1
  User user2
  IdentityFile ~/.ssh/id_rsa_server2
```

2. ​​自动化脚本集成​​

•​​在脚本中免密执行远程命令​​：

```bash
ssh -p 24893 testuser@114.242.206.180 "ls -l /var/log"
```

### ​​五、安全性建议​​

1. ​​禁用密码登录​​（可选）：

修改远程主机的 /etc/ssh/sshd_config：

```bash
PasswordAuthentication no
```

重启 SSH 服务后，仅允许密钥登录。

2. ​​使用强密钥类型​​：

优先选择 ed25519（抗量子计算攻击）而非 RSA。

3. ​​限制密钥使用范围​​：

在 authorized_keys中为公钥添加限制选项：

```bash
from="192.168.1.0/24",command="/usr/bin/rsync" ssh-ed25519 AAAAC3NzaC1lZDI...
```
通过上述配置，SSH 命令可实现完全免密登录，适用于自动化运维、内网穿透等场景。若需进一步优化安全性，可结合密钥密码短语和访问控制策略。


## 内部机器网络配置

编辑/etc/ssh/sshd_config 
```bash
TCPKeepAlive yes
ClientAliveInterval 120
ClientAliveCountMax 30
```

### 笔者实践
```bash
X11Forwarding yes
X11DisplayOffset 10
#X11UseLocalhost yes
#PermitTTY yes
PrintMotd no
#PrintLastLog yes
TCPKeepAlive yes
ClientAliveInterval 120
ClientAliveCountMax 30
```
### continue

编辑$HOME/.ssh/config 
```bash
touch ~/.ssh/config 
cat ~/.ssh/config 
Host *
ServerAliveInterval 120
ServerAliveCountMax 30
```

