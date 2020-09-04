---
title: Deep Learning中大数据集加载的技巧摘要
date: 2020-09-03 19:46:27
tags:
 - Machine Learning
categories:
 - 机器学习
---

## 引入

近期在处理PCAP格式的网络抓包数据集的时候，在数据集处理方面遇到了不少的问题，主要表现为数据集巨大，有几十GB的大小，由于调试的需要免不了要多次从中提取出数据来进行训练。所以在此简单记录和总结一下其中使用到、学习到的一些处理方法。

<!---more--->

## 选择合适的读取方式

库与库之间的体制是不能一概而论的，有的库在加载数据的时候有着先天的劣势，读取同样的数据就是比其他的库要慢。在进行大量数据的读取的时候就不得不考虑使用哪一个库，例如在读取PCAP时，使用scapy就是比dpkg要慢很多，而且从磁盘中读取时速度可能会相差几十倍。类似的，使用某些库进行CSV读取也可以获得更快的速度，甚至更小的内存占用。

## 缓存

最简单的提升速度的方式是在存在大量IO操作和预处理的数据加载之后，使用某种方式进行缓存。

最简单的一个实现是使用Python的pickle模块，它可以快速地将一个Python中的基本量序列化到文件中（或者也可以用来序列化后进行网络传输等操作），不过它是Python专有的，其他语言不能读取。另外也可以使用json模块，将Python列表或字典转换成json文件，但是加载速度会比使用pickle慢。

```Python
def cache_save(variable, cache_file):
    """
    create cache for variable using pickle

    :param variable: variable need to be cached
    :param cache_file: path to cache file
    """
    try:
        with open(cache_file, "wb") as f:
            pickle.dump(variable, f, 0)
    logger.info("Cache is saved to %s", cache_file)


def cache_load(cache_file):
    """
    recover variable from cache file

    :param cache_file: path to cache file
    :return: recovered variable
    """
    with open(cache_file, "rb") as f:
        variable = pickle.load(f)
    logger.info("Cache is loaded from %s", cache_file)
    return variable
```

使用缓存时另外一个比较棘手的问题是何时从缓存中加载，而何时从数据集中重新进行读取？

如果刚对数据集进行了一定的修改，显然应该重新加载一遍数据集，但是如果加载的代码进行了一定的变更呢？如何决定是否需要重新从数据集中加载一遍？这个问题是无法预先回答的，因为很难预测一些更改是否对数据集的加载结果产生了影响。

比较简单的折衷方案是始终尝试从缓存中读取，而在需要重新建立缓存时手动删除缓存文件；又或者是在代码中添加一个BOOL变量来控制。（总之笔者还没有找到一个一劳永逸的解决方案）前者通常要求尝试检查文件是否存在，因此可以通过一个try_load来完成这一个操作。

```Python
def cache_try_load(cache_file, load_function, *args):
    """
    try to load variable from cache file.
    if not exist, call `load_function`.

    :param cache_file:
    :param load_function:
    :param args:
    :return:
    """
    try:
        return cache_load(cache_file)
    except FileNotFoundError:
        variable = load_function(*args)
        cache_save(variable, cache_file)
        return variable
```

使用这样的方法就推荐事先将一项数据的加载过程写成一个函数，以返回值作为加载的结果。

## tf.data中在运行时载入数据

通常学习通过使用`tf.data`来加载数据集的时候，大都是从自内存中的数据（数组）来构建`tf.data.Dataset`开始的。这就要求了所有数据一定要事先存在于内存之中，从而会增加内存的占用。

因此个人觉得在处理一些大数据集的时候就不应该使用`tf.data.Dataset.from_tensor_slices`或`tf.data.Dataset.from_tensors`等函数，而应该使用`tf.data.Dataset.from_generator`这样的函数，使用Python Generator来完成数据集的载入。

Tensorflow在处理一个通过生成器来创建的数据集时，总是需要读取数据的时候才载入一系列数据（也包括使用shuffle buffer时，会载入预定的数量到内存），而不会读入整个数据集。大部分时候深度学习的数据集都是逐项的，相互之间没有计算关系，所以可以逐项载入。

不过如果需要综合处理整个数据集的时候使用Generator可能就会遇到一些问题，例如对抓包数据集中的流提取统计信息，就需要持续统计处理才能得到最终的结果。但对于这样的问题，可以通过和上述的缓存配合来解决。可以先进行统计、提取出训练用的特征，逐项存入一个CSV文件中，再通过逐行读入CSV的生成器来加载数据集。

## 编码实践中的建议

对于大数据的加载和处理往往比想象中的要复杂、困难许多，有些费时，代码的组织可能成为一个大问题（至少对于笔者最近的开发情况来说）。

在进行大数据集读取，尤其是同时还需要进行一些数据处理工作的时候，有以下这些建议：

- **使用通用的、易于扩展的结构**：如果数据集可能出现变动或调整，例如出现数据集替换，或者比较常见的数据集扩展（加入一些自定义的数据集，也许不使用和源数据集一样的保存格式），就需要注意导入的方式一定要具有一定的扩展性，否则之后问题会变得很棘手。

- **分离数据载入、数据处理等步骤**：各个功能的实现不要草草合并在几个函数中，一定要注意各个部分功能的隔离。数据集的读取部分（IO）、处理部分（计算）、最终数据集的导出或者创建一个tf.data.Dataset数据集类，最好进行一定程度的分隔，方便进行替换。这样做的另一个好处是对于每一个部分的处理结果都可以进行缓存，这样如果之前几步没有更改，就可以跳过之前的步骤，从缓存中导入处理结果。另外这么做也有利于增强上一条的通用性、扩展性。