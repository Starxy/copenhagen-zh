---
title: "Multi-factor authentication (MFA)"
---

# 多因素认证 (MFA)

## 概述

MFA 是指用户需要输入不仅仅是密码的信息进行身份验证。主要有五种类型的因素：

- 你知道的东西：密码
- 你拥有的东西：设备、电子邮件地址、短信
- 你是什么：生物特征
- 你所在的位置
- 你的行为

## 基于时间的一次性密码 (TOTP)

TOTP 定义在 [RFC 6238](https://datatracker.ietf.org/doc/html/rfc6238) 中，基于 [RFC 4226](https://www.ietf.org/rfc/rfc4226.txt) 中定义的基于哈希的一次性密码 (HOTP)。

标准TOTP使用安装在用户移动设备上的身份验证器应用生成用户代码。

每个用户都有一个密钥，这个密钥通过二维码与用户的身份验证器应用共享。使用这个密钥和当前时间，身份验证器应用可以生成新的OTP。您的应用请求当前的OTP，并可以通过使用相同的参数生成一个来验证它。由于当前时间用于生成代码，每个代码仅在设定的时间段内有效（通常为30秒）。

### 生成二维码

HMAC SHA-1 用于生成TOTP。密钥正好为160位，必须使用密码学安全的随机生成器生成。每个用户必须有自己的密钥，并且应该存储在您的服务器中。如果担心数据库记录意外泄露，可以在存储前对密钥进行加密。然而，必须注意的是，加密数据无法保护对您服务器具有系统级访问权限的攻击者。

要共享密钥，生成一个 [key URI](https://github.com/google/google-authenticator/wiki/Key-Uri-Format) 并编码为二维码。`secret` 是 base32 编码的。

您应该通过请求生成的OTP来验证用户是否正确扫描了二维码。

```
otpauth://totp/example%20app:John%20Doe?secret=JBSWY3DPEHPK3PXP&issuer=Example%20App&digits=6&period=30
```

当用户请求新的二维码时，生成新的密钥并作废之前的密钥。

### 验证OTP

要验证TOTP，我们首先需要生成一个。

HOTP通过使用HMAC签署计数器值生成。在HOTP中，计数器是整数，每次生成新代码时递增。但在TOTP中，计数器是当前UNIX时间除以间隔（通常为30秒），截断小数部分。

计数器应为8个字节，用HMAC SHA-1哈希。用偏移量提取4个字节，然后提取最后31位并转换为整数。最后，使用后6位作为OTP。

```go
import (
	"crypto/hmac"
	"crypto/sha1"
	"encoding/binary"
	"fmt"
	"math"
)

func generateTOTP(secret []byte) string {
	digits := 6
	counter := time.Now().Unix() / 30

	// HOTP
	mac := hmac.New(sha1.New, secret)
	buf := make([]byte, 8)
	binary.BigEndian.PutUint64(buf, uint64(counter))
	mac.Write(buf)
	HS := mac.Sum(nil)
	offset := HS[19] & 0x0f
	Snum := binary.BigEndian.Uint32(HS[offset:offset+4]) & 0x7fffffff
	D := Snum % uint32(math.Pow(10, float64(digits)))
	// Pad "0" to make it 6 digits.
	return fmt.Sprintf("%06d", D)
}
```

要验证OTP，您可以在自己这边生成一个，并检查是否与用户提供的匹配。

必须实施节流策略。一个简单的例子是在连续5次失败尝试后，阻止尝试15到60分钟。用户还应该被通知更改密码。

## 短信

我们不建议使用基于短信的MFA，因为它可能被拦截且有时不可靠。然而，这比使用身份验证应用可能更容易访问。有关实现验证码的指南，请参见[电子邮件验证码](/email-verification#email-verification-codes)。代码应在5分钟内有效。

必须实施节流策略。一个简单的例子是在连续5次失败尝试后，阻止尝试15到60分钟。用户还应该被通知更改密码。

## WebAuthn

[Web身份验证 API (WebAuthn)](https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API) 允许应用程序使用用户设备，通过公钥密码学进行身份验证。您可以使用设备的PIN码或生物特征验证用户的身份，或者只验证设备。两者作为第二因素都有效，后者可能更用户友好，因为用户不需要输入密码或指纹。

有关实现的指南，请参见 [WebAuthn](/webauthn)。

## 恢复码

如果您的应用使用MFA，我们建议为用户提供一个或多个恢复码。这些是一次性密码，可以在用户失去访问设备时代替通行密钥/OTP用来登录和重置第二因素。必须使用密码学安全的随机生成器生成代码。假设实施适当的节流策略，它们仅需40位熵（使用十六进制编码时为10个字符）即可生成。

除非您能安全地存储这些代码，否则建议使用您首选的密码哈希算法（例如Argon2id）对其进行哈希处理。在这种情况下，代码第一次注册第二因素时才可见。如果他们可以访问他们的第二因素，用户还应该可以选择重新生成它们。
