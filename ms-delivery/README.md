# 微服务持续交付

"持续集成"(Continuous integration)：频繁地将开发代码合并到主干，并保证可以编译通过，并通过基本额单元测试。
坚持持续集成的优点有：
* 快速失败(Fast Fail): 尽早发现错误，"早发现、早治疗"，将错误的成本降到最低。
* 减少代码冲突风险：使用频繁、多次、小的分支合并，来很久不合并代码导致的大量的代码冲突。
* 提升迭代速度和质量：小步快跑，让进度更可控，避免"Deadline前赶工期"的现象。

"持续交付"(Continuous delivery)：频繁地将代码的最新版本，交付给用户（或线上环境），它的优势不言而喻：
* 更快的交付速度：借助自动化的持续交付系统，可以做到1天上线多次
* 产品功能可并行上线：持续交付降低了上线的人力成本，多个功能可并行开发、上线

在互联网软件开发领域，持续集成和持续交付已经成为基本的共识，极大地提升了项目的迭代、交付速度。

与之形成鲜明对比的是，在传统软件开发领域，持续集成和持续交付的理念还没有得到贯彻，软件的开发、上线以周、月为单位，并且功能之间往往难以进行拆分。

本章将围绕上述两个问题，探讨探讨微服务架构下，借助Jenkins实现持续集成、持续部署。
