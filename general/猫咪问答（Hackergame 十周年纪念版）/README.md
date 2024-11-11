# 猫咪问答（Hackergame 十周年纪念版）

## Q1

> 1\. 在 Hackergame 2015 比赛开始前一天晚上开展的赛前讲座是在哪个教室举行的？（30 分）
>
> 提示：填写教室编号，如 5207、3A101。

在中国科学技术大学 Linux 用户协会的[活动页面](https://lug.ustc.edu.cn/wiki/lug/events/hackergame/)找到关于 Hackergame 的活动记录。

第三届 2016 年的链接已失效，第二届 2015 年的链接存档在：[https://lug.ustc.edu.cn/wiki/sec/contest.html](https://lug.ustc.edu.cn/wiki/sec/contest.html)

比赛时间安排中找到「3A204」教室。

答案：`3A204`

## Q2

> 2\. 众所周知，Hackergame 共约 25 道题目。近五年（不含今年）举办的 Hackergame 中，题目数量最接近这个数字的那一届比赛里有多少人注册参加？（30 分）
>
> 提示：是一个非负整数。

我是挨个看了往届比赛的题解仓库，找到每一届的题目数量：

- 2023: 29
- 2022: 33
- 2021: 31
- 2020: 31
- 2019: 28

最靠近 25 的是 2019 年的比赛，找到赛事相关新闻页面：[https://lug.ustc.edu.cn/news/2019/12/hackergame-2019/](https://lug.ustc.edu.cn/news/2019/12/hackergame-2019/)

在新闻页面上发现有「2682」人注册。

答案：`2682`

## Q3

> 3\. Hackergame 2018 让哪个热门检索词成为了科大图书馆当月热搜第一？（20 分）
>
> 提示：仅由中文汉字构成。

又是检索又是图书馆的，那肯定是问答题带来的热门检索了。

于是翻看 [Hackergame 2018 猫咪问答题解](https://github.com/ustclug/hackergame2018-writeups/blob/master/official/ustcquiz/README.md)，有一题：

> 在中国科大图书馆中，有一本书叫做《程序员的自我修养:链接、装载与库》，请问它的索书号是？
>
> 打开中国科大图书馆主页，直接搜索“程序员的自我修养”即可。

那么答案就是「程序员的自我修养」。

但准确信息是在[花絮](https://github.com/ustclug/hackergame2018-writeups/blob/master/misc/others.md)里。

答案：`程序员的自我修养`

## Q4

> 4\. 在今年的 USENIX Security 学术会议上中国科学技术大学发表了一篇关于电子邮件伪造攻击的论文，在论文中作者提出了 6 种攻击方法，并在多少个电子邮件服务提供商及客户端的组合上进行了实验？（10 分）
>
> 提示：是一个非负整数。

在 [USENIX Security](https://www.usenix.org/) 官网搜索「email」，得到论文名称：

「FakeBehalf: Imperceptible Email Spoofing Attacks against the Delegation Mechanism in Email Systems」

在 [论文 PDF](https://www.usenix.org/system/files/usenixsecurity24-ma-jinrui.pdf) 第 6 节 *Imperceptible Email Spoofing Attack*：

> Con-sequently, we propose six types of email spoofing attacks and measure their impact across 16 email services and 20 clients. All 20 clients are configured as MUAs for all 16 providers via IMAP, resulting in **336** combinations (including 16 web interfaces of target providers)

答案：`336`

## Q5

> 5\. 10 月 18 日 Greg Kroah-Hartman 向 Linux 邮件列表提交的一个 patch 把大量开发者从 MAINTAINERS 文件中移除。这个 patch 被合并进 Linux mainline 的 commit id 是多少？（5 分）
>
> 提示：id 前 6 位，字母小写，如 c1e939。

足够争议性的话题，国内外媒体新闻漫天飞，相关信息挺多，但大多是指向 patch 的链接，不是合并。

在 Linux 内核的[官方 Git 仓库](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git)上搜索作者「Greg Kroah-Hartman」，可以得到删除 MAINTAINERS 的 commit：

[https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6e90b675cf942e50c70e8394dfb5862975c3b3b2](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6e90b675cf942e50c70e8394dfb5862975c3b3b2)

答案：`6e90b6`

## Q6

> 6\. 大语言模型会把输入分解为一个一个的 token 后继续计算，请问这个网页的 HTML 源代码会被 Meta 的 Llama 3 70B 模型的 tokenizer 分解为多少个 token？（5 分）
>
> 提示：首次打开本页时的 HTML 源代码，答案是一个非负整数

要用 Llama 3 70B 的 Tokenizer，需要在 [HuggingFace 仓库](https://huggingface.co/meta-llama/Meta-Llama-3-70B) 向 Meta 申请访问权限，比赛时没想着等，找了在线的 Tokenizer：

- [https://tiktokenizer.vercel.app/?model=meta-llama%2FMeta-Llama-3-70B](https://tiktokenizer.vercel.app/?model=meta-llama%2FMeta-Llama-3-70B)
- [https://lunary.ai/llama3-tokenizer](https://lunary.ai/llama3-tokenizer)

得到的结果会上下浮动，有条件还是获得个访问权限，然后只使用 Tokenizer：

```python
import transformers
import requests


session = requests.session()
token = "<your_token>"
resp = session.get(
    "http://202.38.93.141:13030/",
    params={"token": token},
)

tokenizer = transformers.AutoTokenizer.from_pretrained("meta-llama/Meta-Llama-3-70B")
tokens = tokenizer.encode(resp.text)
print(len(tokens))
```

可能是带有 BOS Token 或者 EOS Token，我的代码得到结果是「1835」，正确答案为「1833」。

答案：`1833`

~~看了一些别人的题解，都是哪来的钱哪来的机器，70B 模型说跑就跑啊~~
