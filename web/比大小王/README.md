# 比大小王

## 网页源码分析

游戏状态保存在 `state` 变量中：

```javascript
let state = {
    allowInput: false,
    score1: 0, // 玩家分数
    score2: 0, // 对手分数
    values: null, // 服务器返回题目数字
    startTime: null, // 开始时间
    value1: null, // 正在进行比较的数字 1
    value2: null, // 正在进行比较的数字 2
    inputs: [], // 玩家输入的答案，由 "<" 和 ">" 字符串组成的数组
    stopUpdate: false,
};
```

页面加载时会调用 `loadGame` 函数向 `/game` 发送 POST 请求获取题目：

```javascript
document.addEventListener('load', loadGame());

function loadGame() {
    fetch('/game', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({}),
        })
        .then(response => response.json())
        .then(data => {
            state.values = data.values;
            state.startTime = data.startTime * 1000;
            state.value1 = data.values[0][0];
            state.value2 = data.values[0][1];
            document.getElementById('value1').textContent = state.value1;
            document.getElementById('value2').textContent = state.value2;
            updateCountdown();
        })
        .catch(error => {
            document.getElementById('dialog').textContent = '加载失败，请刷新页面重试';
        });
}
```

服务端返回的 `values` 中包含了开始时间和比大小的一对对数字。

设置好 `state` 后，调用 `updateCountdown` 函数，根据开始时间播放倒计时动画：

```javascript
function updateCountdown() {
    if (state.stopUpdate) {
        return;
    }
    const seconds = Math.ceil((state.startTime - Date.now()) / 1000);
    if (seconds >= 4) {
        requestAnimationFrame(updateCountdown);
    }
    if (seconds <= 3 && seconds >= 1) {
        document.getElementById('dialog').textContent = seconds;
        requestAnimationFrame(updateCountdown);
    } else if (seconds <= 0) {
        document.getElementById('dialog').style.display = 'none';
        state.allowInput = true;
        updateTimer();
    }
}
```

当游戏开始时，调用 `updateTimer` 函数更新时间和对手进度，对手会在 10 秒完成题目：

```javascript
function updateTimer() {
    if (state.stopUpdate) {
        return;
    }
    const time1 = Date.now() - state.startTime;
    const time2 = Math.min(10000, time1);
    state.score2 = Math.max(0, Math.floor(time2 / 100));
    document.getElementById('time1').textContent = `${String(Math.floor(time1 / 60000)).padStart(2, '0')}:${String(Math.floor(time1 / 1000) % 60).padStart(2, '0')}.${String(time1 % 1000).padStart(3, '0')}`;
    document.getElementById('time2').textContent = `${String(Math.floor(time2 / 60000)).padStart(2, '0')}:${String(Math.floor(time2 / 1000) % 60).padStart(2, '0')}.${String(time2 % 1000).padStart(3, '0')}`;
    document.getElementById('score2').textContent = state.score2;
    document.getElementById('progress2').style.width = `${state.score2}%`;
    if (state.score2 === 100) {
        state.allowInput = false;
        state.stopUpdate = true;
        document.getElementById('dialog').textContent = '对手已完成，挑战失败！';
        document.getElementById('dialog').style.display = 'flex';
        document.getElementById('time1').textContent = `00:10.000`;
    } else {
        requestAnimationFrame(updateTimer);
    }
}
```

玩家按键或点击选择答案，会调用 `chooseAnswer` 函数将答案放入 `state.inputs` 中。当玩家获得 100 分时调用 `submit` 函数提交答案，否则显示错误信息：

```javascript
document.addEventListener('keydown', e => {
    if (e.key === 'ArrowLeft' || e.key === 'a') {
        document.getElementById('less-than').classList.add('active');
        setTimeout(() => document.getElementById('less-than').classList.remove('active'), 200);
        chooseAnswer('<');
    } else if (e.key === 'ArrowRight' || e.key === 'd') {
        document.getElementById('greater-than').classList.add('active');
        setTimeout(() => document.getElementById('greater-than').classList.remove('active'), 200);
        chooseAnswer('>');
    }
});
document.getElementById('less-than').addEventListener('click', () => chooseAnswer('<'));
document.getElementById('greater-than').addEventListener('click', () => chooseAnswer('>'));

function chooseAnswer(choice) {
    if (!state.allowInput) {
        return;
    }
    state.inputs.push(choice);
    let correct;
    if (state.value1 < state.value2 && choice === '<' || state.value1 > state.value2 && choice === '>') {
        correct = true;
        state.score1++;
        document.getElementById('answer').style.backgroundColor = '#5e5';
    } else {
        correct = false;
        document.getElementById('answer').style.backgroundColor = '#e55';
    }
    document.getElementById('answer').textContent = choice;
    document.getElementById('score1').textContent = state.score1;
    document.getElementById('progress1').style.width = `${state.score1}%`;
    state.allowInput = false;
    setTimeout(() => {
        if (state.score1 === 100) {
            submit(state.inputs);
        } else if (correct) {
            state.value1 = state.values[state.score1][0];
            state.value2 = state.values[state.score1][1];
            state.allowInput = true;
            document.getElementById('value1').textContent = state.value1;
            document.getElementById('value2').textContent = state.value2;
            document.getElementById('answer').textContent = '?';
            document.getElementById('answer').style.backgroundColor = '#fff';
        } else {
            state.allowInput = false;
            state.stopUpdate = true;
            document.getElementById('dialog').textContent = '你选错了，挑战失败！';
            document.getElementById('dialog').style.display = 'flex';
        }
    }, 200);
}
```

`submit` 函数发送例如 `{ "input": ["<", ">", ...] }` 的 JSON 数据：

```javascript
function submit(inputs) {
    fetch('/submit', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({
                inputs
            }),
        })
        .then(response => response.json())
        .then(data => {
            state.stopUpdate = true;
            document.getElementById('dialog').textContent = data.message;
            document.getElementById('dialog').style.display = 'flex';
        })
        .catch(error => {
            state.stopUpdate = true;
            document.getElementById('dialog').textContent = '提交失败，请刷新页面重试';
            document.getElementById('dialog').style.display = 'flex';
        });
}
```

## 解法一：控制台调用函数

由于按钮点击有 200 毫秒的冷却，所以如果想要自动填写答案，需要先在开发者工具使用覆盖功能修改代码。

没这么麻烦的必要，可以直接在控制台调用 `submit` 函数，发送正确答案：

```javascript
submit(state.values.map(([v1, v2]) => v1 < v2 ? '<' : '>'))
```

需要注意得等倒计时结束，不然会「检测到时空穿越，挑战失败！」。

## 解法二：编写脚本发送请求

```python
import requests
import time


base_url = "http://202.38.93.141:12122/"
token = "<your_token>"

session = requests.session()
session.get(base_url, params={"token": token})

resp = session.post(f"{base_url}/game", json={})
game_data = resp.json()

answers = ["<" if value1 < value2 else ">" for value1, value2 in game_data["values"]]

start_time = game_data["startTime"]
current_time = time.time()
if current_time < start_time:
    wait_time = start_time - current_time
    time.sleep(wait_time)

resp = session.post(f"{base_url}/submit", json={"inputs": answers})
print(resp.json())
```
