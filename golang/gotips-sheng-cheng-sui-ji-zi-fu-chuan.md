# GoTips - 生成随机字符串

最近写一个小工具需要用到随机字符串，就到网上看了下大家的实现方案，很多简单粗暴的方式，本着 \(learn - understand - output\) 的套路，总结记录下相关思路和一些个人理解以备将来回顾查看

### Easy To Think

首先最容易想到的一个简单粗暴的实现就是通过 n 次 `#rand` 来生成 过程：

1. 列出所需字符串包含的内容（a-zA-Z）
2. 通过 `#rand` 方法得到一个随机数
3. 通过 mod 方法取到下标，拿到对应字符（or `#Intn()`）
4. 重复 2，3

```text
import "math/rand"

const letters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

func randString(n int) string {
    b := make([]byte, n)
    for i := range b {
        b[i] = letters[rand.Intn(len(letters))]
    }
    return string(b)
}
```

这种方式由于需要 n 次 `#rand` 所以效率不高

### Better

还看到另外一种更加巧妙的方式，是使用 `#Int63()` 方法，利用位运算来大幅减少 `#rand()` 的次数

其思路也非常清晰易懂：

假如目标字符库为：

```text
const letters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
```

其长度为 52 故使用二进制表示的话只需要 6 bits \(63\) 就可以表示

那么使用 `#Int63()` 来生成一个随机 int64 \(63-bit\) 的值，可以对这个 63-bit 的值加以充分利用

```text
// ex: 5215597282745260129
"100100001100001100001100001100001100001100001100001100001100001"

// 100_100001_100001_100001_100001_100001_100001_100001_100001_100001_100001
```

这样就可以一次得到 10 个最大值为 63 和 1 个最大值为 7 的随机数字

```text
const (
    letterBytes    = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    letterIdxBits  = 6                    // 6 bits to represent a letter index
    letterIdxMask  = 1<<letterIdxBits - 1 // All 1-bits, as many as letterIdxBits
    letterIdxMax   = 63 / letterIdxBits
)

func randString(n int) string {
    b := make([]byte, n)
    for i, r, remain := n-1, randsrc.Int63(), letterIdxMax; i >= 0; {
        // when n > 10
        if remain == 0 {
            r, remain = randsrc.Int63(), letterIdxMax
        }

        idx := int(r & letterIdxMask)
        // idx max value is 63
        if idx >= len(letterBytes) {
            idx >>= 1
        }

        b[i] = letterBytes[idx]
        i--

        r >>= letterIdxBits
        remain--
    }
    return string(b)
}
```

**1. 按位与**

```text
// r & letterIdxMask
100100001100001100001100001100001100001100001100001100001100001 & 111111
```

来拿到一个 index

**2. 右移**

```text
// r >>= 6
// r 右移 6 bits
// tips: 右移 n bits 相当于除以 2^n

// r.value
000000100100001100001100001100001100001100001100001100001100001
```

**3. 重复 1，2**

理解了思路，那么就可以做变通了

比如字符串内容扩展

`"1234567890!-_&.^abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"`

现在长度是 68 那么就可以调整通过 7 bits 来表示 这时一个 Int64 的数字将能够得到 `63/7` 个随机字符的 index

参考：

[https://stackoverflow.com/questions/22892120/how-to-generate-a-random-string-of-a-fixed-length-in-go](https://stackoverflow.com/questions/22892120/how-to-generate-a-random-string-of-a-fixed-length-in-go)

