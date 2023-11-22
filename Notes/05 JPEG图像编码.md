**Baseline 方法是迄今为止实现最广泛的 JPEG 方法。**

## 基于 DCT 的编码/解码过程

![image.png](https://cdn.gallery.uuanqin.top/img/20231102201904.webp)
编码过程：

1. 输入的组件样本为 8×8 块
2. 每个块被前向 DCT（FDCT）转换为一组 64 个值，作为 DCT 系数。
   a. 第一个值称为 DC 系数
   b. 其他 63 个值为 AC 系数
3. 然后使用量化表中 64 个对应值中的一个对 64 个系数中的每个系数进行量化
4. 然后将量化系数传递给熵编码过程，以进一步压缩

解码过程：
1. 熵解码器解码之字形的量化 DCT 系数
2. 解量化后，通过反向 DCT（IDCT）将 DCT 系数转化为 8\*8 样本块

## 前向 DCT（FDCT）

在编码器的输入中，原图像采样以 8×8 块成组，从无符号整数 $[0,2^P-1]$ 的范围转换为有符号整数 $[-2^{P-1},2^{P-1}-1]$，然后输入到 FDCT 中：

$$
F(x,y)=\frac{2C(u)C(v)}{N}\sum^{N-1}_{x=0}{\sum^{N-1}_{y=0}{f(u,v)\cos{\frac{(2x+1)u\pi}{2N}}\cos{\frac{(2y+1)v\pi}{2N}}}}
$$
其中，
$$
\begin{aligned}
C(u),C(v)=&1/\sqrt{ 2 }, 当u,v=0\\
C(u),C(v)=&1,   其他情况
\end{aligned}
$$

## 反向 DCT（IDCT）

解码器的输出中，IDCT 输出 8×8 采样块以形成重构图片：

$$
f(x,y)=\frac{2}{N}\sum^{N-1}_{x=0}{\sum^{N-1}_{y=0}{C(u)C(v)F(u,v)\cos{\frac{(2x+1)u\pi}{2N}}\cos{\frac{(2y+1)v\pi}{2N}}}}
$$
其中，
$$
\begin{aligned}
C(u),C(v)=&1/\sqrt{ 2 }, 当u,v=0\\
C(u),C(v)=&1,   其他情况
\end{aligned}
$$

## 块样本和 DCT 系数的关系

- 对一个块进行前向 DCT 计算后，64 个 DCT 系数结果被均匀量化器量化
- 每一个系数 $S_{vu}$ 的量化器步长为对应量化表中的元素 $Q_{vu}$
  ![image.png](https://cdn.gallery.uuanqin.top/img/20231102202014.webp)

## DC 编码

相邻的 8×8 块之间的 DC 系数通常有着强烈的关联性。量化的 DC 系数被编码为与前块的 DC 项的差值。这种特殊处理是值得的，因为 DC 系数包含了总图像能量的很大一部分。

![image.png](https://cdn.gallery.uuanqin.top/img/20231102202041.webp)

## 之字形扫描

所有的量化系数将以之字形顺序进行组织，将低频系数（一般为非 0）放置在高频系数前，以便于进行熵编码。

![image.png](https://cdn.gallery.uuanqin.top/img/20231102202052.webp)

## 压缩与图像质量

对于较复杂场景的彩色图像，所有基于 DCT 的操作模式通常会产生以下级别的图像质量：

- 0.25-0.5 比特/像素：中等至良好质量
- 0.5-0.75 比特/像素：良好至优秀质量
- 0.75-1.5 比特/像素：杰出质量，满足大多数的应用
- 1.5-2.0 比特/像素：和原图相差无几，满足有高质量需求的应用

根据源特性和场景内容的不同，质量和压缩会有很大的不同。

## 具有多个组件的源图像

源图像可能包含 1~255 个图像组件。每一个组件包含样本的矩形数组。

样本被定义为一个 $[0, 2^{P-1}]$ 范围的无符号整数。

图像中的所有样本必须有着相同的精度 P，对于基于 DCT 的编解码器，P 可以是 8 或 12。

![image.png](https://cdn.gallery.uuanqin.top/img/20231102202132.webp)

## 多组件交织

许多应用程序需要将显示/打印多组件图像的过程与解压过程并行。

这只有在组件在压缩数据流中交织在一起时才可行。

编码交错：如果编码器从 a 压缩一个数据单元，从 B 压缩一个数据单元，从 C 压缩一个数据单元，然后返回到 a......

![image.png](https://cdn.gallery.uuanqin.top/img/20231102202143.webp)

## 不同维度的组件交织顺序

![image.png](https://cdn.gallery.uuanqin.top/img/20231102202151.webp)
上例中，B、C 在水平方向上与 A 相比少了一半样本。在这个例子中，A 的两个数据单元和 B、C 的各一个单元进行交织。

> [!hint] 考点 28：JPEG 编码与解码过程：最小编码单元、操作模式、DC 熵编码、AC 熵编码、游程、SIZE、EOB

## 最小编码单元 MCU

在基于 DCT 的编解码器中，**数据单元**是一个 8x8 的样本块。

<span style="color:#ff0000">最小编码单元 MCU</span>：最小的交错**数据单元**组。对于非交错数据，MCU 是一个数据单元。对于交错数据，MCU 是由扫描中组件的采样因子定义的数据单元序列。交错数据是 MCU 的有序序列，MCU 中包含的数据单元数由交错的元件数及其相对采样因子决定。

![image.png](https://cdn.gallery.uuanqin.top/img/20231102202208.webp)

当两个或更多的组件进行交织时，每一个组件 $C_i$ 被 $V_i$ 个数据单元划分为 $H_i$ 的矩形区域。**最大的组件交织数为 4，且每个 MCU 的数据单元最大值为 10**：$$\sum_{所有参与交织的i}{H_i\times V_i}\le 10$$（注：置于 10 这个数字是规定的，没有理由）

## 操作模式

🔺有四种不同的操作模式，在这些模式下定义了各种编码过程：
- 基于 DCT 的顺序模式（从上到下，一块一块编码）
- 基于 DCT 的渐进模式（从轮廓到细节）
- 无损模式
- 层次模式

![image.png](https://cdn.gallery.uuanqin.top/img/20231102202314.webp)

### 顺序模式与渐进模式

在顺序模式中，每一个图片组件在单个扫描中编码。

在渐进模式中，每一个图片组件在多次扫描中编码。第一次扫描中编码粗糙，但可识别的图像版本可以快速传输，且通过后续的扫描进行改进，直到达到由量化表确定的图像质量水平。

<span style="color:#ff0000">有两种互补的方法可以对量化 DCT 系数块进行部分编码</span>：

1. 在给定的扫描中，只有之字形序列特定的系数带需要进行编码
2. 在给定的扫描中，无需将当前频带内的系数编码为完全（量化）精度。
   1. 最重要的 N 个比特可以在第一次扫描中编码
   2. 在随后的扫描中，次重要的比特再进行编码
   3. 这个过程叫做「连续逼近」

这两种程序可以单独使用，也可以灵活地混合使用。

量化 DCT 系数的传递：

![image.png](https://cdn.gallery.uuanqin.top/img/20231102202328.webp)
光谱选择与逐次逼近：

![image.png](https://cdn.gallery.uuanqin.top/img/20231102202337.webp)

### 层次模式

层次模式以多种分辨率的方式为图片提供金字塔型的编码。每一种的分辨率与其相邻的编码在水平或垂直维度或两者上相差两倍：

1. 对原始图像在每个维度上按所需的 2 的倍数进行滤波和下采样。
2. 使用顺序 DCT、渐进式 DCT 或无损编码器之一对这个减小尺寸的图像进行编码。
3. 解码此缩小尺寸的图像，然后使用接收器必须使用的相同插值滤波器对它进行水平和/或垂直 2 插值和上采样。
4. 使用此上采样图像作为该分辨率下的原始图像的预测，并使用前面介绍的顺序 DCT，逐行 DCT 或无损编码器之一对差异图像进行编码。
5. 重复第 3 步和第 4 步，直到图像的全分辨率被编码

## 基线顺序熵编码

在基线顺序编码器中，FDCT、量化、DC 差分以及之字形排序步骤之后，是熵编码。

在熵编码之前，通常只有很少的非零系数和很多零值系数。熵编码的任务是更有效地编码这些系数。

基线顺序熵编码有两个步骤：
1. 将量化 DCT 系数转换为中间符号序列（游程编码）
2. 为符号分配可变长度代码（哈夫曼编码）

### AC 系数

每一个非零的 AC 系数这样编码：
- `RUN-LENGTH`：之字形扫描序列中，被表示的非零 AC 系数前，连续数字 0 的长度。
  - `RUN-LENGTH` 代表 <span style="color:#ff0000">0~15</span>，Symbol-1 中 `(15, 0)` 代表 `RUN-LENGTH`=16。
  - `(0,0)` 代表 `EOB`（块结束），可以将其视为“转义”符号。
- `SIZE`：编码 `AMPLITUDE` 所用的比特数

![image.png](https://cdn.gallery.uuanqin.top/img/20231102202410.webp)
> [!example] 更多例子
> 0005 → (3,3)(5)  
> 000003 → (5,2)(3)  
> 00,-12 →  (2,4)(-12)
> 0..(16 个 0)..0 1 → (15,0)(0,1)(1)
> 0..(17 个 0)..0 1 → (15,0)(1,1)(1)
> 0..(18 个 0)..0 -3 → (15,0)(2,2)(-3)
> 00000 3 0..0 → (5,2)(3)(0,0)
> 0013 →  (2,1)(1) (0,2)(3)

### AMPLITUDE 和 SIZE 的范围

量化 AC 系数的取值范围决定了 AMPLITUDE 和 SIZE 信息必须表示的值的范围。对 8×8 FDCT 方程的数值分析表明，如果 64 点 (8×8 块) 输入信号包含 N 位整数，则输出数字的非小数部分 (DCT 系数) 最多可以增长 3 位。这也是量化 DCT 系数的最大可能大小。

基线顺序在 $[-2^7,2^7-1]$ 范围内有 8 位整数源样本，因此量化的 AC 系数幅度由 $[-2^{10},2^{10}-1]$ 范围内的整数覆盖。
带符号整数编码使用长度为 1 到 10 位的 symbol-2 AMPLITUDE 码，因此 SIZE 也代表 1 到 10 的值。RUNLENGTH 表示从 0 到 15 的值。

### DC 系数

8×8 样本块的差分 DC 系数的中间表示结构类似：
- Symbol-1 只表示 SIZE 信息
- Symbol-2 表示振幅信息

![image.png](https://cdn.gallery.uuanqin.top/img/20231102202435.webp)

因为 DC 系数是差分编码的，所以它覆盖的整数值 $[-2^{11},2^{11}-1]$ 是 AC 系数的两倍，因此必须为 DC 系数增加一个附加的电平。  
因此，DC 系数大小的 symbol–1 表示从 0 到 11 的值。

> **差分编码**（differential encoding）指的是对数字数据流，除第一个元素外，将其中各元素都表示为各该元素与其前一元素的差的编码。差分编码的简单例子是储存序列式资料之间的差异（而不是储存资料本身）：不存“2, 4, 6, 9, 7”，而是存“2, 2, 2, 3, -2”。

### 可变长度熵编码

对于 DC 和 AC 系数，每个 symbol-1 均使用来自 Huffman 表集中的可变长度代码（VLC）进行编码。 每个 symbol-2 均使用“可变长度整数”（VLI）码进行编码。

VLCs 和 VLIs 是具有可变长度的代码，但 VLI 不是霍夫曼代码。一个重要的区别是，VLC（ Huffman code）的长度直到解码才知道，而 VLI 的长度存储在其前一个 VLC 中。

Huffman 码必须在外部指定为 JPEG 编码器的输入。

请注意，Huffman 表在数据流中的表示形式是一种间接规范，解码器在解压缩之前必须以此间接规范来构造表（解码时需要重新构建 Huffman 树进行解码）。 JPEG 标准包括一组 Huffman 表的示例，但这不是强制性的。

### 基线编码例子

![image.png](https://cdn.gallery.uuanqin.top/img/20231102202510.webp)
> [!hint] 考点 29：JPEG 交换格式（JIF）：图像、帧、扫描和标记

## 图像、帧和扫描的关系

- 压缩的图像数据只包含一张图片。
- **在渐进模式和顺序模式编码过程中，一张图片只包含一帧。**
- **在层次模式中，一张图片可以包含多帧。**
- 一帧可以包含一个或多个扫描。
- 顺序模式下，一个扫描包含一个完整的、单个/多个图像组件的编码。
- <span style="color:#ff0000">在层次模式中：当一张图像的三个组件非交织时，一帧包含三个扫描；如果三个组件交织一起，那么一帧包含一趟扫描。</span>
- 一帧也可以包含两趟扫描：一趟是非交织的组件、另一趟为两组件交织。

## 标记

**标记用于标识压缩数据格式的各种结构部分。**

所有的标记赋予两个字节编码：0xFF+ 不等于 0 或 0xFF 的字节。

标记段包含一个标记以及相关参数的序列。标记段的第一个参数是两字节长的参数，它指定了标记段的字节数（除去两字节标记后的参数长度）

被 SOF 和 SOS 标记码标识出的标记段被视为头（headers）：分别为帧报头和扫描报头。

SOI（0xFFD8）：压缩图片开始标记
EOI（0xFFD9）：压缩图片的结束标记

### 高级语法

基于顺序 DCT、渐进式 DCT 和无损操作模式的语法：
![image.png](https://cdn.gallery.uuanqin.top/img/20231102202539.webp)

#### 帧头语义

帧报头应该出现在帧的开始。该报头指定源图像特征、帧中的组件和每个组件的采样因子，并指定从中检索要与每个组件一起使用的量化表的目标。

![image.png](https://cdn.gallery.uuanqin.top/img/20231102202601.webp)

| 标记结构 | 长度（字节） | 解释                                                              |
| -------- | ------------ | ----------------------------------------------------------------- |
| 0xFFC0   | 2            | SOF marker                                                        |
| Lf       | 2            | Frame header length, not including the first two bytes 0xFF, 0xC0 |
| P        | 1            | Sampling precision, equals 0x08 in a baseline system              |
| Y        | 2            | Image height                                                      |
| X        | 2            | Image width                                                       |
| Nf       | 1            | Number of components in a frame. 1 (grey scale) or 3 (color)      |
| C1       | 1            | Component 1                                                       |
| (H1,V1)  | 1            | Horizontal and vertical sampling factor                           |
| Tq1      | 1            | Quantization table                                                |
| C2       | 1            | Component 2                                                       |
| (H2,V2)  | 1            | Horizontal and vertical sampling factor                           |
| Tq2      | 1            | Quantization table                                                |
| ⋯        | ⋯            | ⋯                                                                 |

#### 扫描头语义

扫描头应在扫描开始时出现。这个报头指定扫描中包含哪些组件，指定从中检索要与每个组件一起使用的熵表的地址。

![image.png](https://cdn.gallery.uuanqin.top/img/20231102202705.webp)

| 标记      | 长度（字节） | 解释                                                                                                                                                         |
| --------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 0xFFDA    | 2            | SOS marker                                                                                                                                                   |
| Ls        | 2            | Scan header length, not including the first two bytes 0xFF, 0xDA                                                                                             |
| Ns        | 1            | Number of components in a scan, in a baseline system, Ns=Nf (Number of components in a frame)                                                                |
| Cs1       | 1            | Component number in a scan                                                                                                                                   |
| (Td1,Ta1) | 1            | Tdn: the four most significant bits, used to select DC entropy coding table <br>Tan: the four least significant bits, used to select AC entropy coding table |
| ⋯         |              |                                                                                                                                                              |
| Ss        | 1            | Default values are [00] [3F] [00] in a baseline system                                                                                                       |
| Se        | 1            | Default values are [00] [3F] [00] in a baseline system                                                                                                       |
| (Ah,Al)   | 1            | Default values are [00] [3F] [00] in a baseline system                                                                                                       |

#### DQT 标记段语义

定义量化表 (DQT) 标记段，用于定义一个或多个量化表。

![image.png](https://cdn.gallery.uuanqin.top/img/20231102202826.webp)

| 标记    | 长度（字节） | 解释                                                                                                                                     |
| ------- | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| 0XFFDB  | 2            | DQT marker                                                                                                                               |
| Lq      | 2            | Quantization table length， not including 0XFF， 0XDB                                                                                    |
| (Pq,Tq) | 1            | Quantization table element precision <br>Pq=0, 8 bits for Q0~Qn, Pq=1, 16 bits for Qt; <br>Tq: Quantization table destination identifier |
| Q0      | 1 or 2       | Quantization table element‐Specifies the kth element out of 64 elements                                                                  |
| Q1      | 1 or 2       | Quantization table element‐Specifies the kth element out of 64 elements                                                                  |
| Qn      | 1 or 2       | Quantization table element‐Specifies the kth element out of 64 elements                                                                  |

#### 哈夫曼表规范语法

哈夫曼表标记 (DHT) 段定义了一个或多个霍夫曼表规范。

![image.png](https://cdn.gallery.uuanqin.top/img/20231102202912.webp)

| 标记    | 长度（字节） | 解释                                                                                                                                                                                                                           |
| ------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 0xFFC4  | 1            | DHT marker                                                                                                                                                                                                                     |
| Ls      | 2            | Huffman table definition length, not including 0xFF, 0xC4                                                                                                                                                                      |
| (Tc,Th) | 1            | TC: Table class – 0 = DC table or lossless table, 1 = AC table. <br>Th:Huffman table destination identifier <br>Th Specifies one of four possible destinations at the decoder into which the Huffman table shall be installed. |
| L 1     | 1            | Number of Huffman codes of length i                                                                                                                                                                                            |
| ⋯       | ⋯            |                                                                                                                                                                                                                                |
| L 16    | 1            |                                                                                                                                                                                                                                |
| V 1     | 1            | Value associated with each Huffman code, t=L1+L2+…L16                                                                                                                                                                          |
| ...     | ...          |                                                                                                                                                                                                                                |
| V t     | 1            |                                                                                                                                                                                                                                |

## JPEG 文件

> [!hint] 考点 30：JPEG 文件交换格式（JFIF）

到目前为止，我们描述的文件格式被称为「JPEG 交换格式 (JIF)」。然而，这种“纯粹”的文件格式很少使用，主要是因为，这个标准的某些缺点:

- 色彩空间定义
- 组件子采样注册
- 像素宽高比定义

JPEG 文件交换格式（JFIF）解决了 JIF 的局限性。JFIF 文件中的图像数据使用 JPEG 标准中的技术进行压缩，因此 JFIF 有时被称为“JPEG/JFIF”。

JPEG 文件交换格式是一种最小的文件格式，它使 JPEG 比特流能够在各种平台和应用程序之间进行交换。

此简化格式的唯一目的是允许交换 JPEG 压缩图像。

尽管 JPEG 文件交换格式 (JFIF) 的语法支持任何 JPEG 过程，但强烈建议将 JPEG 基线过程用于文件交换，这确保了与所有支持 JPEG 的应用程序的最大兼容性。

**JPEG 文件交换格式与标准的 JPEG 交换格式完全兼容。唯一的额外要求是必须在 SOI 标记之后出现 APP0 标记。**

JFIF 文件使用 APP0 标记段，并在帧头中限制某些参数，定义如下：

- 长度、标识符、版本、单位、X 密度、Y 密度、X 缩略图、Y 缩略图、(RGB)n

![image.png](https://cdn.gallery.uuanqin.top/img/20231102203032.webp)

![image.png](https://cdn.gallery.uuanqin.top/img/20231102203038.webp)

![image.png](https://cdn.gallery.uuanqin.top/img/20231102203043.webp)

## 编码过程

编码图像的流程

![image.png](https://cdn.gallery.uuanqin.top/img/20231102203054.webp)
编码帧的流程
![image.png](https://cdn.gallery.uuanqin.top/img/20231102203101.webp)
编码扫描的流程
![image.png](https://cdn.gallery.uuanqin.top/img/20231102203117.webp)
编码重启间隔的流程
![image.png](https://cdn.gallery.uuanqin.top/img/20231102203125.webp)
对最小编码单元进行编码的过程
![image.png](https://cdn.gallery.uuanqin.top/img/20231102203134.webp)
使用哈夫曼编码 AC 系数的过程
![image.png](https://cdn.gallery.uuanqin.top/img/20231102203141.webp)
非零 AC 系数的顺序编码过程
![image.png](https://cdn.gallery.uuanqin.top/img/20231102203149.webp)

## 解码过程

解码压缩图像数据的过程
![image.png](https://cdn.gallery.uuanqin.top/img/20231102203157.webp)
解码一帧的过程
![image.png](https://cdn.gallery.uuanqin.top/img/20231102203206.webp)
解码扫描的过程：
![image.png](https://cdn.gallery.uuanqin.top/img/20231102203212.webp)

解码重启间隙的过程：
![image.png](https://cdn.gallery.uuanqin.top/img/20231102203223.webp)
解码 MCU 的过程：
![image.png](https://cdn.gallery.uuanqin.top/img/20231102203230.webp)
> [!example] 习题助记
> 1. What are markers used for in JIF?  
> 标识压缩数据格式中不同的结构部分  
> 2. Describe the full name of SOI,EOI,SOF,SOS  
> Start of image marker,End of image marker,Start of Frame marker,Start of Scan marker  
> 3. What is JPEG file interchange format used for?  
> 定义分辨率颜色等相关参数，为了能在不同应用和平台转换  
> 4. An image contains ___ frame in the cases of sequential and progressive coding processes.  
> one frame