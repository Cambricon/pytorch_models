# 网络训练性能优化checklist


## 文档修改记录

| 日期 | 版本 | 修改内容 | 修改人 |
| ---- | ---- | -------- | ------ |
| 2021.04.08     |   v0.1   |     初版     |    刘雨鑫    |
| 2021.04.26     |   v0.2   |  增加dump工具的Debug方法  |  唐悦然  |
|      |      |          |        |
|      |      |          |        |


## 1. 概述

在精度调试过程中我们会遇到以下几种情况：

- loss不收敛。随着训练代数增加，loss逐渐与预期的loss曲线偏离，最终不再收敛；
- loss爆炸。loss在某一代突然变成一个极大值，在此之前loss值均正常并符合预期；
- loss上升。loss在某一代开始上升，最终变为一个极大值。
- 网络突然中断挂掉

在网络的复杂环境下，以上问题通常都比较难定位，因此需要一个通用的思路和工具来针对性的解决这些问题。

## 2. 原因分析&&调试方法

接下来针对概述中遇到的精度问题，列出我们之前调试过程中定位出的原因，并给出相应的定位调试方法。

### 2.1. loss爆炸

Loss直接爆炸飞掉有几种可能性：一种是前向算子计算出错，另一种是反向算子计算出错。

定位问题方法：先看是哪一次迭代导致的loss爆炸，然后打印出这一次迭代更新前的权重，如果权重看起来数值正常，说明上一次反向计算是没有问题的，那么需要打印loss爆炸的这一次迭代每一层的正向计算输出值，定位看是哪一层计算导致的loss爆炸。

### 2.2. loss不下降

loss不下降最大可能就是权重根本没有更新梯度, 所以只需要对比更新前后的权重是否发生变化即可。

### 2.3. Loss下降但是收敛不到需要精度

这种情况比较复杂，我们总结了一下可能存在以下几种情况会导致最后训练的精度异常：

- 网络中使用的算子精度问题。

- 自适应量化算法导致精度不达标。

- 随机性算子或者算子随机性导致精度不达标。

- 超参数据集没有对齐。

针对以上可能的精度问题情况，我们总结了一个定位这种问题的标准调试流程：

- 首先需要对齐和GPU模型的超参、环境、python&&pytorch版本以及数据集。

- 如果上面超参对齐了，可以关闭自适应，用最大位宽去跑，排除自适应量化算法的干扰。

- 如果最大位宽精度还是有问题，就需要把dropout这种随机性算在放在cpu上，其他算子放在MLU上运行训练，同时比较GPU和MLU的loss的变化，尽量在训练的早期发现精度差异。

- 如果排除随机性算子依然最后训练精度不达标，最后可以使用二分法，将一半的算子放在MLU上，一半放在CPU上训练，重复这个操作缩小问题算子的范围。


### 2.4. 网络突然中断挂掉

这种情况也是我们在训练中间经常遇到的，但是挂掉也分两种情况：一种是会提示是MLU runtime或者算子库的错误，一种是直接segment fault。

- 如果是第一种情况，那么需要首先把网络设置成同步模式，并把CNNL的LOG打开, 具体可以参考：
```bash
export CATCH_ASYNC_DISABLE=1
export CNNL_MIN_LOG_LEVEL=INFO
```

- 如果是第二种情况，可能是由于框架本身程序问题或者host（cpu）端内存泄漏导致的，首先设置DEBUG模式环境变量，编译pytorch和catch为debug模式，然后可以运行valgrind查看是否存在内存泄漏：： 
```bash
export DEBUG=1
valgrind --tool=memcheck --leak-check=full --show-leak-kinds=all --log-file=XXX python XXX.py (-param XXX)
```

- 如果没有内存泄漏可以使用gdb运行程序，看具体是哪个地方导致挂掉。


## 3. 使用DUMP工具辅助定位问题

如果loss爆炸可以在特定的迭代稳定复现，或在一定范围内大概率复现，可以尝试使用Catch的DUMP工具进行调试：
使用Python的上下文管理器（即with语句）将网络中可能有故障的部分用DUMP工具覆盖（支持前向和反向算子），并在指定的迭代范围内启用DUMP工具，即可将当前范围内调用的MLU算子的输入输出信息保存到文件。推荐使用如下方式配置和使用DUMP工具：

```python

    # In function train:
    for i, (images, target) in enumerate(train_loader):
        from torch_mlu.core.dumptool import Dumper
        debug_iter=[0, 1, 2, 3]
        with Dumper(dump_dir=f"./dump_iter_{i}_rank_{args.rank}_model_{args.arch}",
                    enable = (i in debug_iter),
                    use_cpu = True,
                    level = 1) :
            ... # input pre-processing
            output = model(images)
            loss = criterion(output, targe)
            loss.backward()
        # End of Dumper
```

上述模式下，将在当前目录下生成一组按iter、rank和网络名称区分的文件夹，包含了前向反向过程中依次调用的算子，以及其输入输出的数据，并给出了非原位算子CPU的参考值。level=1提供Tensor的绝对值之和，可以用于结果比对。

除了CPU与MLU比较之外，也可以在不同iter之间比较算子的权重是否更新，以及定位loss爆炸发生的位点。

## 4. 总结

除了以上针对三种网络训练时的精度、运行问题提出的定位方法以外，适配网络的同事还需要注意的是：由于训练网络是一个大量时间成本消耗的过程，所以尽可能的缩小问题的范围，用小的测试去复现错误的环境，这样可以提高定位问题的效率。

