# 签到

如网页源码所示，填写错误时会跳转到 `/pass?=false`，正确时会跳转到 `/pass?=true`。

```javascript
function submitResult() {
    const inputs = document.querySelectorAll('.input-box');
    let allCorrect = true;

    inputs.forEach(input => {
        if (input.value !== answers[input.id]) {
            allCorrect = false;
            input.classList.add('wrong');
        } else {
            input.classList.add('correct');
        }
    });

    window.location = `?pass=${allCorrect}`;
}
```

所以直接在地址栏修改 `pass` 参数为 `true` 或手动发送 GET 请求就好：

```http
GET /?pass=true HTTP/1.1
```
