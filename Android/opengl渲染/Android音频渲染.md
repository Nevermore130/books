### SDK 层的音频渲染 ###

Android 系统在 SDK 层（Java 层提供的 API）为开发者提供了 3 套常用的音频渲染方法，分别是：MediaPlayer、SoundPool 和 AudioTrack

* MediaPlayer 适合在后台长时间播放本地音乐文件或者在线的流式媒体文件。它的封装层次比较高，使用方式也比较简单。
* SoundPool 也是一个端到端的音频播放器，优点是：延时较低，比较适合有交互反馈音的场景，适合播放比较短的音频片段，比如游戏声音、按键声、铃声片段等，**它可以同时播放多个音频。**
* AudioTrack 是直接面向 PCM 数据的音频渲染 API，所以也是一个更加底层的 API，提供了非常强大的控制能力，适合低延迟的播放、流媒体的音频渲染等场景，由于是直接面向 PCM 的数据进行渲染，**所以一般情况下需要结合解码器来使用。**

### NDK 层的音频渲染 ###

Android 系统在 NDK 层（Native 层提供的 API，即 C 或者 C++ 层可以调用的 API）提供了 2 套常用的音频渲染方法，分别是 **OpenSL ES 和 AAudio，它们都是为 Android 的低延时场景（实时耳返、RTC、实时反馈交互）而设计**的

* OpenSL ES：是 Khronos Group 开发的  OpenSL ES™ API 规范的实现，API 接口设计会有一些晦涩、复杂，目前 Google 已经不推荐开发者把 OpenSL ES 用于新应用的开发了，在 Android8.0 系统以下以及一些碎片化的 Android 设备上它具有更好的兼容性
* AAudio：专门为低延迟、高性能音频应用而设计的，API 设计精简，是 Google 推荐的新应用构建音频的应用接口

### AudioTrack ###

只允许输入 PCM 裸数据。与 MediaPlayer 相比，对于一个压缩的音频文件（比如 MP3、AAC 等文件），它**需要开发者自己来实现解码操作和缓冲区控制**

看一下 AudioTrack 的工作流程：

1. 根据音频参数信息，配置出一个 AudioTrack 的实例。
2. 调用 play 方法，将 AudioTrack 切换到播放状态。
3. 启动播放线程，循环向 AudioTrack 的缓冲区中写入音频数据。
4. 当数据写完或者停止播放的时候，停止播放线程，并且释放所有资源。



#### 第一步：配置 AudioTrack ####

```c
public AudioTrack(int streamType, int sampleRateInHz, int channelConfig,
        int audioFormat, int bufferSizeInBytes, int mode);
```

* streamType，在 Android 手机上有多重音频管理策略，比如你按一下手机侧边的按键或者在系统设置中，可以看到有多个类型的音量管理，这其实就是不同音频策略的音量控制展示。**当系统有多个进程需要播放音频的时候，管理策略会决定最终的呈现效果**，该参数的可选值以常量的形式定义在类 AudioManager 中，主要包括：

```
STREAM_VOCIE_CALL：电话声音
STREAM_SYSTEM：系统声音
STREAM_RING：铃声
STREAM_MUSCI：音乐声
STREAM_ALARM：警告声
STREAM_NOTIFICATION：通知声
```

* sampleRateInHz，采样率，**即播放的音频每秒钟会有多少次采样**，可选用的采样频率列表为：8000、16000、22050、24000、32000、44100、48000 等。采样率越高声音的还原度就越高，普通的语音通话可能 16k 的采样频率就够了，但是如果高保真的场景，比如：K 歌、音乐、短视频、ASMR 等，就需要 44.1k 以上的采样频率，你可以根据自己的应用场景进行合理的选择。

* channelConfig，声道数（通道数）的配置，可选值以常量的形式配置在类 AudioFormat 中，**常用的是 CHANNEL_IN_MONO（单声道）、CHANNEL_IN_STEREO（双声道）。因为现在大多数手机的麦克风都是伪立体声采集，考虑到性能，我建议你使用单声道进行采集**，然后在声音处理阶段（比如混响、HRTF 等效果器）转变为立体声的效果。

* audioFormat，这个参数是用来配置“数据位宽”的，即采样格式，可选值以常量的形式定义在类 AudioFormat 中，分别为 ENCODING_PCM_16BIT（16bit）、ENCODING_PCM_8BIT（8bit）。注意，前者是可以保证兼容所有 Android 手机的
* bufferSizeInBytes，它配置的是 AudioTrack 内部的音频缓冲区的大小，AudioTrack 类提供了一个帮助开发者确定这个 bufferSizeInBytes 的函数，原型如下

```c
int getMinBufferSize(int sampleRateInHz, int channelConfig, int audioFormat); 
```

* mode，AudioTrack 提供了两种播放模式，可选值以常量的形式定义在类 AudioTrack 中，一个是 MODE_STATIC，需要一次性将所有的数据都写入播放缓冲区，简单高效，通常用于播放铃声、系统提醒的音频片段；另一个是 **MODE_STREAM，需要按照一定的时间间隔不间断地写入音频数据，理论上它可用于任何音频播放的场景。在实际开发中，建议采用 MODE_STREAM 这种播放模式。**

#### 第二步：将 AudioTrack 切换到播放状态 ####

需要先判断 AudioTrack 实例是否初始化成功，如果当前状态是初始化成功的话，那么就调用它的 play 方法，切换到播放状态

```c
if (null != audioTrack && audioTrack.getState() != AudioTrack.STATE_UNINITIALIZED)        
{
    audioTrack.play();
}
```

#### 第三步：开启播放线程 ####

切换为播放状态之后，需要开发者自己启动一个线程，用于向 AudioTrack 里面送入 PCM 数据

```c
playerThread = new Thread(new PlayerThread(), "playerThread");
playerThread.start();
```

来看这个线程中执行的任务，代码如下：

```java
class PlayerThread implements Runnable {
    private short[] samples;
    public void run() {
        samples = new short[minBufferSize];
        while(!isStop) {
            int actualSize = decoder.readSamples(samples);
            audioTrack.write(samples, actualSize);
        }
    }
}
```

代码中的 decoder 是一个解码器实例,假设已经构建成功，然后从解码器中拿到 PCM 采样数据，最后调用 write 方法写入 AudioTrack 的缓冲区中。循环往复地不断送入 PCM 数据，音频就能够持续地播放了。

这个 write 方法是阻塞的，比如：**一般写入 200ms 的音频数据需要执行接近 200ms 的时间，所以这要求在这个线程中不应该做更多额外耗时的操作，比如 IO、等锁。**

#### 第四步：销毁资源 ####

```c
if (null != audioTrack && audioTrack.getState() != AudioTrack.STATE_UNINITIALIZED)   
{
    audioTrack.stop();
}

isStop = true;
if (null != playerThread) {
    playerThread.join();
    playerThread = null;
}
//释放 AudioTrack
audioTrack.release();
```



### Oboe 介绍 ###

Google 推出了自己的 Oboe 框架。Oboe 使用和 AAudio 近乎一致的 API 接口为开发者封装了底层的实现，自动地根据当前 Android 系统来选择 OpenSL ES 还是 AAudio，当然也给开发者提供了接口，开发者可以自由地选择底层的实现。**Oboe 的整体 API 接口以及设计思想与 AAudio 一致**

##### 集成 Oboe 到工程里 #####

```groovy
dependencies {
    implementation 'com.google.oboe:oboe:1.6.1'
}
```

在 CMake 中增加头文件引用与库的链接：

```cmake
# Find the Oboe package
find_package (oboe REQUIRED CONFIG)
# Specify the libraries which our native library is dependent on, including Oboe
target_link_libraries(native-lib log oboe::oboe)
```

业务代码中引入必要的头文件：

```c
#include <oboe/Oboe.h>
```

### 在工程里使用 Oboe ###

首先，我们通过 AudioStreamBuilder 来创建 Stream，AudioStreamBuilder 是按照 Builder 设计模式设计的类

```c++
oboe::AudioStreamBuilder builder;
builder.setPerformanceMode(oboe::PerformanceMode::LowLatency)
  ->setDirection(oboe::Direction::Output)//播放的设置
  ->setSharingMode(oboe::SharingMode::Exclusive)//独占设备，对应的是Shared
  ->setChannelCount(oboe::ChannelCount::Mono)//单声道
  ->setFormat(oboe::AudioFormat::Float);//格式采用Float，范围为[-1.0，1.0]，还有一种是I16，范围为[-32768, 32767]
```

然后设置 Callback，定义一个 AudioStreamDataCallback 的子类，重写 onAudioReady 方法来实现自己填充数据的逻辑，但这个方法不可以太耗时，否则会出现卡顿

```c++
class MyCallback : public oboe::AudioStreamDataCallback {
public:
    oboe::DataCallbackResult
    onAudioReady(oboe::AudioStream *audioStream, void *audioData, int32_t numFrames) {
        auto *outputData = static_cast<float *>(audioData);
        const float amplitude = 0.2f;
        for (int i = 0; i < numFrames; ++i){
            outputData[i] = ((float)drand48() - 0.5f) * 2 * amplitude;
        }
        return oboe::DataCallbackResult::Continue;
    }
};
 
MyCallback myCallback;
builder.setDataCallback(&myCallback);
```

最终调用 openStream 来打开 Stream，根据返回值来判断是否创建成功，如果创建失败了，可以使用 convertToText 来查看错误码。

```c++
oboe::Result result = builder.openStream(mStream);
if (result != oboe::Result::OK) {
  LOGE("Failed to create stream. Error: %s", convertToText(result));
}
```

播放音频

调用了 requestStart 方法之后，就要在回调函数中填充数据

```c++
mStream->requestStart();

//关闭 AudioStream
mStream->close();
```















