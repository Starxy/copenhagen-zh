---
title: "Elliptic curve digital signature algorithm (ECDSA)"
---

# 椭圆曲线数字签名算法 (ECDSA)

ECDSA 是一种使用椭圆曲线密码学的数字签名算法。使用私钥对消息进行签名，并使用公钥验证签名。

消息在签名前会使用诸如 SHA-256 这样的算法进行哈希。

```go
import (
	"crypto/ecdsa"
	"crypto/rand"
	"crypto/sha256"
)

msg := "Hello world!"
hash := sha256.Sum256([]byte(msg))
signature, err := ecdsa.SignASN1(rand.Reader, privateKey, hash[:])
```

## 签名

ECDSA 签名由一对正整数 (r, s) 表示。

### IEEE P1363

在 IEEE P1363 格式中，签名是 r 和 s 的连接。值作为大端字节进行编码，其大小等于曲线的大小。例如，P-256 是 256 位或 32 字节。

```ts
r || s;
```

### PKIX

在 PKIX 工作组的 [RFC 5480](https://datatracker.ietf.org/doc/html/rfc5480) 中，签名是 ASN.1 DER 编码的 r 和 s 序列。

```
SEQUENCE {
    r     INTEGER,
    s     INTEGER
}
```

## 公钥

ECDSA 公钥由一对正整数 (x, y) 表示。

### SEC1

在 [SEC 1](https://www.secg.org/sec1-v2.pdf) 中，公钥可以被编码为未压缩或压缩形式。未压缩的公钥是 x 和 y 的连接，并带有一个前导 `0x04` 字节。值作为大端字节进行编码，其大小等于曲线的大小。例如，P-256 是 256 位或 32 字节。

```
0x04 || x || y
```

压缩的公钥是 x 值，加上如果 x 为偶数则为前导 `0x02` 字节，如果 x 为奇数则为 `0x03` 字节。y 值可以从 x 和曲线导出。

```
0x02 || x
0x03 || x
```

### PKIX

在 PKIX 工作组的 [RFC 5480](https://datatracker.ietf.org/doc/html/rfc5480) 中，公钥表示为一个 `SubjectPublicKeyInfo` ASN.1 序列。`subjectPublicKey` 是压缩或未压缩的 SEC1 公钥。

```
SubjectPublicKeyInfo := SEQUENCE {
    algorithm           AlgorithmIdentifier,
    subjectPublicKey    BIT STRING
}
```

ECDSA 的 `AlgorithmIdentifier` 是包含 ECDSA 对象标识符（`1.2.840.10045.2.1`）和曲线（例如，P-256 曲线的 `1.2.840.10045.3.1.7`）的 ASN.1 序列。

```
AlgorithmIdentifier := SEQUENCE {
    algorithm   OBJECT IDENTIFIER
    namedCurve  OBJECT IDENTIFIER
}
```
