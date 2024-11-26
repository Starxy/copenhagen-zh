---
title: "Server-side tokens"
---

# 服务器端令牌

## 目录

- [服务器端令牌](#服务器端令牌)
	- [目录](#目录)
	- [概述](#概述)
	- [生成令牌](#生成令牌)
	- [存储令牌](#存储令牌)

## 概述

“服务器端令牌”是存储在服务器上的任意长、随机字符串。它可以保存在数据库或内存存储中（如 Redis），用于身份验证和验证。通过检查令牌是否存在于存储中可以进行验证。示例包括会话 ID、电子邮件验证令牌和访问令牌。

```sql
CREATE TABLE token (
    token STRING NOT NULL UNIQUE,
    expires_at INTEGER NOT NULL,
    user_id INTEGER NOT NULL,

    FOREIGN KEY (user_id) REFERENCES user(id)
)
```

对于一次性使用的令牌，任何检索操作也应保证删除。在 SQL 中，例如，在获取令牌时应使用事务等原子操作。

## 生成令牌

令牌应至少具有 112 位的熵（120-256 位是个不错的范围）。例如，可以生成 15 个随机字节并用 base32 编码得到一个 24 个字符的令牌。如果通过逐字符随机选择来生成令牌，应确保具有类似的熵水平。有关更多信息，请参考[生成随机值](/random-values)页面。

必须使用加密安全的随机生成器来生成令牌。应避免使用标准数学包中提供的快速伪随机生成器。

令牌应区分大小写，但如果存储不区分大小写（例如 MySQL），则可能需要将令牌生成限制为小写字母。

> 对于 120 位令牌，即使每秒生成 10,000 个令牌，系统中有 1,000,000 个有效令牌，猜中一个有效令牌也需要 20 亿年。

```go
import (
    "crypto/rand"
    "encoding/base32"
)

bytes := make([]byte, 15)
rand.Read(bytes)
sessionId := base32.StdEncoding.EncodeToString(bytes)
```

UUID v4 可能符合这些要求（122 位的熵），但请记住 UUID v4 的空间效率低下，规范也不保证使用加密安全的随机生成器。

## 存储令牌

如密码重置令牌等需要额外安全级别的令牌，应使用 SHA-256 进行哈希处理。由于令牌足够长且随机，可使用 SHA-256 而非较慢的算法。在查询之前，通过将传入的令牌进行哈希来验证。

现实生活中的泄漏示例包括 [Paleohacks](https://www.vpnmentor.com/blog/report-paleohacks-breach/) 和 [Spoutible](https://www.troyhunt.com/how-spoutibles-leaky-api-spurted-out-a-deluge-of-personal-data/)。
