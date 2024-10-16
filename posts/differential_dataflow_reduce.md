# Differential Dataflow: Reduce
今天聊一下 [Differential Dataflow] (后面简称 DD) 的 `reduce` 算子, `reduce` 类似 SQL 中的 group by 语句, 将数据按 key 分组, 然后将
每个分组的数据按一定的逻辑输出一个或几个值, 这个逻辑可以是 sum, count, min, max 等. 在 DD 中, 这个逻辑是
```rust
/// Input data must be structured as `(key, val)` pairs.
/// The user-supplied reduction function takes as arguments
///
/// 1. a reference to the key,
/// 2. a reference to the slice of values and their accumulated updates,
/// 3. a mutuable reference to a vector to populate with output values and accumulated updates.
///
/// The user logic is only invoked for non-empty input collections, and it is safe to assume that the
/// slice of input values is non-empty. The values are presented in sorted order, as defined by their
/// `Ord` implementations.

L: FnMut(&K, &[(&V, R)], &mut Vec<(V2, R2)>)+'static;
```

这篇文章主要讲一下 `reduce` 在流式系统中输出形状以及一些常见的优化方法.

## DD 中的 reduce 输出
我们以 `count` (`count` 基于 `reduce`) 为例
```rust
    timely::execute_directly(move |worker| {
        let mut probe = ProbeHandle::new();
        let mut input = worker.dataflow::<u32, _, _>(|scope| {
            let (handle, input) = scope.new_collection();
            input
                .count()
                .inspect(|((k, v), t, r)| println!("key: {k}, count: {v}, time: {t}, diff: {r}"))
                .probe_with(&mut probe);
            handle
        });

        for i in 1..5 {
            for _ in 0..i {
                input.insert("a".to_string());
            }
            sync!(input, probe, worker);
        }
    })
```
这里我们一共进行了 4 轮输入, 第 i 轮输入 i 条数据, 最终的输出:
```
key: a, count: 1, time: 0, diff: 1
key: a, count: 1, time: 1, diff: -1
key: a, count: 3, time: 1, diff: 1
key: a, count: 3, time: 2, diff: -1
key: a, count: 6, time: 2, diff: 1
key: a, count: 6, time: 3, diff: -1
key: a, count: 10, time: 3, diff: 1
```
这里我们发现一个很有意思的现象,除了最后一条数据外,其他的数据都是一对一对的形式出现,他们的值相等, 时间正好差 1, diff 刚好相反.
如果我们在 time = 3 这个时间点去看数据的话, 那些一对一对的数据都会互相抵消, 最终只剩下 count = 10, time =3 这条数据. 这
和最终的 count 能对上!.

**那些一对一对的数据在某个时间点互相抵消** 是一个非常重要的性质. 曾经我有一个疑问, 为什么每个时刻都要产生两条数据, 一条是前面一个状态的'负值', 一条是一个完整的新值, 比如说我们能不能产生这样的数据流:
```
key: a, count: 1, time: 0, diff: 1
key: a, count: 2, time: 1, diff: 1
key: a, count: 3, time: 2, diff: 1
key: a, count: 4, time: 3, diff: 1
```
每条数据相当与是之前状态的一个增量值, 这样不仅减少了数据量, 而且还体现了 **differential**, DD 不就是干增量计算的吗?, 后来想了想, 感觉自己太 naive 了, 我这里举两个例子, 一个是 `arrangement`, 我们分别对上面两条 stream 进行 arrange, 上面
那条数据流对应的 `arrangment` 在 `compaction frontier` 提升到 3 的时候, 历史的数据都会互相抵消, 最后这个 `arrangement` 中只会剩下一条数据. 下面那条数据流由于有很多不同的增量值, 所以无论 `compaction frontier` 如何提升,
`arrangement` 始终要维护大量的数据. 另外一个例子计算的正确性问题, 假设我们在上面两条 stream 后面在添加一个 filter 算子:
`|count| count > 7`, 那么上面的 流将输出最后一条数据, 符合要求, 但是下面那条流找不到满足要求的数据,所以什么也不会输出.


## 数据倾斜
这是一个老生常谈的问题,原始的 `reduce` 在 input 规模逐渐变大的时候计算时间势必也会随着增加, 同时 `reduce` 依赖 input, 
output arrangement, 占用的内存也会不断增加. 这里举一个求和的例子:
```rust
    execute_directly(move |worker| {
        let mut probe = ProbeHandle::new();
        let mut input = worker.dataflow::<u32, _, _>(|scope| {
            let (handle, input) = scope.new_collection::<(String, i64), _>();
            input
                .reduce(|_key, input, output| {
                    let sum: i64 = input.iter().map(|(v, diff)| (**v) * (*diff as i64)).sum();
                    output.push((sum, 1));
                })
                .probe_with(&mut probe);
            handle
        });

        let round = 10000;
        let batch = 10000;

        let mut rng = FastRng::new();
        for i in 0..round {
            let timer = Instant::now();
            for _ in 0..batch {
                let d: i32 = rng.gen();
                input.update(("a".to_string(), d as i64), 1);
            }
            sync!(input, probe, worker);
            let elapsed = timer.elapsed();
            println!("round[{i}]: time: {:?}", elapsed);
        }
    })
```
这个例子会执行 10000 次循环, 每次循环输入 10000 条数据, 然后等待计算完成. 执行时间如下:
```
round[0]: time: 2.934484ms
round[1]: time: 3.545811ms
round[2]: time: 4.654763ms
...
round[76]: time: 55.205525ms
round[77]: time: 53.153426ms
round[78]: time: 56.380268ms
...
round[399]: time: 301.233007ms
round[400]: time: 327.155308ms
round[401]: time: 273.926508ms

```
可以看到随着循环次数的时间, 执行所需要的时间越来越久. 

下面我们分情况进行不同的优化来解决这个问题.

### Sum
这里我们讨论 `sum` 场景下的优化方案. 这种情况下我们的 arrangement 其实不需要维护不同的 value, DD 中提供了 `explode` 方
法, 可以将类似下面的数据流
```
(K, V, T, Diff)
(a, 1, 1, 1)
(a, 2, 1, 2)
(a, 3, 1, 4)
```
转化成
```
(K, V, T, Diff)
(a, (), 1, 1)
(a, (), 1, 4)  diff = 2 * 2
(a, (), 1, 12) diff = 3 * 4
```
主要原理是通过把 value 转移到 Update 中的 Diff 部分, 然后依赖 arrangement 自动合并 diff 来降低需要维护的数据数量, 这样
较少了内存的占用, 同时直接计算出了 sum, 这个 sum 是增量式的计算, 但是 reduce 每次都会重复计算. 效率会有很大的差别, 我们
看下使用 `explode` 之后的例子:
```rust
    execute_directly(move |worker| {
        let mut probe = ProbeHandle::new();
        let mut input = worker.dataflow::<u32, _, _>(|scope| {
            let (handle, input) = scope.new_collection::<(String, i64), _>();
            input
                .explode(|(k, v)| Some((k, v)))
                .count() // <--- 把 diff 部分移动到 value
                .probe_with(&mut probe);
            handle
        });

        let round = 10000;
        let batch = 10000;

        let mut rng = FastRng::new();
        for i in 0..round {
            let timer = Instant::now();
            for _ in 0..batch {
                let d: i32 = rng.gen();
                input.update(("a".to_string(), d as i64), 1);
            }
            sync!(input, probe, worker);
            let elapsed = timer.elapsed();
            println!("round[{i}]: time: {:?}", elapsed);
        }
    })
```
我们直接看执行时间:
```
round[1]: time: 413.117µs
round[2]: time: 412.139µs
round[3]: time: 414.514µs
...
round[9997]: time: 413.117µs
round[9998]: time: 412.139µs
round[9999]: time: 414.514µs

```
我们可以看到每次循环的执行时间都在 410µs 左右, 没有随着数据量的增加而降低执行速度, 而且整个执行速度比 naive 的方式快了几个
数量级.

### 区分 <值为0 | 没有数据>
上面的例子中其实存在一个 bug, 我们看下面这个例子:
```rust
    execute_directly(move |worker| {
        let mut probe = ProbeHandle::new();
        let mut input = worker.dataflow::<u32, _, _>(|scope| {
            let (handle, input) = scope.new_collection::<(String, i64), _>();
            input
                .explode(|(k, v)| Some((k, v)))
                .count()
                .inspect(|s| println!("{:?}", s))
                .probe_with(&mut probe);
            handle
        });

        input.update(("a".to_string(), 1 as i64), 1);
        input.update(("a".to_string(), -1), 1);

        sync!(input, probe, worker);
    })
```
理论上这个例子应该输出 `(("a", 0), 0, 1)`, 但是这个例子没有输出任何数据. 原因是这里的 value=0, 这意味着在 arrangement 中对应的 Diff=0, 这个时候 DD 直接把这条数据干掉了, 所以我们需要用一种方法让 DD 区分这两种情况.

DD 中的 Diff 其实可以是除了整数之外的类型, 只要满足 `Semigroup` 即可, `Semigroup` 中的 `is_zero` 方法用来判断是否要
干掉数据, 所以我们我们需要在这个方法上做文章, 我们可以定义 `Sum` 类型, 同时包含 sum 值和 数据量 count, 只要当这两个值
都为 0 的时候才可以把数据干掉, 具体的证明过程就留给读者了.

```rust
#[derive(Clone, Debug, Serialize, Deserialize, Ord, PartialOrd, Eq, PartialEq)]
struct Sum {
    value: i64,
    count: i64,
}

impl IsZero for Sum {
    fn is_zero(&self) -> bool {
        self.count == 0 && self.value == 0
    }
}

impl Semigroup for Sum {
    fn plus_equals(&mut self, rhs: &Self) {
        self.value += rhs.value;
        self.count += rhs.count;
    }
}
```
修改后的程序:
```rust
    execute_directly(move |worker| {
        let mut probe = ProbeHandle::new();
        let mut input = worker.dataflow::<u32, _, _>(|scope| {
            let (handle, input) = scope.new_collection::<(String, i64), _>();
            input
                .explode(|(k, v)| Some((k, Sum { value: v, count: 1 })))
                .count()
                .map(|(k, v)| (k, v.value))
                .inspect(|s| println!("{:?}", s))
                .probe_with(&mut probe);
            handle
        });
        input.update(("a".to_string(), 1 as i64), 1);
        input.update(("a".to_string(), -1), 1);

        sync!(input, probe, worker);

        input.update(("b".to_string(), 1 as i64), 1);
        input.update(("b".to_string(), 2), -1);
        sync!(input, probe, worker);
    })

```
输出:
```
(("a", 0), 0, 1)
(("b", -1), 1, 1)
```
满足要求.

### Min/Max
求最大/最小值的情况下, 由于存在数据会被移除的情况, 所以所有数据必须保留, 比如最开始的数据流是 (1,2,3), 这时最小值
是1, 但是在后面某个时候 1 被移除了, 这时最小值是 2, 以此类推, 我们需要保留所有数据. 那要如何处理数据倾斜呢, 一种
可能的办法是对数据进行分区, 默认情况下数据按 key 值分区, 我们可以修改为 (key, hash(value) >> 56), 这样之前一个
key 一个分区会变成一个 key 对应 256 个分区, 我们计算子分区的数据后最后在聚合到一起. 这时一种行之有效的方法. 但是
随着数据量的增加, 计算时间还是会慢慢增长. 

有的时候需要特殊情况特殊处理, 对于 `min/max` 这种需求, 我们是可以有其他优化方法的, 因为 trace 中 value 本身已经
是有序的, 每次 `reduce` 相当于重组了这部分数据 (历史数据进行了合并), 这可能是一个比较耗时的操作, 我实验性的实现了
一个 `min_total` 算子, 在我的测试下, 每次运行时间基本是固定的, 时间复杂度从 O(Trace) 变成了 O(Batch).
代码在[这里](https://github.com/nooberfsh/timelyext/blob/main/src/operators/min_total.rs)
目前该算子只支持在 `TotalOrder Timestamp` 下面. `Partial Time` 有点苦难. 我看过 DD 的 `reduce` 代码,
但是还没完全看懂, 有时候看和 `Partial Time` 相关的逻辑就脑子疼, sad~~~. 等后面完全看懂了(不太确定..), 我会
尝试性的写各 `min` 算子, 同时我还要探索以下 `reduce` 本身的优化方法, 一个优化点是: `reduce` logic 中第一个参数是到某个
Time 的所有 value 值, 这一步需要需要便利所有的历史数据同时还要分配内存, 所以我想到一种方式是把这个操作 lazy 化,
从 Vec 变成某个 Iterator, 用户需要数据就从 Iterator 中去读, 在 `min` 的场景下之要读一次就好了(没有 retract).
这样能大大减少读取执行时间. 

#### Monotonic Min/Max
这种情况使用 `sum` 即可.

## 总结
这片文章总结了一些 `reduce` 相关的优化技巧, 文章中的例子都放在了这个 [repo](https://github.com/nooberfsh/dd_reduce)
大家有兴趣的可以看以下.

# 参考
[Robust Reductions in Materialize](https://github.com/frankmcsherry/blog/blob/master/posts/2020-04-01.md)


[Differential Dataflow]: https://github.com/TimelyDataflow/differential-dataflow