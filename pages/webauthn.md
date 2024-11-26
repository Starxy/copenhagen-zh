---
title: "WebAuthn"
---
# WebAuthn

## 概述

[Web身份验证（WebAuthn）标准](https://www.w3.org/TR/webauthn-2/)允许用户使用设备进行身份验证，可以是PIN码或生物识别。私钥存储在用户设备中，公钥存储在你的应用程序中。应用程序可以通过验证签名来认证用户。由于凭证与用户设备绑定，并且不可能进行暴力破解，潜在攻击者需要物理访问设备。

WebAuthn通常有两种使用方式：通过密码钥匙或安全令牌。虽然没有严格定义，但密码钥匙通常指可以替代密码的凭证并存储在验证器中（驻留密钥）。另一方面，安全令牌则用作在密码验证后使用的第二因素。2FA的凭证通常加密并存储在依赖方的服务器中。这两种情况下，它们都是现有方法的更安全替代方案。

使用WebAuthn，应用程序还可以通过制造商验证设备。这需要声明，不在本页涵盖。

## 术语

- 依赖方：你的应用程序。
- 验证器：持有凭证的设备。
- 挑战：随机生成的单次使用[令牌](/server-side-tokens)，用于防止重放攻击。推荐的最小熵为16字节。
- 用户存在：用户可访问设备。
- 用户验证：用户通过PIN码或生物识别验证其身份。
- 驻留密钥，可发现凭证：存储在验证器（用户设备和安全令牌）中的凭证。非驻留密钥加密并存储在依赖方服务器中（你的数据库）。

## 注册

在注册步骤中，验证器创建一个新凭证并返回其公钥。

在客户端，从服务器获取新挑战，并使用[Web身份验证API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API)创建新凭证。这会提示用户使用设备进行身份验证。像Safari这样的浏览器只允许在用户交互（按钮点击）后调用此方法。

```ts
const credential = await navigator.credentials.create({
    publicKey: {
        attestation: "none",
        rp: { name: "My app" },
        user: {
            id: crypto.getRandomValues(new Uint8Array(32)),
            name: username,
            displayName: name,
        },
        pubKeyCredParams: [
            {
                type: "public-key",
                // ECDSA with SHA-256
                alg: -7,
            },
        ],
        challenge,
        authenticatorSelection: {
            userVerification: "required",
            residentKey: "required",
            requireResidentKey: true,
        },
        excludeCredentials: [
            {
                id: new Uint8Array(/*...*/),
                type: "public-key",
            },
        ],
    },
});
if (!(credential instanceof PublicKeyCredential)) {
    throw new Error("Failed to create credential");
}
const response = credential.response;
if (!(response instanceof AuthenticatorAttestationResponse)) {
    throw new Error("Unexpected");
}

const clientDataJSON: ArrayBuffer = response.clientDataJSON;
const attestationObject: ArrayBuffer = response.attestationObject;
```

- `rp.name`: 你的应用程序名称。
- `user.id`: 用于验证器的随机用户ID。这可以不同于应用程序使用的实际用户ID。
- `user.name`: 便于识别的用户标识符（用户名，电子邮件）。
- `user.displayName`: 便于识别的显示名称（无需唯一）。
- `excludeCredentials`: 用户凭证的列表，以避免重复凭证。

算法ID来自[IANA COSE算法注册表](https://www.iana.org/assignments/cose/cose.xhtml)。推荐使用ECDSA和SHA-256（ES256），因为它广泛支持。你也可以使用`-257`支持RSA（RS256），以兼容更多设备，但仅支持它的设备较少。

大多数情况下，`attestation`应设置为`"none"`。我们不需要验证验证器的真实性，并非所有验证器都支持这一操作。

对于密码钥匙，确保公钥是驻留密钥并要求用户验证。

```ts
const credential = await navigator.credentials.create({
    publicKey: {
        // ...
        authenticatorSelection: {
            userVerification: "required",
            residentKey: "required",
            requireResidentKey: true,
        },
    },
});
```

对于安全令牌，可以跳过用户验证，凭证不需要是驻留密钥。通过将`authenticatorAttachment`设置为`cross-platform`限制验证器为安全令牌。

```ts
const credential = await navigator.credentials.create({
    publicKey: {
        // ...
        authenticatorSelection: {
            userVerification: "discouraged",
            residentKey: "discouraged",
            requireResidentKey: false,
            authenticatorAttachment: "cross-platform",
        },
    },
});
```

客户端数据JSON和验证器数据会发送到服务器进行验证。发送二进制数据的简单方法是使用base64编码。另一个选择是使用CBOR等方案，将类似JSON的数据编码为二进制。

第一步是解析声称对象，该对象用CBOR编码。包括声明和验证器数据。你可以使用声明来验证用户的设备（如果需要）。如果在客户端将其设置为`"none"`，则验证声明格式为`none`。

```go
var attestationObject AttestationObject

// 解析声称对象

if attestationObject.Fmt != "none" {
    return errors.New("invalid attestation statement format")
}

type AttestationObject  struct {
    Fmt                  string // "fmt"
    AttestationStatement AttestationStatement // "attStmt"
    AuthenticatorData    []byte // "authData"
}

type AttestationStatement struct {
    // 见规范
}
```

接下来解析验证器数据。

- 字节0-31：依赖方ID哈希。
- 字节32：标志：
    - 位0（最低有效位）：用户存在。
    - 位2：用户验证。
    - 位6：包含凭证数据。
- 字节33-36：签名计数器。
- 可变字节：凭证数据（二进制）。

依赖方ID为域名，不包括协议或端口，验证器数据包括其SHA-256哈希。对于localhost，依赖方ID为`localhost`。检查用户存在标志和用户验证标志（如果需要）。每次使用凭证时签名计数器都会增加，可用于检测伪造设备。如果你的应用程序预期用于硬件安全令牌，其中凭证绑定到令牌上，你需要将计数器与凭证一起存储并确保其值大于之前的尝试。然而，由于密码钥匙可以在设备间共享，因此可以忽略。

然后，从凭证数据中提取凭证ID和公钥。

- 字节0-15：验证器ID。
- 字节16和17：凭证ID长度。
- 可变字节：凭证ID。
- 可变字节：COSE公钥。

公钥是CBOR编码的COSE密钥。

```go
import (
    "bytes"
    "crypto/sha256"
    "encoding/binary"
    "encoding/json"
    "errors"
)
if len(authenticatorData) < 37 {
    return errors.New("invalid authenticator data")
}
rpIdHash := authenticatorData[0:32]
expectedRpIdHash := sha256.Sum256([]byte("example.com"))
if bytes.Equal(rpIdHash, expectedRpIdHash[:]) {
    return errors.New("invalid relying party ID")
}

// 检查“用户存在”标志。
if (authenticatorData[32] & 1) != 1 {
    return errors.New("user not present")
}
// 如果需要用户验证，检查“用户验证”标志。
if ((authenticatorData[32] >> 2) & 1) != 1 {
    return errors.New("user not verified")
}
if ((authenticatorData[32] >> 6) & 1) != 1 {
    return errors.New("missing credentials")
}

if (len(authenticatorData) < 55) {
    return errors.New("invalid authenticator data")
}
credentialIdSize:= binary.BigEndian.Uint16(authenticatorData[53 : 55])
if (len(authenticatorData) < 55 + credentialIdSize) {
    return errors.New("invalid authenticator data")
}
credentialId := authenticatorData[55 : 55+credentialIdSize]
coseKey := authenticatorData[55+credentialIdSize:]

// 解析COSE公钥
```

公钥的结构取决于使用的算法。下面是使用(x, y)作为公钥的ECDSA公钥。验证算法和曲线。

```
{
    1: 2 // EC2密钥类型
    3: -7 // ECDSA P-256与SHA-256的算法ID
    -1: 1 // P-256的曲线ID
    -2: 0x00...00 // x坐标的位串
    -3: 0x00...00 // y坐标的位串
}
```

接下来，验证JSON编码的客户端数据。源是包含协议和端口的应用程序域名。客户端数据中的挑战是base64url编码，不带填充。

```go
import (
    "bytes"
    "crypto/sha256"
    "encoding/base64"
    "encoding/json"
    "errors"
)

var expectedChallenge []byte

// 验证挑战并从存储中删除。

var credentialId string

var clientData ClientData

// 解析JSON

if clientData.Type != "webauthn.create" {
    return errors.New("invalid type")
}
if !verifyChallenge(clientData.Challenge) {
    return errors.New("invalid challenge")
}
if clientData.Origin != "https://example.com" {
    return errors.New("invalid origin")
}

type ClientData struct {
    Type	  string // "type"
    Challenge string // "challenge"
    Origin	  string // "origin"
}
```

最后，用用户的公钥和凭证ID创建一个新用户。建议将COSE编码的公钥转换为更紧凑和标准的格式（[ECDSA](/cryptography/ecdsa#public-keys)）。

## 认证

在认证步骤中，验证器使用私钥创建一个新签名。

在服务器上生成一个挑战并认证用户。

```ts
const credential = await navigator.credentials.get({
    publicKey: {
        challenge,
        userVerification: "required",
    },
});

if (!(credential instanceof PublicKeyCredential)) {
    throw new Error("Failed to create credential");
}
const response = credential.response;
if (!(response instanceof AuthenticatorAssertionResponse)) {
    throw new Error("Unexpected");
}

const clientDataJSON: ArrayBuffer = response.clientDataJSON;
const authenticatorData: ArrayBuffer = response.authenticatorData;
const signature: ArrayBuffer = response.signature;
const credentialId: ArrayBuffer = publicKeyCredential.rawId;
```

要实现使用安全令牌的2FA，传递用户凭证列表到`allowCredentials`以支持非驻留密钥。

```ts
const credential = await navigator.credentials.get({
    publicKey: {
        challenge,
        userVerification: "required",
        allowCredentials: [
            {
                id: new Uint8Array(/*...*/),
                type: "public-key",
            },
        ],
    },
});
```

客户端数据、验证器数据、签名和凭证ID会发送到服务器。首先验证挑战、验证器和客户端数据。这部分几乎与验证声明的步骤相同，只是客户端数据类型应为`webauthn.get`。

```go
if clientData.Type != "webauthn.get" {
    return errors.New("invalid type")
}
```

另一个不同之处在于验证器不包含凭证部分。

使用凭证ID获取凭证的公钥。**对于2FA，确保凭证属于认证用户。**跳过此检查将允许恶意行为者完全跳过2FA。签名是验证器数据和客户端数据JSON的SHA-256哈希。对于ECDSA，签名是[ASN.1 DER编码的](/cryptography/ecdsa#pkix)。

```go
import (
    "crypto/ecdsa"
    "crypto/sha256"
)

clientDataJSONHash := sha256.Sum256(clientDataJSON)
// 将验证器数据与客户端数据JSON的哈希连接。
data := append(authenticatorData, clientDataJSONHash[:]...)
hash := sha256.Sum256(data)
validSignature := ecdsa.VerifyASN1(publicKey, hash[:], signature)
```