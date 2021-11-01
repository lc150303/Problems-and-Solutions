# 使用 Kaldi 提取音素（phonemes）

[Kaldi](http://kaldi-asr.org/doc/index.html) 是一个 C++ 的语音识别工具库，里面包含了提取 MFCC 频谱、构建解码图等等工具。在 Kaldi 平台上搭建的语音识别模型有提取[音素（phonemes，简写 phones）](https://en.wikipedia.org/wiki/Phoneme)的功能。

本文给出一个利用 Kaldi 提取音素的可行流程。

## 目的
利用 Kaldi 平台上现有模型，从语音**音频数据**（.wav 文件）中提取音素。

## 难点
Kaldi 工具库是一套非常复杂的系统，其中很多工具是基于信号处理和语言学的，不像现在的神经网络这样端到端，要完全掌握 Kaldi 中的各种概念和计算方式从时间精力开销上来说是不可行的。

## 可行的办法
大体上按照官方推荐的 [Librispeech 教程](desh2608.github.io/2020-05-18-using-librispeech)走，主要不同点是处理自己的数据（步骤3）、把单词晶格文件转化为音素晶格和从 Python 中读取晶格文件。

### 1. 安装 Kaldi
首先要使用 Linux 系统。  
从 [github库](http://github.com/kaldi-asr/kaldi) 下载或克隆源码，按照[官方教程](http://kaldi-asr.org/doc/install.html#install_install)即可完成 CPU 版本的安装。如果要测试 GPU 版，则在编译 src 文件夹前把 src/cudamatrix 中 Makefile 中 TESTFILES 改为 BINFILES，即可在编译完 src 后运行 src/cudamatrix/cu-ector-test。

### 2. 下载 Librispeech 模型
从[官网模型库](http://kaldi-asr.org/models.html)下载 M13 号模型，其中的3个文件（Chain 1d，language models，i-vector extractor）都要下载。

### 3. 准备我们自己的数据集
#### a. 统一音频格式
Librispeech 是在 16kHz 采样率的位宽 16bit 的单通道音频上训练的，我们要把我们的音频保存成此格式。这里我们可以使用 python 中的 **moviepy** 和 **scipy** 包。
```python
""" 从视频中保存音轨 """
from moviepy.editor import VideoFileClip
VideoFileClip("xx.mp4").audio.write_audiofile("xx.wav", verbose=False, fps=16000, nbytes=2, ) # fps 是采样率，nbytes 是位宽

""" 从音频文件改变格式 """
from moviepy.editor import AudioFileClip
AudioFileClip("origin.wav", fps=25000).write_audiofile("target.wav", verbose=False, fps=16000, nbytes=2, )

""" 双通道变单通道 """
import numpy as np
from scipy.io import wavfile
r, data = wavfile.read("xx.wav")
data = np.mean(data, axis=1, dtype=np.int16)    # 两通道取平均，dtype 决定位宽
wavfile.write("xx.wav", r, data)
```

#### b.
详细说明可见[Kaldi 官方教程](kaldi-asr.org/doc/data_prep.html)。我们有4个文件需要创建:
1. wav.scp
1. utt2spk
1. spk2utt
1. text

其中 wav.scp 的每一行是一个样本路径，格式是
```cpp
样本名（空格）音频文件路径（\n）
如：
sample1 /home/xxx/data/sample1.wav
```
所有文件中的样本名要对应，行与行按样本名字典序排序，且同一文件中每个样本名要 unique。乱序或者有重复样本名在后面处理时会报错。

utt2spk 每一行是一个样本说话者，格式是
```cpp
样本名（空格）说话者id（\n）
如：
sample1 sample1
```
对于未知 说话人id 的数据集，可以直接采取样本名作为说话人id。

spk2utt 在采取样本名作为说话人id时与 utt2spk 完全相同，复制一份重命名即可。若是其他的说话人id，则按[官方教程](kaldi-asr.org/doc/data_prep.html)用命令
```shell
# 在 kaldi 主目录
utils/utt2spk_to_spk2utt.pl data/train/utt2spk > data/train/spk2utt
```

text 文件每行是一个样本的 transcript，如果数据集没提供则可以写任意单词，注意单词全大写。（可能并不需要该文件）如
```cpp
样本名（空格）说话文本（\n）
如：
sample1 RANDOM
```

### 4. 解码数据集获得单词晶格文件
按照官方推荐的 [Librispeech 教程](desh2608.github.io/2020-05-18-using-librispeech)，从 **Feature extraction** 做到完成 **Decoding**，其中我们要把作者的`for decode_set in test_dev93 test_eval92` 改成我们自己的数据集位置。

### 5. 把单词晶格文件转化为音素晶格文件

完成上一步骤后我们得到的单词晶格应该在 `kaldiroot/egs/wsj/s5/exp/chain_cleaned/tdnn_1d_sp/xxx/lat.*.gz`，这里的 **xxx** 为你为自己数据集指定的解码结果文件夹，如果完全按照官方的话就是 `decode_${decode_set}_tgsmall` (其中 `${decode_set}` 为 test_dev93 或 test_eval92)；`lat.*.gz` 中间的 * 是数字，范围为调用 `steps/nnet3/decode.sh` 时的 `--nj` 参数，比如从1到8。

#### a. 剪枝
先在 `kaldiroot/egs/wsj/s5` 目录中运行
```shell
source path.sh
```
然后使用 lattice-1best 函数进行剪枝，具体命令如下
```shell
run.pl JOB=1:8 my_log/1best.JOB.log   lattice-1best --acoustic-scale=0.1  \    
  "ark:gunzip -c ./exp/chain_cleaned/tdnn_1d_sp/xxx/lat.JOB.gz|" \
  "ark,t:|gzip -c>data/GRID/my_lats/lat-1best.JOB.gz"
```
逐一解释代码：
* run.pl 是 Kaldi 官方提供的用于多线程批量处理的脚本，它可以把命令后段的真实指令分发到多个进程上并行处理。
* JOB=1:8 要对应解码得到的 lat.*.gz 文件中间的数字范围，如果有 lat.1.gz 到 lat.20.gz 就改成 JOB=1:20。
* my_log/1best.JOB.log 给每个线程一个 log 文件地址，因为子线程的输出不会打印到屏幕上，如果前面是 JOB=1:8 那么这里 JOB 就是会被脚本自动替换为 1~8。
* lattice-1best 真正工作的指令，后面都是这个指令的参数。
* --acoustic-scale=0.1 解码（decoding）后保存文件时把 acoustic-scale 乘了10，这里我们乘0.1进行还原。（我不确定该参数真正正确该设多少）
* "ark:gunzip -c ./exp/chain_cleaned/tdnn_1d_sp/xxx/lat.JOB.gz|" 读取的文件地址,这里 xxx 即为上面所述的你指定的解码结果文件夹，lat.JOB.gz 会被脚本自动替换为 lat.1.gz 到 lat.8.gz。
* "ark,t:|gzip -c>data/GRID/my_lats/lat-1best.JOB.gz" 保存的剪枝后的文件地址，路径名可以任意指定，末尾文件名是带 JOB 字符的 gz 文件。

**上方这一个命令中的 JOB 是作为变量使用，该词不要更改为其他词**。

#### b. 单词晶格转为音素晶格
单词晶格文件内容为
```text
sample_name
0 1 单词 啥,啥,transition-id_transition-id_...
1 2 单词 啥,啥,transition-id_transition-id_...
```
每行第三个元素“单词”就是识别的结果，最后面是用下划线`_`分隔的数字是 transition-id，每个对应0.01秒，数量就是这个单词持续时间。  
转化为音素晶格后会变成
```text
sample_name
0 1 音素id 啥,啥,transition-id_transition-id_...
1 2 音素id 啥,啥,transition-id_transition-id_...
```
每行一个音素id，后面的 transition-id 数量为这个音素的持续时间。总行数和单词晶格文件不同，但是全文件总 transition-id 数量即音频总时长保持恒定。

具体命令如下
```shell
run.pl JOB=1:8 my_log/lat_2_phones.JOB.log lattice-align-phones --replace-output-symbols=true \
  exp/chain_cleaned/tdnn_1d_sp/final.mdl \
  "ark:gunzip -c data/GRID/my_lats/lat-1best.JOB.gz|" \
  "ark,t:|gzip -c>data/GRID/my_lats/lat_ali_phones.JOB.gz"
```
解释关键代码：
* lattice-align-phones 是真正工作的指令，按音素排列晶格文件。
* --replace-output-symbols=true 关键参数，把单词换成音素。
* exp/chain_cleaned/tdnn_1d_sp/final.mdl 解码时使用的模型，按上述流程的话这个模型位置应该是相同的，否则自行在工作目录下搜索 **final.mdl**。

### 6. 从 Python 读取晶格文件
```shell
conda install -c pykaldi pykaldi
```
如果下载不成功，就从 [conda 仓库](https://anaconda.org/Pykaldi/pykaldi/files?) 下载对应自己 python 版本的安装包，注意下载文件中的 **py某某** 指定了版本。
```shell
conda install --use_local pykaldi-某某.tar.bz2
```
耐心等待解压，会显示 0% 但是实际在工作。

然后进入 Python
```python
from kaldi.util.table import SequentialCompactLatticeReader
lat_rspec = "ark:gunzip -c /你的path-to-lat/lat_ali_phones.1.gz|"
lat_reader = SequentialCompactLatticeReader(lat_rspec)
for sample_name, value in lat_reader:
    print(sample_name)
    print(value) # 属于 sample_name 的所有音素结果，由 \n 分隔，每行的内容已经在上述步骤5.b中介绍
```
至此完工。

## 未解决的问题 
1. 步骤5.a中剪枝时 `--acoustic-scale` 到底应该设置多少。
2. 经过[教程](desh2608.github.io/2020-05-18-using-librispeech/)最后一步用 RnnLM 进行 rescoring 后的晶格文件如何转化为音素晶格？因为 RnnLM 没有 final.mdl 所以好像没法用 `lattice-align-phones` 工具。
