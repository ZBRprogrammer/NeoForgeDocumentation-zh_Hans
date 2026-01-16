# 调试性能分析器(Debug Profiler)

Minecraft提供了一个调试性能分析器(Debug Profiler)，用于提供系统数据、当前游戏设置、JVM数据、世界(level)数据和各端(sided)的刻(tick)信息，以查找耗时代码。考虑到诸如`TickEvent`和正在刻(ticking)的`BlockEntity`等因素，这对于想要查找卡顿源的模组开发者(Modders)和服务器管理员来说非常有用。

## 使用调试性能分析器(Using the Debug Profiler)

调试性能分析器使用起来非常简单。它需要使用调试键位`F3 + L`来启动性能分析器。10秒后，它会自动停止；但是，也可以通过再次按下该键位来提前停止。

:::note
自然，你只能分析实际被执行的代码路径。你想要分析的`Entities`和`BlockEntities`必须存在于世界中才能出现在结果中。
:::

停止调试器后，它会在你的运行目录(run directory)下的`debug/profiling`子目录中创建一个新的压缩文件。
文件名将按照日期和时间格式化为`yyyy-mm-dd_hh_mi_ss-WorldName-VersionNumber.zip`

## 解读性能分析结果(Reading a Profiling result)

在每个端文件夹(`client`和`server`)内，你会找到一个包含结果数据的`profiling.txt`文件。在顶部，它首先告诉你它运行了多少毫秒以及在该时间内运行了多少游戏刻。

在此之下，你会发现类似于下面片段的信息：

```
[00] tick(201/1) - 41.46%/41.46%
[01] |   levels(201/1) - 96.62%/40.05%
[02] |   |   ServerLevel[New World] minecraft:overworld(201/1) - 98.80%/39.58%
[03] |   |   |   tick(201/1) - 99.98%/39.57%
[04] |   |   |   |   entities(201/1) - 56.83%/22.49%
[05] |   |   |   |   |   tick(44717/222) - 95.81%/21.54%
[06] |   |   |   |   |   |   minecraft:skeleton(4585/23) - 13.91%/3.00%
[07] |   |   |   |   |   |   |   #tickNonPassenger 4585/22
[07] |   |   |   |   |   |   |   travel(4573/23) - 33.12%/0.99%
[08] |   |   |   |   |   |   |   |   #getChunkCacheMiss 7/0
[08] |   |   |   |   |   |   |   |   #getChunk 47227/234
[08] |   |   |   |   |   |   |   |   move(4573/23) - 40.10%/0.40%
[09] |   |   |   |   |   |   |   |   |   #getEntities 4573/22
[09] |   |   |   |   |   |   |   |   |   #getChunkCacheMiss 1353/6
[09] |   |   |   |   |   |   |   |   |   #getChunk 28482/141
[08] |   |   |   |   |   |   |   |   unspecified(4573/23) - 36.24%/0.36%
[08] |   |   |   |   |   |   |   |   rest(4573/23) - 23.66%/0.23%
[09] |   |   |   |   |   |   |   |   |   #getChunkCacheMiss 59/0
[09] |   |   |   |   |   |   |   |   |   #getChunk 65867/327
[09] |   |   |   |   |   |   |   |   |   #getChunkNow 531/2
```

像`[03] tick(201/1) - 99.98%/39.57%`这样的条目是`ProfilerFiller#push`和`pop`的结果。这意味着：

- `[03]` - 该节(section)的深度。
- `tick` - 该节的名称。
    - 如果时间段没有关联的子节，则为`unspecified`。
- `201` - 在性能分析器运行期间，此节被调用的次数。
- `1` - 此节在每个游戏刻(tick)中被调用的平均次数（向下取整）。
- `99.98%` - 相对于其父节所花费时间的百分比。
    - 对于第0层，它是一个游戏刻(tick)所用时间的百分比。
    - 对于第1层，它是其父节所用时间的百分比。
- `39.57%` - 相对于整个游戏刻所花费时间的百分比。

还有一些条目看起来像`[07] #tickNonPassenger 4585/22`，这是`ProfilerFiller#incrementCounter`的结果。这意味着：

- `[07]` - 该节(section)的深度。
- `#tickNonPassenger` - 正在递增的计数器(counter)名称。
    - 前缀`#`是自动添加的。
- `4585` - 在性能分析器运行期间，此计数器被递增的次数。
- `22` - 此计数器在每个游戏刻中被递增的平均次数（向下取整）。

## 分析你自己的代码(Profiling your own code)

调试性能分析器(Debug Profiler)对`Entity`和`BlockEntity`有基本支持。如果你想分析其他内容，可能需要像这样手动创建你自己的节(sections)：

``` java
Profiler.get().push("yourSectionName");
//你想要分析的代码
Profiler.get().pop();
```

现在你只需要在结果文件中搜索你的节名称即可。