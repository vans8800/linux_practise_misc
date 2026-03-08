#  **JSON 快速入门手册：前端小白的第一个数据格式**

---



在 Web 开发中，**JSON (JavaScript Object Notation)** 是绝对的主角。它是前后端沟通的“通用语言”，也是配置文件的标准格式。掌握它，你就拿到了数据交互的钥匙。

------

## **1. 什么是 JSON？**

**JSON** 全称是 **J**ava**S**cript **O**bject **N**otation。

- **本质**：一种轻量级的**数据交换格式**。
- 特点：
  - **纯文本**：人类可读，机器易解析。
  - **语言无关**：虽然名字里有 JavaScript，但 Python、Java、Go、C# 等所有编程语言都能完美支持。
  - **结构简单**：只有两种结构：**对象**（键值对集合）和 **数组**（有序列表）。

> 💡 **一句话理解**：JSON 就是用来把数据打包成字符串，方便在网络上传输或保存到文件里。

------

## **2. JSON 的核心语法（只有 6 种数据类型）**

JSON 非常严格，记住这 **6 种** 允许的数据类型，你就掌握了 90%：

| 类型                | 描述         | 示例                | ⚠️ 注意事项                             |
| :------------------ | :----------- | :------------------ | :------------------------------------- |
| **String** (字符串) | 文本数据     | `"Hello World"`     | **必须**用双引号 `"`，不能用单引号 `'` |
| **Number** (数字)   | 整数或浮点数 | `42`, `3.14`, `-10` | 不能带引号；不能是 `NaN` 或 `Infinity` |
| **Boolean** (布尔)  | 真或假       | `true`, `false`     | 小写，不能带引号                       |
| **Null** (空值)     | 表示“无”     | `null`              | 小写，不能带引号                       |
| **Object** (对象)   | 键值对集合   | `{"name": "Alice"}` | 用 `{}` 包裹，键必须是**双引号**字符串 |
| **Array** (数组)    | 有序列表     | `[1, 2, 3]`         | 用 `[]` 包裹，元素可以是上述任意类型   |

### **❌ 常见错误（新手必看）**



```json
// 错误 1: 键名没有用双引号
{ name: "Alice" }  // ❌ 错
{ "name": "Alice" } // ✅ 对

// 错误 2: 使用了单引号
{ 'name': 'Alice' } // ❌ 错

// 错误 3: 尾部多了一个逗号
{ "name": "Alice", "age": 18, } // ❌ 错 (JSON 不允许尾随逗号)

// 错误 4: 使用了 undefined 或函数
{ "func": function() {} } // ❌ 错 (JSON 只存数据，不存代码)
```

------

## **3. 实战示例：一个完整的 JSON 结构**

想象我们要描述一个“用户信息”，包含基本信息、标签列表和地址对象。

```json
{
  "id": 1001,
  "username": "coder_xiao_bai",
  "isActive": true,
  "balance": 99.50,
  "tags": ["frontend", "json", "learner"],
  "address": {
    "city": "Shanghai",
    "zip": "200000",
    "coordinates": null
  },
  "favorites": [
    { "food": "Pizza", "score": 9 },
    { "food": "Sushi", "score": 8 }
  ]
}
```

**结构解析**：

1. 最外层是一个 **Object** `{...}`。
2. `tags` 是一个 **Array** `[...]`，里面全是字符串。
3. `address` 是一个嵌套的 **Object**。
4. `favorites` 是一个 **Array**，里面包含了两个 **Object**。
5. `coordinates` 的值是 `null`，表示未知。

------

## **4. 前端开发中的 JSON 操作（JavaScript）**

在前端（浏览器环境），你主要会用到两个全局方法：`JSON.parse()` 和 `JSON.stringify()`。

### **场景 A：后端发来了 JSON 字符串，我要变成 JS 对象**

当通过 `fetch` 或 `axios` 获取数据时，拿到的通常是字符串。

```javascript
const jsonString = '{"name": "Alice", "age": 25}';

// ❌ 错误做法：直接当对象用
// console.log(jsonString.name); // undefined

// ✅ 正确做法：解析 (Parse)
const userObj = JSON.parse(jsonString);

console.log(userObj.name); // 输出: "Alice"
console.log(typeof userObj); // 输出: "object"
```

### **场景 B：我要把 JS 对象发给后端，需要转成 JSON 字符串**

发送数据前，必须序列化。

```javascript
const data = {
  action: "login",
  payload: { id: 123, token: "abc-xyz" }
};

// ✅ 正确做法：序列化 (Stringify)
const jsonString = JSON.stringify(data);

console.log(jsonString); 
// 输出: "{\"action\":\"login\",\"payload\":{\"id\":123,\"token\":\"abc-xyz\"}}"

// 发送给服务器
// fetch('/api', { body: jsonString ... })
```

### **💡 格式化技巧（调试神器）**

直接在控制台打印 JSON 字符串是一团乱麻？使用 `JSON.stringify` 的第二个和第三个参数进行美化：

```javascript
const uglyJson = '{"user":{"name":"Bob","hobbies":["coding","gaming"]}}';
const obj = JSON.parse(uglyJson);

// 参数说明：(对象, 替换函数, 缩进空格数)
const prettyJson = JSON.stringify(obj, null, 2);

console.log(prettyJson);
/* 输出：
{
  "user": {
    "name": "Bob",
    "hobbies": [
      "coding",
      "gaming"
    ]
  }
}
*/
```

------

## **5. JSON vs JavaScript 对象：有什么区别？**

很多小白容易混淆这两者，请记住：

| 特性         | JSON (数据格式)                                           | JavaScript Object (代码结构)                   |
| :----------- | :-------------------------------------------------------- | :--------------------------------------------- |
| **键名引号** | **必须**双引号 `"key"`                                    | 可以省略引号 `key` (如果符合变量命名规则)      |
| **值的内容** | 只能是数据 (String, Number, Array, Object, Boolean, Null) | 可以是任何东西 (函数, Date, undefined, Symbol) |
| **注释**     | **不支持**注释                                            | 支持 `//` 和 `/* */` 注释                      |
| **尾部逗号** | **禁止** `,`                                              | 允许 (ES5+)                                    |
| **存在形式** | 字符串 (String)                                           | 内存中的对象实例                               |

> **记忆口诀**：JSON 是死板的字符串，只能存数据；JS 对象是灵活的代码，能存函数和逻辑。

------

## **6. 常用工具推荐**

不要手写 JSON！容易出错。使用以下工具辅助：

1. VS Code 插件:
   - 安装 **Prettier**：自动格式化 JSON 文件，修复尾部逗号等错误。
   - 安装 **JSON Tools**：提供一键格式化、验证功能。
2. 在线验证器:
   - [JSONLint](https://jsonlint.com/)：粘贴你的 JSON，它会告诉你哪里错了。
   - [JSON Formatter](https://jsonformatter.org/)：美化及查看结构。
3. 浏览器开发者工具:
   - 在 Chrome DevTools 的 Network 面板点击响应内容，选择 "Preview" 或 "Response"，浏览器会自动渲染成漂亮的树状 JSON 结构。

------

## **7. 给小白的最后建议**

1. **双引号是铁律**：写 JSON 时，养成肌肉记忆，键和字符串值永远用双引号 `"`。
2. **小心尾部逗号**：复制粘贴代码时，最容易在多行的最后一项后面多加一个逗号，这是 JSON 报错的头号原因。
3. **不要用 JSON 存日期**：JSON 没有 Date 类型。通常存为字符串 `"2023-10-01T12:00:00Z"`，拿到后再在 JS 里 `new Date()` 转换。
4. **安全性**：永远不要使用 `eval()` 来解析 JSON！一定要用 `JSON.parse()`。`eval()` 会执行恶意代码，极其危险。

------

🎉 **恭喜！** 你已经掌握了 JSON 的核心。现在你可以自信地去阅读 API 文档、配置项目文件，或者处理前后端数据交互了。去试着写一个自己的 JSON 配置文件吧！



## 8.基于shell终端执行js脚本

JavaScript 完全可以脱离 HTML，直接在 Linux Shell 终端中运行。你只需要安装一个 **JavaScript 运行时环境**。

目前最主流的选择是 **Node.js**。它让 JavaScript 变成了像 Python、Bash 一样的脚本语言，可以直接在终端执行 `.js` 文件。

```bash
node json_demo.js
```

