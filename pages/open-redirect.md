---
title: "Open redirect"
---

# 开放重定向

开放重定向是一种漏洞，您的应用程序允许用户控制重定向。

例如，您可能希望在用户登录后将其重定向回原始页面。为此，您在登录页面和表单中添加了一个 `redirect_to` URL 查询参数。当用户登录时，用户将被重定向到 `redirect_to` 中定义的位置。

```
https://example.com/login?redirect_to=%2Fhome
```

但如果您在没有验证的情况下接受任何重定向位置会怎样？

```
https://example.com/login?redirect_to=https%3A%2F%2Fscam.com
```

这乍看之下似乎无害，但它大大增加了诈骗用户的可能性。用户可能会被重定向到由攻击者制作的相同网站，并被要求再次输入密码。