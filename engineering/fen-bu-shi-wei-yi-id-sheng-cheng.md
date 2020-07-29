# 分布式唯一 ID 生成

在各种分布式系统中，有很多场景需要唯一性标识：

* 日常业务中的各种用户数据
* 分布式事务 ID
* MQ 消费幂等校验
* ...

### UUID

> A Universally Unique IDentifier \(UUID\) URN Namespace 
>
> RFC: [https://www.ietf.org/rfc/rfc4122.txt](https://www.ietf.org/rfc/rfc4122.txt)

UUID 的标准形式：由 32 个 16 进制数及 '-' 符组成，形式为 8-4-4-4-12 的 36 位字符串

优点： 

1. 性能高，纯本地生成

缺点： 

1. 生成无顺序性，不适宜用做索引 
2. 太长，很多场景不适宜

虽然有一些缺点，但 UUID 仍是一种非常常用的唯一标识方案

### Snowflake

Snowflake 是 Twitter 开源的一个分布式 ID 生成算法

> [https://developer.twitter.com/en/docs/basics/twitter-ids](https://developer.twitter.com/en/docs/basics/twitter-ids) 
>
> [https://github.com/twitter-archive/snowflake/tree/snowflake-2010](https://github.com/twitter-archive/snowflake/tree/snowflake-2010)

```text
id is composed of:
    1. time - 41 bits (millisecond precision w/ a custom epoch gives us 69 years)
    2. configured machine id - 10 bits - gives us up to 1024 machines
    3. sequence number - 12 bits - rolls over every 4096 per machine (with protection to avoid rollover in the same ms)
```

#### Generate

snowflake 最终生成 64 位的数字：

1. 最高位作为符号位置 0 
2. 下面 `41 bits` 存储毫秒精度时间戳 
3. 接着 `10 bits` 存储分布式节点\|机器ID 
4. 最低 `12 bits` 存储顺序序列号（递增）

从生成规则可以看出其几个特点： 

1. `41 bits` 毫秒级时间戳，以上线时间点为零点，可以表示 69 年 
2. `10 bits` 机器码位，可以表示 1024 台机器 
3. `12 bits` 的序号位，可以表示 2^12 = 4096 个 ID（每毫秒）

优点： 

1. ID 整体趋势递增 
2. 性能高 
3. 弱依赖第三方系统 
4. 生成规则可灵活调整

缺点： 1. 强依赖机器时钟

#### Complete

整体实现难度还是比较简单的，不过还是需要注意几个重点：

* 生成方法需要是线程安全的
* 需要处理机器时钟回退的问题 \(time.now &lt; last\_time\)
  * 一种方式是直接抛出异常，由外部或人为干预处理
  * 也可以阻塞到时钟追齐
* 处理同一毫秒内，序号位溢出（生成数超过最大上限）问题
  * 由于单毫秒单机器可生成的上限是 4096，当超过时可以阻塞到下一毫秒再生成

```go
package snowflake

import (
    "fmt"
    "sync"
    "time"
)

const (
    BitsTimeLen     = 41
    BitsMachineLen  = 10
    BitsSequenceLen = 12

    // 4095, all low-12 bits value is 1
    MaskSequence = int((1 << BitsSequenceLen) - 1)
)

type SnowConfig struct {
    StartTimestamp time.Time
    MachineID      int
}

type Snowflake struct {
    mu             *sync.Mutex
    startTimestamp int64
    lastTimestamp  int64
    machineID      int
    sequence       int
}

func NewSnowflake(cfg *SnowConfig) (*Snowflake, error) {
    flake := &Snowflake{
        mu:             new(sync.Mutex),
        startTimestamp: cfg.StartTimestamp.UnixNano() / 1000000,
        machineID:      cfg.MachineID,
        sequence:       MaskSequence,
    }

    return flake, nil
}

func (flake *Snowflake) NextID() (uint64, error) {
    flake.mu.Lock()
    defer flake.mu.Unlock()

    curTimestamp := flake.CurSnowTimeStamp()
    // check machine clock
    if flake.lastTimestamp > curTimestamp {
        return 0, fmt.Errorf("abnormal machine clock, diff: %d", flake.lastTimestamp-curTimestamp)
    }

    if flake.lastTimestamp < curTimestamp {
        flake.lastTimestamp = curTimestamp
        flake.sequence = MaskSequence
    }

    flake.sequence = (flake.sequence + 1) & MaskSequence
    // out of limit
    if flake.sequence == 0 {
        flake.lastTimestamp++
        // time.Sleep((flake.lastTimestamp - curTimestamp) * time.Millisecond)
        for flake.lastTimestamp > flake.CurSnowTimeStamp() {
            // block
        }
    }

    if flake.lastTimestamp > flake.MaxTimestamp() {
        return 0, fmt.Errorf("out of time limit, MaxTimestamp: %d", flake.MaxTimestamp())
    }

    return uint64(flake.lastTimestamp)<<(BitsMachineLen+BitsSequenceLen) |
        uint64(flake.machineID)<<BitsSequenceLen |
        uint64(flake.sequence), nil
}

func (flake *Snowflake) MaxTimestamp() int64 {
    return int64((1 << BitsTimeLen) - 1)
}

func (flake *Snowflake) CurSnowTimeStamp() int64 {
    return time.Now().UnixNano()/1000000 - flake.startTimestamp
}
```

```text
goos: darwin
goarch: amd64
BenchmarkNextID-4        4904384           246 ns/op
PASS
ok      command-line-arguments    1.481s
```

### 其他

其他的还有美团的 Leaf，主要包括两种方式：

* Leaf-segment
* Leaf-snowflake

深入可参考：

* [https://tech.meituan.com/2017/04/21/mt-leaf.html](https://tech.meituan.com/2017/04/21/mt-leaf.html)
* [https://tech.meituan.com/2019/03/07/open-source-project-leaf.html](https://tech.meituan.com/2019/03/07/open-source-project-leaf.html)
* [https://github.com/Meituan-Dianping/Leaf](https://github.com/Meituan-Dianping/Leaf)

以及 sony 实现：

* [https://github.com/sony/sonyflake](https://github.com/sony/sonyflake)

