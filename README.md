---
description: 无利不起早
---

# Java WebSocket学习笔记

WebSocket算是一个比较新的HTTP升级协议了。第一次见识这东西是在配置V2Ray的时候，了解到ws是一种全双工的长连接，那时候还是个小白，只是为了梯子自己跟着教程一步步走。WebSocket的诞生就是为了解决C/S之间的数据交换的效率问题，并且可以通过HTTP中介并不需要额外的监听需求，也实现了全双工，长连接让服务端可以反推送消息给客户端。

本次学习主要参考为Oracle早年由其架构师Danny Coward，也是Java EE中WebSocket API的规范领导者出品的《Java WebSocket Programming - Develop, Deploy, and Secure Dynamic Web Applications》一书，清华大学出版社出品其中文译本《Java WebSocket编程 - 开发、部署和保护动态Web应用》，这里可以为大家提供中英文书籍及其附赠源代码。

可以到这里下载书籍资料：[https://github.com/SkyJourney/wsAPIbook](https://github.com/SkyJourney/wsAPIbook)

