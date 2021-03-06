### STEP8: 基本音频播放

本文github[备份地址](https://github.com/dustpg/BlogFM/issues/19)

目前为止, Mapper 0的游戏应该能够顺利玩了, 就差声音了. 所以接下来就是对于音频的模拟, 算是换一个口味轻松轻松(实际并不轻松)吧. 个人还是挺喜欢8Bit风格的音乐的!

本步骤中, 如果没有特殊说明则表示是NTSC制式

这里使用音频API是XAudio2.7, 需要[DX的运行时](http://www.microsoft.com/download/en/details.aspx?id=6812). XAudio2.8是Win8自带的, 不过考虑到Win7还是使用2.7. 不过这样导致Win8以及后面的系统, 也需要下载DX的运行时库. 并且由于2.8接口上部分不兼容2.7, 但是还是用一个接口名, 这就很难受了, 只能说微软S那啥.

### 播放音频

同样因为可能读者拥有自己了解的音频API, 这里不对音频API做过多解释, 不过一般来说音频API就是读取样本缓存然后播放, 不考虑特效的话, 还是很简单的.

这一节简单介绍一下各个部分的特性.

### APU
FC的CPU叫做2A03, 之前介绍了比起6502缺了点啥, 现在就是说多了点啥: p[APU](https://wiki.nesdev.com/w/index.php/APU) - pseudo Audio Processing Unit

之所以有一个前缀p是因为没有专门处理音频的物理芯片 -- 这句话其实有歧义. 其实CPU和pAPU是一个芯片, 可以认为 pAPU + 6502 = 2A03

默认情况下, pAPU支持5个声道:

 1. 两个[方波](https://zh.wikipedia.org/wiki/方波)声道
 2. 一个[三角波](https://zh.wikipedia.org/wiki/三角波)声道
 3. 一个利用[线性反馈移位寄存器](https://zh.wikipedia.org/wiki/线性反馈移位寄存器)的噪声声道
 4. 一个用来播放[DPCM](https://en.wikipedia.org/wiki/Differential_pulse-code_modulation)的增量调制声道

还有就是Mapper额外搭载的, 这里不谈. 至于

 - PCM(脉冲编码调制)
 - APCM(自适应脉冲编码调制)
 - DPCM(差分脉冲编码调制)
 - ADPCM(自适应差分脉冲编码调制)

可以自行了解之间的区别, 不过这里为了方便, 样本格式采用IEEE单精度浮点表示.

### 方波
```
***   ***   ***   ***   ***   ***

---------------------------------

   ***   ***   ***   ***   ***
```
振幅浮点表示就是1.0和-1.0, 然后交错起来(也可以是0.0和1.0, 更为方便).

#### 占空比
感觉和矩形/方形的有点联系, 50%才叫真正的"方波", 2A03中方波拥有4种占空比(Duty Cycle):

0. 0 1 0 0 0 0 0 0 [12.5%]
1. 0 1 1 0 0 0 0 0 [25%]
2. 0 1 1 1 1 0 0 0 [50%]
3. 1 0 0 1 1 1 1 1 [25% 反相]


反相的话, 单独听是听不出与不反相有啥区别的. 不过通过混音就可能会有点区别

#### 音量
方波是有音量控制的, 最大15.

### 三角波澜
```
*       *       *       *
 *     * *     * *     * *
--*---*---*---*---*---*---*---
   * *     * *     * *     * 
    *       *       *       *
```

振幅线性地在1.0和-1.0之间来回振荡(也可以是0.0和1.0, 更为方便)

三角波没有音量控制, 取而代之的是更为细腻的长度(时长)播放控制.

### 噪声
其实直接播放上面的就是噪声了(笑). 噪声声道通过一个伪随机的位发生器发出噪声. 由于是1bit随机以及最大音量15, 生成和方波类似, 只不过是从预设到随机了.

#### 线性反馈移位寄存器(LFSR)
噪声声道有一个15bit的LFSR, 每次输出最低的bit位.算法如下:

 1. 将D0位与D1做异或运算
 2. 如果是短模式则是D0位和D6位做异或运算
 3. LFSR右移一位, 并将之前运算结果作为最高位(D14)

### DMC
用于DPCM生成, 2A03的ΔPCM大致解码流程: 

  - 一个字节为一次循环, 从D0到D7
  - 如果是1则Δ为+2, 否则Δ就是-2.
  - 是一个7bit的数据, 超过会被钳制. 

#### PCM
由于DMC相关寄存器特性, DMC声道也能播放7bit的PCM数据. 不过, 如果说播放DPCM是硬件解码的话, 播放PCM就是软件解码了. 播放完全由CPU控制, 会消耗大量CPU资源.


### 状态机实现

模拟器对于音频的实现主流的方案是利用目前数据生成样本数据再传给音频API, 同时为了避免出现'饥饿'(starvation, 声部匮乏, 出现'卡音'现象)会缓存几帧的音频再播放. 不过这里嘛...

最初的音频实现, 我们实现简单点: 状态机. 将各个部分实现为状态的切换. 比如:

 - 方波#1, 频率500Hz, 音量12, 占空比50%

这样会一直地播放这个方波, 直到方波#1的状态被改变. 音量为0的话记为停止播放:

 - 方波#1, 频率500Hz, 音量0, 占空比50%

.

三角波有点特殊, 没有音量, 所以就频率0Hz定为静音(或者20Hz~20kHz有效). 还有一点由于三角波的特殊性, 我们需要完整地播完一个'三角波', 避免出现"爆音"(状态机特有)

细节部分就在下节介绍.

### REF
 - [APU](http://wiki.nesdev.com/w/index.php/APU)
 - [APU REF](http://nesdev.com/apu_ref.txt)

 