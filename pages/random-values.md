---
title: "Generating random values"
---

# 生成随机值

## 概述

标准数学包提供的伪随机生成器通常速度快但可预测。在处理加密时，获得强随机生成器是至关重要的。

本页面描述如何从随机生成的位生成随机字符串和数字。

## 随机字符串

生成随机字符串最简单且安全的方法是生成随机字节并使用base16（十六进制）、base32或base64编码方案对其进行编码。

```go
import (
	"crypto/rand"
	"encoding/base32"
)

func generateRandomString() string {
	bytes := make([]byte, 12)
	rand.Read(bytes)
	return base32.StdEncoding.EncodeToString(bytes)
}
```

### 自定义字符集

如果字符集长度适合现有的编码方案（例如base64），可以自定义使用的字符。

```go
import (
	"crypto/rand"
	"encoding/base32"
)

var customEncoding = base32.NewEncoding("0123456789ABCDEFGHJKMNPQRSTVWXYZ")

func generateRandomString() string {
	bytes := make([]byte, 12)
	rand.Read(bytes)
	return customEncoding.EncodeToString(bytes)
}
```

如果不适合，则需要一个高质量的[随机数生成器](#随机整数)来生成自定义范围内的整数。

```go
const alphabet = "abcdefg"

func generateRandomString() string {
	var result string
	for i := 0; i < 12; i++ {
		result += string(alphabet[generateRandomInt(0, len(alphabet))])
	}
	return result
}
```

## 随机整数

如果范围是2的幂（2, 4, 8, 16等），可以使用简单的位掩码来实现。

```go
bytes := make([]byte, 1)
rand.Read(bytes)
value := bytes[0] & 0x03 // 生成[0, 3]之间的随机值
```

对于自定义范围，一个简单的方法是生成一个非常大的随机数并使用取模操作。由于这会引入[模偏差](#偏差)，随机整数必须足够大。例如，如果最大值是10并且我们生成32位随机数，偏差约为1/250,000,000，这对于大多数用例来说足够了。

```go
import (
	"crypto/rand"
	"encoding/binary"
)

// 生成一个[0, max)之间的随机整数。
// `max`不应是非常大的数字。
func generateRandomUint32(max uint32) uint32 {
	bytes := make([]byte, 4)
	rand.Read(bytes)
	randUint32 := binary.BigEndian.Uint32(bytes) // 将字节转换为uint32
	return randUint32 % max
}
```

另一种常见的方法是将最大值乘以一个随机浮点数。这也可能[引入偏差](#偏差)，但如果最大值足够小，这种偏差是分散的。

```go
func generateRandomUint32(max uint32) uint32 {
	return uint32(max * generateRandomFloat32())
}
```

最安全的方法是使用拒绝采样，不断生成随机值直到其小于最大值。为了增加随机值小于最大值的可能性，我们只需生成表示最大值所需的位数。例如，如果最大值是10，我们只需生成4位。在下面的代码中，我们生成一个随机字节，然后屏蔽前4位以获得4个随机位（8-4=4）。

```go
import (
	"crypto/rand"
	"math/big"
)

func generateRandomUint64(max *big.Int) uint64 {
	randVal := new(big.Int)
	shift := max.BitLen() % 8
	bytes := make([]byte, (max.BitLen() / 8) + 1)
	rand.Read(bytes)
	if shift != 0 {
		bytes[0] &= (1 << shift) - 1
	}
	randVal.SetBytes(bytes)
	for randVal.Cmp(max) >= 0 {
		rand.Read(bytes)
		if shift != 0 {
			bytes[0] &= (1 << shift) - 1
		}
		randVal.SetBytes(bytes)
	}
	return randVal.Uint64()
}
```

### 生成0到1之间的随机浮点数

一种常见的方法是生成一个随机整数并将其除以一个非常大的数。在这样做时，确保分母足够大，并且分母是2的幂，以便能被float64准确表示。

```go
func generateRandomFloat64() float64 {
	return float64(generateRandomInteger(1<<53)) / (1 << 53)
}
```

另一种方法是为尾数（float64）生成52个随机位，并将其转换为[0, 1)之间的浮点数。这通常更快，因为它避免了除法。

```go
import (
	"crypto/rand"
	"math"
)

func generateRandomFloat64() float64 {
	bytes := make([]byte, 7)
	rand.Read(bytes)
	bytes = append(make([]byte, 1), bytes...)
	// 设置指数部分为0b01111111111
	bytes[0] = 0x3f
	bytes[1] |= 0xf0
	return math.Float64frombits(binary.BigEndian.Uint64(bytes)) - 1
}
```

## 偏差

一个非常常见的偏差是模偏差。例如，如果`RANDOM_INT`是[0, 10)之间的整数，某些数字会出现3次（0, 1），而其他数字会出现2次（2, 3）。

```
RANDOM_INT % 4
```

要计算近似偏差，可以使用下面的公式。

```
1 / ( RANDOM_BITS - LOG2(MAX) )
```

例如，如果我们使用8个随机位且最大值为100，近似偏差为0.6：

```
1 / ( 8 - LOG2(100) ) ≈ 1 / (8-6.4) ≈ 0.6
```

将最大值乘以随机浮点数也可能引入偏差。在此示例中，`RANDOM_FLOAT`在[0, 1)范围内，因此输出将在[0, 5)内。

```
FLOOR( RANDOM_FLOAT * 5 )
```

假设`RANDOM_FLOAT`可以是8个数之一：0, 0.125, 0.25, ..., 0.875。在这种情况下，（0, 1, 3）会出现1/4次，而（2, 4）只出现1/8次。虽然这是一个极端的例子，但确保随机浮点数相对于最大值提供足够的“随机性”是很重要的。
