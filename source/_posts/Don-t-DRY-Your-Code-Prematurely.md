---
title: 【译】不要过早 DRY 你的代码
date: 2024-06-07 10:52:06
tags:
- Translation
---

> 原文链接：[Don't DRY Your Code Prematurely](https://testing.googleblog.com/2024/05/dont-dry-your-code-prematurely.html)

我们大概已经了解过[“Don't Repeat Yourself”](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)或者说 DRY 这一编程美德了。不过先让我们停下来想一想：重复的代码真的是冗余的吗？还是说这些功能会随着时间独自进化成不同的样子呢？**严格遵循 DRY 原则相当于过早引入了抽象，这会让未来的改动变得比实际更复杂。**

<!--more-->

**仔细考虑一下代码是真的冗余了还是只是表面上很相似。**函数和类可能看起来很像，但它们也许是服务于不同上下文或业务逻辑，这些东西可能随时间变化成完全不同的样子。多想想这些函数的作用是否能在一定时间内保持一致，而不仅仅是去在意代码的长短。**在设计抽象层的时候，千万不要过早把不同行为合并到一起，因为这些行为在长期来看很可能是完全相左的。**

什么时候抽象会导致我们的代码更复杂呢？来看两段不同的代码实现：

遵循 DRY 的**实现1**
```python
# Premature DRY abstraction assuming
# uniform rules, limiting entity-
# specific changes.
class DeadlineSetter:
 def __init__(self, entity_type):
   self.entity_type = entity_type

 def set_deadline(self, deadline):
   if deadline <= datetime.now():
     raise ValueError(
         “Date must be in the future”)
task = DeadlineSetter(“task”)
task.set_deadline(
    datetime(2024, 3, 12))
payment = DeadlineSetter(“payment”)
payment.set_deadline(
    datetime(2024, 3, 18))
```

看起来有些重复的**实现2**
```python
# Repetitive but allows for clear,
# entity-specific logic and future
# changes.
def set_task_deadline(task_deadline):
  if task_deadline <= datetime.now():
    raise ValueError(
        “Date must be in the future”)

def set_payment_deadline(
    payment_deadline):
  if payment_deadline <= datetime.now():
    raise ValueError(
        “Date must be in the future”)

set_task_deadline(
    datetime(2024, 3, 12))
set_payment_deadline(
    datetime(2024, 3, 18))
```

实现2似乎是违背了 DRY 原则，因为 `ValueError` 非常巧合地长成一模一样了。然而，task 和 payment 表达了两种不同的概念，所以它们也很可能需要两种不同的处理方式。如果 payment 在未来的某一天要添加一种新的鉴定逻辑，那么在实现2中可以轻松实现，但对实现1来说就有很强的侵入性了。

**当你不是很确定的时候，尽量让不同行为保持独立，直到出现了足够多的、相似的模式来支撑行为的合并。**往小了说，管理重复的代码也要比理解过早提出来的抽象层要简单得多。**在开发的早期阶段，先容忍那些为数不多的重复，然后等着时间来告诉你何时需要抽象。**

未来的需求常常是难以预测的。想想[“You Aren’t Gonna Need It”](https://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it) 或者说 YAGNI 原则。随着时间的推移，你要么会发现那些重复代码其实并不是问题，要么会收到一个明确的信号提醒你需要仔细设计一个抽象层了。
