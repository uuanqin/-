## Windows Bitmap🖼

### 文件结构

> [!hint] 考点 4：Windows Bitmap 文件结构

![image.png](https://cdn.gallery.uuanqin.top/img/20231104183213.webp)
bfOffBits：是从位图文件头到位图数据的偏移量。即位图信息头 (BITMAPINFOHEADER)+ 全部颜色对照表 (RGBQUAD) 的字节数。

数据由代表位图的连续行或扫描线的字节值数组组成。每条扫描线由连续的字节组成，这些字节按从左到右的顺序代表扫描线中的像素。
代表扫描线的字节数取决于位图的宽度。
<span style="color:#ff0000">扫描线以 32bit 为边界</span>，必须用 0 填充至末尾。
- 这意味着宽度为 8 位，20 位或 30 位的单色位图将具有相同的扫描线大小：32 位
- 40 位宽的单色位图将具有 64 位的扫描线大小。

Bitmap 中的扫描线是**自底向上**存的
- 数组中的第一个字节代表 bitmap 左下角
- 数组中的最后一个字节代表 bitmap 右上角

### 位图种类

#### 灰度位图

- 每个像素使用 **8 个比特（1 字节）**。Info Header 中的 biBitCount 将为 8。
- 每个字节代表 256 种灰度阴影。白色为 FF，黑色为 00
- 实质上是 8-bit 彩色位图的特例。存在一个颜色表，其中条目 0 指向黑色，条目 255 指向白色，而介于两者之间的条目指向灰色阴影。对于一个给定大小的图像来说，8-bit 灰度位图和 8-bit 彩色位图是一样的

#### 24-Bit 彩色位图

- 每个像素使用 **24 个比特（3 个字节）**。Info Header 中的 biBitCount 将为 24。
- 每个像素 3 个字节代表红绿蓝。白色 FFFFFF，黑色 000000，红色 FF0000，绿色 00FF00，蓝色 0000FF

#### 8-Bit 彩色位图

- 每个像素使用 8 个比特，只有 256 种颜色，通过调色板实现
- 颜色表在 info header 和 Data 之间，每一对含有 4 个字节（R, G, B, 0x0）

颜色表：

![image.png|475](https://cdn.gallery.uuanqin.top/img/20231101184757.webp)

## WAV 格式 🎵

> [!hint] 考点 5：WAVE 格式

> [!hint] 考点 8：列出一个块的组件

WAV 是存储数字音频（waveform）的简单文件格式，使用了 RIFF 结构将文件内容分为不同的块（chunk）。
* 每个块均由标头和数据字节组成。
- Header 指定块数据字节的类型和大小。

它支持各种 bit 分辨率、采样率和音频通道

流行于 Windows 平台，广泛应用于处理数字音频波形程序中

![image.png](https://cdn.gallery.uuanqin.top/img/20231101184818.webp)
![image.png](https://cdn.gallery.uuanqin.top/img/20231101184830.webp)
所有使用 WAV 的应用必须能够读取两个必要块（fmt、data），并且可以有选择的忽略可选块。

程序复制一个 WAV，应该复制 WAV 中所有的块，即使有一些块最后不会被解释

次序要求：在 WAV 文件中，**除了 Format 块必须在 Data 块前外**，块的顺序没有限制

注意，format 块也许不是第一个块。一个 WAV 文件通常有三个 chunk 以及一个可选 chunk，其在文件中的排列方式依次是：
1. RIFF chunk
2. Format chunk
3. Fact chunk（附加块，可选）
4. Data chunk

所有的数据都以 8 位字节存储，以 [[🌱数据的“大端方式”和“小端方式”|小端]] 形式存于字节中

「块（的）数据」和「数据块」不是一个概念：**数据块**有「块大小」和一些「块数据」，其他所有块也是如此。

### 组成

WAV 组成：不同类型的 chunk 的集合。每个块包含头及数据，头指定了**块数据**的类型和大小，一些类型的块可能含有子块。**RIFF 文件块必须以 2 个字节（1 word）对齐。**

#### 【必含】fmt 块

fmt 块，采样格式，包含描述波形的参数，比如采样率、位分辨率和通道数

![image.png](https://cdn.gallery.uuanqin.top/img/20231101185009.webp)

```C
typedef struct {
    ID chunkID;                 // 块的标识。总是「fmt 」注意末尾有空格。
    long chunkSize;             // 块的大小（除去chunkID、chunkSize）
    short wFormatTag;           // 存储数据时使用的压缩方法
    unsigned short wChannels;   // 音频的通道数。 1:单声道; 4:四通道
    unsigned long dwSamplesPerSec;  // 采样率。标准：44.1KHZ 48KHZ 32KHZ 22.05KHZ 96KHZ
    unsigned long dwAvgBytesPerSec; // 每秒播放多少字节，用于估计RAM需要的大小。
                                    // 其值应等于 dwSamplesPerSec * wBlockAlign的上取
    unsigned short wBlockAlign; // 采样帧的大小（字节）
							    // 其值等于 wChannels * (wBitsPerSample / 8) 的上取
								// 例如：
								// 3个未压缩的通道组成一个块，每个通道2个字节，则Block align=6
    unsigned short wBitsPerSample;   // 采样点的位分辨率
    unsigned short cbSize;      // Size of the extension，扩展字段的长度
    // 也许有额外的域, 取决于 wFormatTag
} FormatChunk;
```

| 🔺wFormatTag 值 | 含义                           |
| --------------- | ------------------------------ |
| 0x0001          | WAVE_FORMAT_PCM 未压缩的数据   |
| 0x0002          | WAVE_FORMAT_ADPCM              |
| 0x0006          | WAVE_FORMAT_ALAW （电话格式）  |
| 0x0007          | WAVE_FORMAT_MULAW （电话格式） | 
| 0x55=85         | WAVE_FORMAT_MP3                |

##### WAVEFORMATEXTENSIBLE 扩展的 WAVE 格式块

```C
typedef struct {
    WAVEFORMATEX Format;
    union {
        WORD wValidBitsPerSample; // 信号精度的比特数
							      // 假设使用24bit，但只前20位有效，则wValidBitsPerSample=20
        WORD wSamplesPerBlock; // 一个音频压缩块数据的样本数
        WORD wReserved;
    } Samples;
    DWORD dwChannelMask; // 指定流中通道到扬声器位置的分配的位掩码
    GUID SubFormat; // 对每一个波形数据类型定义的 ID。例如WMA中不同压缩方式。
} WAVEFORMATEXTENSIBLE
```

位掩码 dwChannelMask：

![image.png|500](https://cdn.gallery.uuanqin.top/img/20231101185109.webp)
<span style="color:#ff0000">用于超过 2 通道或 16 分辨率的音频数据</span>：
* `wFormatTag` = FFFE（-2）
* `cbSize` = 24

以 6 通道 5.1 格式为例子：

```C
WAVEFORMATPCMEX waveFormatPCMEx;
// FormatChunk
wFormatTag = WAVE_FORMAT_EXTENSIBLE; //FFFE
wChannels = 6;
dwSamplesPerSec = 48000L;
dwAvgBytesPerSec = 864000L;          // nBlkAlign * nSamp/Sec = 18 * 48000 
wBlockAlign = 18;
wBitsPerSample = 24;                 // Container has 3 bytes
cbSize = 22;
// 以下是拓展的域
wValidBitsPerSample = 20;            // Top 20 bits have data
dwChannelMask = KSAUDIO_SPEAKER_5POINT1;
// SPEAKER_FRONT_LEFT | SPEAKER_FRONT_RIGHT |
// SPEAKER_FRONT_CENTER |SPEAKER_LOW_FREQUENCY |
// SPEAKER_BACK_LEFT | SPEAKER_BACK_RIGHT
SubFormat = KSDATAFORMAT_SUBTYPE_PCM; // Specify PCM
```

#### 【必含】data 块

音频数据，包含实际波形数据，比如所有通道的数据波形

```C
typedef struct {
    ID chunkID;
    long chunkSize;
    unsigned char waveformData[];
} DataChunk;
```

交错立体声波样本：多通道采样存储于交错的波形数据中；8bit 样本采用无符号数据表示，其他样本使用有符号数据表示。

对于多声道声音，每个声道的单个采样点是交错的。一组交错的采样点称为一个采样帧。

![image.png](https://cdn.gallery.uuanqin.top/img/20231101185123.webp)
假设存在两个通路，先存放 time1 的左通道和右通道数据，再存放 time2，以此类推。

#### 其他

fact 块（详见 [[03 音频编码基础]] 的 ADPCM 章节）、cue 块

## AVI 格式 🎵🎞

> [!hint] 考点 3：AVI 容器

AVI 是另一种 RIFF，由微软开发

音视频交织，即视频段数据紧接音频数据。这允许媒体播放器**以块来读数据而不是文件整体**。

![image.png|475](https://cdn.gallery.uuanqin.top/img/20231101185135.webp)

![image.png](https://cdn.gallery.uuanqin.top/img/20231101185202.webp)

*检查一下 hdrl 和 strl 的缩进关系。和 010 实验有点不一样*（没时间了）

注意 AVI 是 Wave 格式，必须以两个字节（1 word）对齐，填充。解码时不需要考虑填充的数据。

索引目的：为了支持随机访问

> [!hint] 考点 6：解释 FOURCC 中 "00dc"，"vids" 等标签的意思

🔺流列表中的四字节码：
- <span style="color:#ff0000">vids：video stream</span>
- auds：audio stream
- mp4v：MEPG4 visual

数据块可以直接驻留在 movi 列表中，或者被 res 列表包围。

🔺FOURCC，定义块中信息的类型：
- db：未压缩的视频帧
- d**c**：压缩的视频帧
- <span style="color:#ff0000">wb：音频流</span>

![image.png](https://cdn.gallery.uuanqin.top/img/20231101185239.webp)

> [!Example] 习题助记
> 1. What does fourcc code “vids” stands for?  
> 2. What does fourcc code “auds” stands for?  
> 3. What does fourcc code “00db” stands for?  
> 4. What does fourcc code “01wb” stands for?
> 5. ln which scemarops is wFormatTag equal to -2 in a WAVE file?
> 	音频通道大于 2，每一个抽样信号大于 16bit