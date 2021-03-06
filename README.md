# Android-Opensl
Android Opensl 使用指南

## 1.修改cmake配置
```cmake
target_link_libraries( # Specifies the target library.
        native-lib
        OpenSLES
        ${log-lib})
```

## 2.创建 OpenSLES 引擎
```c++
SLEngineItf CreateSl() {

    SLresult result;
    SLEngineItf engineItf;

    result = slCreateEngine(&engineSL, 0, 0, 0, 0, 0);
    if (result != SL_RESULT_SUCCESS) return nullptr;

    result = (*engineSL)->Realize(engineSL, SL_BOOLEAN_FALSE);
    if (result != SL_RESULT_SUCCESS) return nullptr;

    result = (*engineSL)->GetInterface(engineSL, SL_IID_ENGINE, &engineItf);
    if (result != SL_RESULT_SUCCESS) return nullptr;

    return engineItf;
}

    //创建 OpenSLES 引擎
    SLEngineItf engineItf = CreateSl();
    if (!engineItf) {
        LOGE("OpenSLES 创建失败");
        return env->NewStringUTF(hello.c_str());
    }
    LOGE("OpenSLES 创建成功");
```

## 3.创建混音器
```c++
    //创建混音器
    SLObjectItf pMix = nullptr;
    SLresult result;
    result = (*engineItf)->CreateOutputMix(engineItf, &pMix, 0, 0, 0);
    if (result != SL_RESULT_SUCCESS) LOGE("CreateOutputMix 失败");
    result = (*pMix)->Realize(pMix, SL_BOOLEAN_FALSE);
    if (result != SL_RESULT_SUCCESS) LOGE("Realize pMix 失败");
    SLDataLocator_OutputMix outputMix = {SL_DATALOCATOR_OUTPUTMIX, pMix};
    SLDataSink audioSink = {&outputMix, 0};
```

## 4.配置音频信息
```c++
    //缓冲队列
    SLDataLocator_AndroidSimpleBufferQueue androidSimpleBufferQueue = {
            SL_DATALOCATOR_ANDROIDSIMPLEBUFFERQUEUE, 10};
    //音频格式
    SLDataFormat_PCM pcm = {
            SL_DATAFORMAT_PCM,
            2,//声道数
            SL_SAMPLINGRATE_44_1,
            SL_PCMSAMPLEFORMAT_FIXED_16,
            SL_PCMSAMPLEFORMAT_FIXED_16,
            SL_SPEAKER_FRONT_LEFT | SL_SPEAKER_FRONT_RIGHT,
            SL_BYTEORDER_LITTLEENDIAN //字节序，小端
    };
    SLDataSource dataSource = {&androidSimpleBufferQueue, &pcm};
```

## 5.创建播放器
```c++
    SLObjectItf player;
    SLPlayItf iPlayer;
    SLAndroidSimpleBufferQueueItf iSimpleBufferQueue;
    const SLInterfaceID ids[] = {SL_IID_BUFFERQUEUE};//哪些接口
    const SLboolean req[] = {SL_BOOLEAN_TRUE};//是否开放接口
    result = (*engineItf)->CreateAudioPlayer(engineItf, &player, &dataSource, &audioSink,
                                             sizeof(ids) / sizeof(SLInterfaceID),
                                             ids,
                                             req);
    if (result != SL_RESULT_SUCCESS) LOGE("CreateAudioPlayer 失败");
    result = (*player)->Realize(player, SL_BOOLEAN_FALSE);//SL_BOOLEAN_FALSE 阻塞式等待
    if (result != SL_RESULT_SUCCESS) LOGE("player Realize 失败");

    //获取 player 接口
    result = (*player)->GetInterface(player, SL_IID_PLAY, &iPlayer);//通过iPlayer接口获得具体的实现
    if (result != SL_RESULT_SUCCESS) LOGE("iPlayer GetInterface 失败");

    //获取 player 队列
    result = (*player)->GetInterface(player, SL_IID_BUFFERQUEUE,
                                     &iSimpleBufferQueue);//通过iPlayer接口获得具体的实现
    if (result != SL_RESULT_SUCCESS) LOGE("iSimpleBufferQueue GetInterface 失败");

    //设置回调函数，播放队列空调用
    result = (*iSimpleBufferQueue)->RegisterCallback(iSimpleBufferQueue, PCMCall, 0);
    if (result != SL_RESULT_SUCCESS) LOGE("iSimpleBufferQueue RegisterCallback 失败");

    //设置播放状态
    result = (*iPlayer)->SetPlayState(iPlayer, SL_PLAYSTATE_PLAYING);
    if (result != SL_RESULT_SUCCESS) LOGE("iPlayer SetPlayState 失败");

    //启动队列回调
    result = (*iSimpleBufferQueue)->Enqueue(iSimpleBufferQueue, "", 1);
    if (result != SL_RESULT_SUCCESS) LOGE("iSimpleBufferQueue Enqueue 失败");
```
## 6.回调函数
```c++
extern void PCMCall(SLAndroidSimpleBufferQueueItf caller, void *pContext);
void PCMCall(SLAndroidSimpleBufferQueueItf caller, void *pContext) {
    LOGE("调用 PCMCall");
    static FILE *fp = nullptr;
    static char *buf = nullptr;
    if (!buf) {
        buf = new char[1024 * 1024];
    }

    if (!fp) {
        fp = fopen("/sdcard/test.pcm", "rb");
    }
    if (!fp) return;

    if (feof(fp) == 0) {
        unsigned int len = fread(buf, 1, 1024, fp);
        if (len > 0) {
            (*caller)->Enqueue(caller, buf, len);
        }
    }
}
```

## 7.其他
具体的代码请看native-lib.cpp

音频文件在file目录下

