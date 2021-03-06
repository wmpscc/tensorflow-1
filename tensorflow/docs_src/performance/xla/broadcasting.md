# 语义广播

本文档描述 XLA 中 语义的广播是如何工作的。

## 何为广播？

广播是让不同形状的数组的变得相容从而参与算术操作的一种过程。这个术语来自于 Numpy [(广播)](http://docs.scipy.org/doc/numpy/user/basics.broadcasting.html)。

广播在一些操作中会用到，比如不同阶的多维数组之间的操作，或具有不同但相容的形状的多维数组之间的操作。以加法 `X+v` 为例，这里 `X` 是一个矩阵（2阶数组），`v` 是一个矢量（1阶数组）。为执行逐个元素的加法，XLA 需要将矢量 `v` "广播" 为和 `X` 一样的秩，即将 `v` 复制多次。此矢量的长度必需至少与矩阵的某一维度相同。

比如：

    |1 2 3| + |7 8 9|
    |4 5 6|

此矩阵的维度为 (2,3)，矢量的维度为 (3)。矢量通过逐行复制得到和矩阵相同的秩：

    |1 2 3| + |7 8 9| = |8  10 12|
    |4 5 6|   |7 8 9|   |11 13 15|

在 Numpy 中，这个过程被称为 [广播](http://docs.scipy.org/doc/numpy/user/basics.broadcasting.html)。

## 广播的原则

XLA 是一个使用 XLA 语言的底层基础设施，它要求语法尽可能地严格和显式声明，避免隐式变换和某些 "神奇" 的功能，即使它们会让一些计算稍微容易定义，但带来代价是，在代码中引入了过多的假设，长远来看是难于改变的。如果确有必要，可以在客户层封装代码中加入这些隐式的和神奇的功能。

对于广播而言，当操作不同秩的数组时，我们要求采用显式地指定广播。这一点和 Numpy 不同，后者会尽可能地推断出广播规范。

## 将低阶数组广播为高阶数组

**标量**总是可以广播为数组而无需显式指定维度的广播规范。因而，一个标量和一个数据之间的逐个元素二元操作相当于将此标量作用于数组的每个元素。比如，将一个标量加上一个矩阵上，相当于产生一个矩阵，其每个是原矩阵相应元素加上该标量。

    |1 2 3| + 7 = |8  9  10|
    |4 5 6|       |11 12 13|

在二元操作中，大部分广播需求可通过一个维度元组来得到。当操作的输入具有不同的秩时，此广播元组指定了**更高阶**数组中的哪些维度与**更低阶**数组中的维度匹配。

还是考虑前面的示例，我们将加法中的标量换成维度为 (3) 的矢量，矩阵维度为 (2,3)。**如果不指定广播，这个操作是非法的。** 为了正确地进行矩阵-矢量加法，需要将广播维度指定为 (1)，意思是矢量的维度与矩阵的第 1 维匹配。在 2D 中，如果维度 0 视为行，维度 1 视为列，则此广播意味着此矢量的每一个元素变成一列，列的长度与输入矩阵行数相同：

    |7 8 9| ==> |7 8 9|
                |7 8 9|

考虑一个更复杂的例子，让一个 3 元素矢量（维度为 (3)）与一个 3x3 矩阵（维度为 (3,3)）相加。这时，有两种可能的广播方式：

(1) 维度为 1 的广播。矢量的每个元素变成一列，此矢量为矩阵的每一行复制一次。

    |7 8 9| ==> |7 8 9|
                |7 8 9|
                |7 8 9|

(2) 维度为 0 的广播。矢量的每个元素变成一行，此矢量为矩阵的每一列复制一次。

     |7| ==> |7 7 7|
     |8|     |8 8 8|
     |9|     |9 9 9|

> 注意：当让一个 2x3 矩阵加上一个 3 元素的矢量时，维度为 0 的广播是非法的。

广播维度可以是一个元组，用于描述秩更小的形状如何广播为一个更大秩的形状。比如，给定一个 2x3x4 的方阵和一个 3x4 矩阵，广播元组 (1,2) 表示让矩阵的维度匹配到方阵的第 1 维和第 2 维。

这种广播类型用于 `ComputationBuilder` 中的二元操作，使用时需要指定 `broadcast_dimensions` 参数。比如，参见源码 [ComputationBuilder::Add](https://www.tensorflow.org/code/tensorflow/compiler/xla/client/computation_builder.cc)。
在 XLA 源码中，这种广播类型有时候称为 "InDim" 广播。

### 形式化定义

广播允许低阶数组匹配高阶数组，即指定高阶数组中的哪些维度用于匹配。比如，对于维度为 MxNxPxQ 的数组，维度为 T 的矢量可以按下列方式匹配：

             MxNxPxQ

    3 维:          T
    2 维:        T
    1 维:      T
    0 维:    T

在每一种情况中，T 必须与高阶数组的相应维度相等。然后，矢量的值会被传播到其它的所有的维度。

为了将 TxV 矩阵匹配到 MxNxPxQ 数组上，需要用到一对广播维度：

             MxNxPxQ
    2,3 维:      T V
    1,2 维:    T V
    0,3 维:  T     V
    等等

广播元组中的维度的顺序必须与低阶数组保持一致。元组中的第一个元素指的是高阶数组中的那个维度必须与低阶数组的第 0 维匹配；第二个元素匹配第 1 维，依此类推。广播维度还必须是严格递增的。比如，在前面的示例中，不允许让 V 匹配 N 且让 T 匹配 P；让 V 同时匹配 P 和 N 同样是非法的。

## 广播具有退化维度的秩相似的数组

一个常见的广播问题是广播具有相同的秩但是不同的维度大小的两个数组。类似于 Numpy 的规则，只有在两个数组**相容**的条件下这种广播才有可能。两个数组的所有维度相容时，它们才是相容的。两个维度相容的条件是：

  * 它们相等，或
  * 其中之一为 1 (即"退化"的维度)

当两个相容的数组相遇时，它们的操作结果的形状的各个维度都为输入在各维度上的最大值。

示例：

  1.  (2,1) 和 (2,3) 广播为 (2,3).
  2.  (1,2,5) 和 (7,2,5) 广播为 (7,2,5)
  3.  (7,2,5) 和 (7,1,5) 广播为 (7,2,5)
  4.  (7,2,5) 和 (7,2,6) 不相容，无法广播

其中有一种特例，即每个输入数组都在一个不同的维度上具有退货的维度。这时，结果为它们的"外操作"：(2,1) 和 (1,3) 广播为 (2,3)。更多示例，参考 [Numpy 关于广播的文档](http://docs.scipy.org/doc/numpy/user/basics.broadcasting.html)。

## 广播的复合

一个低阶数组到高阶数组的广播和退化维度的广播可用于同一个二元操作上。比如，大小为 4 的矢量和大小为 1x2 的矩阵可以用广播维度 (0) 来实现相加：

    |1 2 3 4| + [5 6]    // [5 6] 是一个 1x2 矩阵，不是矢量

首先，矢量通过传播维度广播为 2 阶矩阵。单个值的广播维度 (0) 表示矢量的 0 维匹配矩阵的 0 维。这就会产生一个大小为 4xM 的矩阵，其中 M 用于匹配 1x2 数组中的相应维度的大小。因而，产生了一个 4x2 矩阵：

    |1 1| + [5 6]
    |2 2|
    |3 3|
    |4 4|

然后，"退化维度广播" 将 1x2 矩阵的零维广播并匹配右手边的矩阵的相应维度大小：

    |1 1| + |5 6|     |6  7|
    |2 2| + |5 6|  =  |7  8|
    |3 3| + |5 6|     |8  9|
    |4 4| + |5 6|     |9 10|

更复杂的一个例子是将一个大小为 1x2 的矩阵加到一个大小为 4x3x1 的数组上，广播维度为 (1,2)。首先，1x2 矩阵通过广播维度变为 3 阶方阵，这是一个大小为 Mx1x2 的中间结果，其中 M 由更大的那个操作数（这里是 4x3x1 的数组）的大小决定，因而得到 4x1x2 的中间数组。M 在零维上（最左边的维度），是因为 1 维和 2 维都被映射到了原来的 1x2 的矩阵上。这个中间数组可以通过退化维度广播来加到 4x3x1 矩阵上，最后产生一个 4x3x2 数组。

