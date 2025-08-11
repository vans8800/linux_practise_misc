JSON（JavaScript Object Notation）是一种轻量级的数据交换格式，其数据类型设计简洁且通用。

以下是 JSON 支持的 ​​常见数据类型​​ 及其核心规则和示例：

## ​​一、简单数据类型（Primitive Types）​​
---

### 1. ​​字符串（String）​​

​​规则​​：必须用**双引号**包裹，支持 Unicode 字符和转义字符（如 `、\t、"` 等）。示例​​：

```json
{ "name": "Alice", "description": "A \"special\" user" }
```

### 2. ​​数字（Number）​​
​​
规则​​：支持整数和浮点数，不区分类型（如 42和 3.14），不支持八进制、十六进制或 NaN。 ​示例​​：

```bash
{ "age": 30, "price": 9.99, "temperature": -5.5 }
```

### 3. ​​布尔值（Boolean）​​

​​规则​​：仅有两个值：true或 false，区分大小写。 ​​示例​​：

```bash
{ "is_student": false, "is_active": true }
```

### 4. ​​空值（Null）​​

​​规则​​：表示“无值”，用 null表示，与 JavaScript 的 undefined不同。

​​示例​​：

```
{ "address": null }
```

## ​​二、复杂数据类型（Complex Types）​​
---

### 1. ​​数组（Array）​​

​​规则​​：**有序值**集合，用方括号 []包裹，元素间用逗号分隔，支持混合类型（但通常建议同类型）。​​示例​​：

```json
{ "scores": [90, 85.5, true, null] }
```

### 2. ​​对象（Object）​​

​​规则​​：无序键值对集合，用花括号 {}包裹，键必须是字符串（双引号），值可以是任意类型。 ​​示例​​：

```json
{
  "person": {
    "name": "Bob",
    "hobbies": ["reading", "sports"]
  }
}
```

## 三、关键特性与注意事项​​
​​---

### 类型严格性​​

- JSON 不支持 undefined、Date对象或函数，这些需转换为字符串或其他兼容类型。

- 日期通常以字符串格式表示（如 "2025-08-11T09:13:19Z"）。
​​
### 嵌套结构​​

对象和数组可以相互嵌套，形成复杂数据结构：

```json
{
  "users": [
    { "id": 1, "profile": { "age": 25 } },
    { "id": 2, "profile": { "age": 30 } }
  ]
}
```

### ​​转义字符​​

字符串中的特殊字符需转义，例如双引号 \"、反斜杠 \\等。

## ​​四、常见误区​​
---

- ​​单引号问题​​：JSON 字符串必须用双引号，单引号会导致解析失败。
​​
- 尾逗号问题​​：对象或数组的最后一个元素后不能有逗号：

```json
// 错误示例
{ "keys": ["a", "b",] }
```

## ​​五、实际应用场景
​​
### ​​API 数据交换
​​
服务器返回 JSON 格式数据，如：

```json
{ "status": "success", "data": { "id": 123 } }
```
​​
### 配置文件​​
使用 JSON 存储应用配置：

```json
{ "database": { "host": "localhost", "port": 3306 } }
```
​​
### 前端与后端通信​
​
前端通过 JSON 传递表单数据：
```json
{ "username": "user1", "password": "secret" }
```

## ​​总结​
---
​
JSON 的数据类型设计简洁，兼顾了可读性和跨语言兼容性。

掌握其核心类型（字符串、数字、布尔、数组、对象、null）及嵌套规则，是进行数据交换和配置管理的基础。实际开发中需注意类型转换和格式规范（如双引号、转义字符等）。


## JSON 与 Docker
---

docker inspect test1 | jq '.[].NetworkSettings.Networks'是用于获取 Docker 容器 test1的网络配置信息，并通过 jq工具格式化输出。

以下是该命令的解析、使用场景及可能的问题排查方法：

### ​​命令解析
​​
- ​​docker inspect test1​​该命令会返回容器 test1的完整配置信息（JSON 格式），包括网络、存储、运行状态等。
​​
- jq '.[].NetworkSettings.Networks'​​使用 jq工具解析 JSON 数据，提取容器的网络配置。

  - .表示根对象
  
  - []遍历数组（如果有多个网络配置）

NetworkSettings.Networks是 Docker 容器网络配置的字段路径。​​典型输出示例​​

```json
{
  "bridge": {
    "IPAMConfig": null,
    "Links": null,
    "Aliases": null,
    "NetworkID": "ffe9bc57a881ca79a60126b552dba79ad4f4c8784c55f0118f0c61c97d2d5de7",
    "EndpointID": "872b7a7cd16e483a634e5853e645aff4aa324d389845670e3c225fedae7738e7",
    "Gateway": "172.17.0.1",
    "IPAddress": "172.17.0.2",
    "IPPrefixLen": 16,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "MacAddress": "02:42:ac:11:00:02",
    "DriverOpts": null
  }
}
```

### ​​关键字段​​：

- IPAddress: 容器在 Docker 网络中的 IP 地址（默认桥接网络为 172.17.0.0/16）。

- Gateway: 网关地址（默认指向宿主机的 docker0网桥 IP）。

- MacAddress: 容器的 MAC 地址。

- NetworkID: 网络的唯一标识符。

### ​​使用场景​​
​
​网络调试​​

- 检查容器是否分配了 IP 地址。

- 验证网关是否正确（默认应指向宿主机的 docker0网桥）。
​​
- 多网络配置​​如果容器连接到多个网络（如自定义网络），会显示多个网络配置项，例如：

```bash
{
  "bridge": { ... },
  "custom_network": { ... }
}
```

​​自动化脚本​​结合 jq提取特定字段（如 IP 地址）用于脚本逻辑：

```bash
docker inspect test1 | jq -r '.[].NetworkSettings.Networks.bridge.IPAddress'
```
​​
### 常见问题排查​​

**​​容器不存在或未运行​​**

错误提示：Error response from daemon: No such container: test1

解决方法：检查容器名称或 ID 是否正确，运行 docker ps -a确认容器状态。

**​​网络未正确配置​​**

输出为空或 null：可能容器未成功连接到网络。

解决方法：重启容器或检查网络配置（如 docker network connect）。

**​​jq解析失败​​**

错误提示：jq: error (at <stdin>:1): Cannot index array with string "NetworkSettings"

解决方法：确保 jq已安装，并正确处理 JSON 结构（可能需要调整过滤器）。

**​​扩展命令**
​​
​​查看所有网络信息​​

```bash
docker network inspect bridge
```

​​查看容器 IP 地址（简化版）​​

```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' test1
```

​​结合 grep快速过滤​​

```bash
docker inspect test1 | grep -A 10 "NetworkSettings"
```

## JSON 根对象
---

在 Docker 的 docker inspect命令和 jq工具中，​​根对象（Root Object）​​ 是 JSON 数据结构的顶层入口，所有字段和嵌套结构均从根对象开始。以下是具体解析：

### 1. 根对象的定义​​

​​根对象​​ 是 JSON 数据的最外层对象，由 docker inspect返回的完整元数据构成。例如，运行 docker inspect test1会返回类似以下结构的 JSON：

{
  "Id": "abc123",
  "Name": "test1",
  "NetworkSettings": {
    "Networks": {
      "bridge": { ... },
      "custom_network": { ... }
    }
  }
}

此时，最外层的整个 JSON 对象就是根对象。

### ​​2. 根对象的作用
​​
​​访问所有字段的起点​​无论是容器、镜像还是网络，docker inspect返回的 JSON 数据均以根对象为起点。例如：

- .表示根对象本身。

- .Id访问根对象的 Id字段。

- .NetworkSettings.Networks访问根对象下的嵌套字段。
​​
处理嵌套数据的关键​​Docker 的网络配置通常嵌套在多层结构中（如 NetworkSettings.Networks），根对象是定位这些字段的起点。

### ​​3. 在 jq中使用根对象​​
​
​直接引用根对象​​在 jq过滤器中，.始终指向当前处理的 JSON 对象的根。例如：


**输出根对象的全部内容**

```bash
docker inspect test1 | jq '.'
```

​​逐层访问嵌套字段​​通过 .连接符逐级深入：

**提取容器在 bridge 网络中的 IP 地址**

docker inspect test1 | jq '.[].NetworkSettings.Networks.bridge.IPAddress'

这里的路径解析为：

- .→ 根对象
- [0](@ref)→ 根对象下的第一个容器（若有多个容器）

  - .NetworkSettings→ 根对象的 NetworkSettings字段

  - .Networks→ NetworkSettings的 Networks字段

  - .bridge→ Networks中的 bridge网络配置

  - .IPAddress→ 最终的 IP 地址。
​​
### 4. 常见误区​​


已知信息：

```json
[root@localhost aippc]# docker inspect redis | jq '.[].NetworkSettings.Networks'
{
  "searxng-docker_searxng": {
    "IPAMConfig": null,
    "Links": null,
    "Aliases": [
      "redis",
      "688fc592057f"
    ],
    "NetworkID": "d85178d67dd29f16e05f2db16b37ca6b37e61783a33114f4f615bd13bd2b4dc1",
    "EndpointID": "1613cbc7a07fd87635a34420329d0e36021dd1e5460af5083646bc115d85bdab",
    "Gateway": "172.25.0.1",
    "IPAddress": "172.25.0.3",
    "IPPrefixLen": 16,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "MacAddress": "02:42:ac:19:00:03",
    "DriverOpts": null
  }
}
```


```bash
[root@localhost aippc]# docker inspect redis | jq '.[].NetworkSettings.Networks | to_entries[]'
{
  "key": "searxng-docker_searxng",
  "value": {
    "IPAMConfig": null,
    "Links": null,
    "Aliases": [
      "redis",
      "688fc592057f"
    ],
    "NetworkID": "d85178d67dd29f16e05f2db16b37ca6b37e61783a33114f4f615bd13bd2b4dc1",
    "EndpointID": "1613cbc7a07fd87635a34420329d0e36021dd1e5460af5083646bc115d85bdab",
    "Gateway": "172.25.0.1",
    "IPAddress": "172.25.0.3",
    "IPPrefixLen": 16,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "MacAddress": "02:42:ac:19:00:03",
    "DriverOpts": null
  }
}
[root@localhost aippc]# docker inspect redis | jq '.[].NetworkSettings.Networks | to_entries[] | .value'
{
  "IPAMConfig": null,
  "Links": null,
  "Aliases": [
    "redis",
    "688fc592057f"
  ],
  "NetworkID": "d85178d67dd29f16e05f2db16b37ca6b37e61783a33114f4f615bd13bd2b4dc1",
  "EndpointID": "1613cbc7a07fd87635a34420329d0e36021dd1e5460af5083646bc115d85bdab",
  "Gateway": "172.25.0.1",
  "IPAddress": "172.25.0.3",
  "IPPrefixLen": 16,
  "IPv6Gateway": "",
  "GlobalIPv6Address": "",
  "GlobalIPv6PrefixLen": 0,
  "MacAddress": "02:42:ac:19:00:03",
  "DriverOpts": null
}
```


​​混淆数组与对象​​如果 Networks是一个对象（键值对形式），直接使用 .Networks会返回整个对象；若需遍历其值，需用 .或 to_entries[]：

**遍历所有网络配置**

```bash
docker inspect test1 | jq '.[].NetworkSettings.Networks | to_entries[] | .value'
```

多容器场景​​当检查多个容器时，jq的 []会遍历根对象数组中的每个容器：提取所有容器的 bridge 网络 IP

```bash
docker inspect $(docker ps -aq) | jq '.[].NetworkSettings.Networks.bridge.IPAddress'
```

### 5. 实际应用示例​​


​​场景1 ：提取容器 IP 地址​​

**命令**

```bash
docker inspect test1 | jq -r '.[].NetworkSettings.Networks.bridge.IPAddress'
```

**解析**

- .[] → 遍历根对象数组（若有多个容器）

- .NetworkSettings → 根对象的 NetworkSettings 字段

- .Networks.bridge → bridge 网络的配置

- .IPAddress → 最终的 IP 地址

​​场景2 ：查看所有网络配置​​

```bash
docker inspect test1 | jq '.NetworkSettings.Networks'
```

```bash

# 输出：
{
  "bridge": { ... },
  "custom_network": { ... }
}
```

场景3： 提起IP地址

```bash
[root@localhost aippc]# docker inspect redis | jq -r '.[].NetworkSettings.Networks | to_entries[]'
{
  "key": "searxng-docker_searxng",
  "value": {
    "IPAMConfig": null,
    "Links": null,
    "Aliases": [
      "redis",
      "688fc592057f"
    ],
    "NetworkID": "d85178d67dd29f16e05f2db16b37ca6b37e61783a33114f4f615bd13bd2b4dc1",
    "EndpointID": "1613cbc7a07fd87635a34420329d0e36021dd1e5460af5083646bc115d85bdab",
    "Gateway": "172.25.0.1",
    "IPAddress": "172.25.0.3",
    "IPPrefixLen": 16,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "MacAddress": "02:42:ac:19:00:03",
    "DriverOpts": null
  }
}
```

```bash
[root@localhost aippc]# docker inspect redis | jq  '.[].NetworkSettings.Networks | to_entries[].value.IPAddress'
"172.25.0.3"
[root@localhost aippc]# docker inspect redis | jq  -r '.[].NetworkSettings.Networks | to_entries[].value.IPAddress'
172.25.0.3
```

### ​​总结​
​
- ​​根对象是 JSON 数据的入口​​，所有字段均从根对象开始访问。

- 在 jq中，.表示根对象，通过 .连接符可逐层提取嵌套字段。

- 理解根对象的结构对解析复杂 JSON（如 Docker 网络配置）至关重要。



## JSON to_entries 和 with_entries
---

to_entries是 jq中的一个核心函数，其作用是将 ​​对象（Object）转换为键值对数组（Array of Key-Value Pairs）​​，从而简化对嵌套 JSON 数据的遍历和操作。以下是具体解析：

### 1. 核心功能​
​

​​输入对象​​：假设存在如下 JSON 对象：

```json
{
  "network1": { "ip": "172.18.0.3", "gateway": "172.18.0.1" },
  "network2": { "ip": "10.0.0.2", "gateway": "10.0.0.1" }
}
```
​​输出结果​​：

```json
[
  { "key": "network1", "value": { "ip": "172.18.0.3", "gateway": "172.18.0.1" } },
  { "key": "network2", "value": { "ip": "10.0.0.2", "gateway": "10.0.0.1" } }
]
```

每个元素包含 key（原对象的键）和 value（原对象的值）。

### ​​2. 在 Docker 网络信息中的应用​​

在用户之前的命令中，Networks字段的结构是对象（键为网络名称，值为网络配置），例如：

```json
"Networks": {
  "searxng-docker_searxng": { "IPAddress": "172.25.0.3", ... },
  "bridge": { "IPAddress": "172.17.0.2", ... }
}
```

直接使用 .Networks.IPAddress会失败，因为 Networks是对象而非数组。

通过 to_entries[]将其转换为数组后，可遍历每个网络配置：

```bash
jq -r '.Networks | to_entries[] | .value.IPAddress'
```

输出：

172.25.0.3
172.17.0.2

### ​​3. 与 with_entries的区别​​

- ​​to_entries​​：仅将对象转换为键值对数组。

​​- with_entries​​：在转换的基础上允许对键或值进行修改，例如重命名键：

```bash
jq 'with_entries(.key |= "new_" + .)'  # 将键前缀添加 "new_"
```

### ​​4. 典型使用场景​​


场景 1：遍历对象的所有值


**提取所有网络配置的 IP 地址**

```bash
[root@localhost aippc]# docker inspect redis | jq  -r '.[].NetworkSettings.Networks | to_entries[].value.IPAddress'
172.25.0.3
```

场景 2：过滤特定键值对

**仅显示网关为 172.25.0.1 的网络**

```bash

[root@localhost aippc]# docker inspect redis | jq  -r '.[].NetworkSettings.Networks | to_entries[] |  select(.value.Gateway == "172.25.0.1")'
{
  "key": "searxng-docker_searxng",
  "value": {
    "IPAMConfig": null,
    "Links": null,
    "Aliases": [
      "redis",
      "688fc592057f"
    ],
    "NetworkID": "d85178d67dd29f16e05f2db16b37ca6b37e61783a33114f4f615bd13bd2b4dc1",
    "EndpointID": "1613cbc7a07fd87635a34420329d0e36021dd1e5460af5083646bc115d85bdab",
    "Gateway": "172.25.0.1",
    "IPAddress": "172.25.0.3",
    "IPPrefixLen": 16,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "MacAddress": "02:42:ac:19:00:03",
    "DriverOpts": null
  }
}
```

场景 3：合并多个对象

**合并两个对象的所有键值对**

```bash
jq -s 'add' obj1.json obj2.json
```

### ​​5. 为什么需要 to_entries？​​
​​
- JSON 结构限制​​：JSON 对象的键是无序的，且无法直接通过索引访问。
​​
- 遍历需求​​：jq的 []运算符对数组有效，对对象需先转换为数组。
​​
- 动态键处理​​：当键名未知或动态变化时（如 Docker 网络名称），to_entries提供了统一处理方式。

### 总结​​

to_entries是处理 JSON 对象的关键工具，通过将对象转换为键值对数组，使开发者能灵活遍历、过滤和操作嵌套数据。在 Docker 网络配置分析中，它解决了动态网络名称导致的路径访问难题。
