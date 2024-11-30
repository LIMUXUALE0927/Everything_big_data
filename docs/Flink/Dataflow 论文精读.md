# Dataflow 论文精读

[论文地址](https://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/43864.pdf)

[视频](https://www.bilibili.com/video/BV1oG411x7oG/?spm_id_from=333.999.0.0&vd_source=f3af28d1fd89af1eb80db058885d7130)

[文档](https://nxwz51a5wp.feishu.cn/docx/doxcnRSOOdA8AlbKJjzkLFlZ3ec)

## 触发器模型

很多现有的计算框架采用了 Watermark 作为窗口计算的触发器。但是 Watermark 存在两个问题：

- 有时 Watermark 上升太快。这意味着可能有 Watermark 以下的数据晚到。对于很多分布式数据源，要得到一个完美的事件时间 Watermark 是很难的，因此我们不可能依靠它得到 100% 的正确性。​
- 有时 Watermark 上升太慢。因为 Watermark 是全局进度的度量，整个数据管道的 Watermark 可能被单条数据拖慢。就算是一个事件时间偏差保持稳定的健康数据管道，根据不同的数据源，基本的偏差仍会达到几分钟甚至更多。因此，单单使用 Watermark 来判断窗口计算结果是否可以发往下游大概率会产生比 Lambda 架构更大的延迟。

​Lambda 架构在处理这一问题的方式上给了我们启示，它规避了这个问题：它不追求更快地给出正确的计算结果，它只是简单地用流处理尽快提供正确计算结果的估计值，而批处理最终会保证这个值的一致性与正确性。因此触发器基于这种思想，**通过尽可能快地先给出一个计算结果，来追求实时性；通过多次的计算结果，之后的每一次都比之前的计算结果要更准确，来追求准确性**。我们称这个特性为触发器，因为它规定了一个窗口何时触发计算得到输出结果。

---

## 增量计算模型

增量计算模型是对触发器模型的一个补充，描述了触发器模型的多次计算结果之间的关系。

​ 三种策略：​

- **丢弃**：触发计算后，窗口内容被丢弃，后续的计算结果与先前的计算结果没有关系。​
- **累积**：触发计算后，窗口内容被完整保留在持久化状态中，后续的计算结果会修正先前的结果。​
- **累积并撤回**：触发计算后，在累积语义的基础上，输出结果的拷贝也被存储在了持久化状态中。当之后窗口再次触发计算时，会先引发先前结果的撤回，然后新的计算结果再发往下游。

示例：

![](https://raw.githubusercontent.com/LIMUXUALE0927/image/main/img/202411300401386.png)

丢弃策略：

```text
First trigger firing:  [5, 8, 3]
Second trigger firing:           [15, 19, 23]
Third trigger firing:                         [9, 13, 10]
```

累计策略：

```text
First trigger firing:  [5, 8, 3]
Second trigger firing: [5, 8, 3, 15, 19, 23]
Third trigger firing:  [5, 8, 3, 15, 19, 23, 9, 13, 10]
```
