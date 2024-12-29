---
title: 用 org mode 管理生活和工作
date: 2021-03-13 22:40:00
categories:
    - emacs
tags:
    - gdt
    - org-mode
---

# 起因

notion 是个方便高效的工具，用起来很顺手，但是最近频繁的 Notion Incident 邮件报警看得我着实心烦，很担心哪天要用的时候服务挂了干着急。所以决定寻找 notion 的替代。

![img](images/2021/03/Snipaste_2021-03-13_22-03-37.png)

# 目标

对我来说，一个高效的任务管理系统需要有一下功能：

1.  任务管理：管理 deadline、与他人共同工作的项目。
2.  材料记录和收集：记录想法、临时的一些思路、可以灵活的添加链接、图片等。
3.  保存和分享：持久化保存数据、不限于工具限制，便于导出分享。
4.  定期 review：达成一个长期的目标需要定期 review 短期的 task，保证方向合理和根据达成情况适当调整目标。
5.  易用：学习和维护成本低，用工具管理 task，而不是被工具牵制。

# 工具选择

基于前面描述的功能点，很多工具就被否定了，

1.  各种 todo 管理 app，以及附带番茄钟计时器的 app，例如：[Pomotodo](https://pomotodo.com/intl/en/)，[Wunderlist](https://www.wunderlist.com/)。 过于简单。
2.  专门的任务管理 app，例如：[omnifocus](https://www.omnigroup.com/omnifocus/)。 不便于添加各种资料。
3.  notion：不便于导出，服务频繁出问题。

最终我选择了 emacs 的 org mode。因为我有 emacs 的使用经历，上手 org-mode 较为容易；其次是会简单的写和改 elisp 代码，配置起来难度不大。

1.  简单但是比 markdown 强大的多的纯文本记录格式。
2.  task 支持状态、标签和时间，能方便的移动 task，便于调整 task 和 review。
3.  agenda view 过滤和查看 task。
4.  通过 pandoc 实现丰富的导出功能，就算哪天不用 org mode，也不影响查阅内容。
5.  添加图片确实不方便，但通过 org-download 也基本能实现拖拽添加。
6.  能**华丽呼哨**的定制，~感觉这个才是从 notion 换到 emacs 的动力~。

![img](images/2021/03/Snipaste_2021-03-13_21-36-25.png)

# 我的 org-mode workflow

## task 状态

参考别人的用法后，规划了如下几个 task 状态，含义如下：

-   TODO：收集到 inbox 中，但是还没有决定是否要做
-   NEXT：review TODO 的 task 以后，规划要做
-   PROG：正在进行
-   DONE：完成
-   HOLD：由于某些原因 task 暂停了
-   CANCELLED：取消不做了

状态的变更规则如下：

1.  TODO -> NEXT -> PROG -> DONE
2.  PROG -> HOLD
3.  PROG -> CANCELLED

除了上述状态，如果有的 task 有开始或截止时间，那么还可以加上 SCHEDULED 或者 DEADLINE。

## daily highlight

经过长久的实践，发现不管怎么规划 task，一旦事情堆的太多，看着整页的 NEXT 就会不想做。为此每天开始都先定一下当天**必须**要完成的事情，叫做 daily highlight，不限于工作的任务，休息日的时候也可以是看一本书之类的。daily highlight 不要过多，2～3 件即可。~上班的时候集中精力做完 daily highlight 以后就可以愉快的摸鱼了~。

目前使用 org mode 的时间还不长，上述的 workflow 可能还会随着时间不断更新。

# 导入 notion 文件

从 notion 导出的格式为 markdown，可以使用 pandoc 转为 org 格式。

    pandoc -f markdown -t org markdown_file.md | sed -E "/^[[:space:]]+:/ d"

# References

1.  [A Guide to My Organizational Workflow: How to Streamline Your Life](http://www.cachestocaches.com/2020/3/my-organized-life/)
2.  [An Org-Mode Workflow for Task Management](https://whhone.com/posts/org-mode-task-management/)
3.  [Org-mode Workflow: A Preview](https://blog.jethro.dev/posts/org_mode_workflow_preview/)
