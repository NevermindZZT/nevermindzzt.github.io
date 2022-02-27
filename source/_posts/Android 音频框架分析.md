---
title:      Android 音频框架分析
date:       2020-10-15 10:45:00
subtitle:   
author:     Letter
cover:    
catalog:    true
categories: Android
tags:       Android
---
## 初始化

![Audio 初始化流程](../../out/source/_posts/Android%20音频框架分析/初始化流程/Audio%20初始化流程.png)

系统开机启动时，加载`init.rc`，启动`mediaserver`，在Android7.0之后，启动`mediaserver`的脚本被分离在了`audioserver.rc`文件中

```rc
service media /system/bin/mediaserver
    class main
    user media
    group audio camera inet net_bt net_bt_admin net_bw_acct drmrpc mediadrm
    ioprio rt 4
```

`mediaserver`源码位置为`frameworks/av/media/mediaserver`，`mediaserver`目录中主要只有`main_mediaserver.cpp`一个文件，用于启动`AudioFlinger`, `AudioPolicyService`, `RadioService`等服务，其中与音频相关的主要是`AudioFlinger`和`AudioPolicyService`

```cpp
int main(int argc __unused, char** argv)
{
    ...
    if (doLog && (childPid = fork()) != 0) {
        /** 启动log线程 */
        ...
    } else {
        ...
        AudioFlinger::instantiate();
        MediaPlayerService::instantiate();
        ResourceManagerService::instantiate();
        CameraService::instantiate();
        AudioPolicyService::instantiate();
        SoundTriggerHwService::instantiate();
        RadioService::instantiate();
        registerExtensions();
        ProcessState::self()->startThreadPool();
        IPCThreadState::self()->joinThreadPool();
    }
}
```

在`AudioPolicyService`中，会在`onFristRef`方法中启动`AudioPolicyManager`

```cpp
void AudioPolicyService::onFirstRef()
{
    {
        ....
        // start tone playback thread
        mTonePlaybackThread = new AudioCommandThread(String8("ApmTone", this));
        // start audio commands thread
        mTonePlaybackThread = new AudioCommandThread(String8("ApmAudio", this));
        // start output activity command thread
        mTonePlaybackThread = new AudioCommandThread(String8("ApmOutput", this));
#ifdef USE_LEGACY_AUDIO_POLICY
        // 使用老版本的 audio policy 初始化方式
        ....
#else
        // 使用最新的 audio policy 初始化方式
        ALOGI("AudioPolicyService CSTOR in new mode");

        mAudioPolicyClient = new AudioPolicyClient(this);
        mAudioPolicyManager =createAudioPolicyManager(mAudioPolicyClient);    // 创建 AudioPolicyManager 对象
        ....
#endif
    }
    // load audio processing modules
    sp<AudioPolicyEffects> audioPolicyEffects = new AudioPolicyEffects();
    {
        ....
        mAudioPolicyEffects = audioPolicyEffects;
    }
}
```

`createAudioPolicyManager`函数位于`frameworks/av/services/audiopolicy/manager/AudioPolicyFactory.cpp`文件，这个文件只有这一个函数，直接调用了`AudioPolicyManager`的构造函数

```cpp
extern "C" AudioPolicyInterface* createAudioPolicyManager(
        AudioPolicyClientInterface *clientInterface)
{
    return new AudioPolicyManager(clientInterface);
}
```

然后在`AudioPolicyManager`的构造函数中，会对`audio_policy.conf`文件(Android 7.0之后可以配置成audio_policy_configuration.xml)进行解析，加载所有的`HwModule`，创建输入输出流和播放、录音线程

```cpp
AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface *clientInterface)
    :
#ifdef AUDIO_POLICY_TEST
    Thread(false),
#endif //AUDIO_POLICY_TEST
    mLimitRingtoneVolume(false), mLastVoiceVolume(-1.0f),
    mA2dpSuspended(false),
    mSpeakerDrcEnabled(false),
    mAudioPortGeneration(1),
    mBeaconMuteRefCount(0),
    mBeaconPlayingRefCount(0),
    mBeaconMuted(false)
{
    ...
    /** 加载audio_policy.conf文件 */
    if (ConfigParsingUtils::loadAudioPolicyConfig(AUDIO_POLICY_VENDOR_CONFIG_FILE,
                 mHwModules, mAvailableInputDevices, mAvailableOutputDevices,
                 mDefaultOutputDevice, mSpeakerDrcEnabled) != NO_ERROR) {
        if (ConfigParsingUtils::loadAudioPolicyConfig(AUDIO_POLICY_CONFIG_FILE,
                                  mHwModules, mAvailableInputDevices, mAvailableOutputDevices,
                                  mDefaultOutputDevice, mSpeakerDrcEnabled) != NO_ERROR) {
            ALOGE("could not load audio policy configuration file, setting defaults");
            defaultAudioPolicyConfig();
        }
    }
    // mAvailableOutputDevices and mAvailableInputDevices now contain all attached devices

    // must be done after reading the policy (since conditionned by Speaker Drc Enabling)
    /** 调节音量曲线 */
    mEngine->initializeVolumeCurves(mSpeakerDrcEnabled);

    // open all output streams needed to access attached devices
    audio_devices_t outputDeviceTypes = mAvailableOutputDevices.types();
    audio_devices_t inputDeviceTypes = mAvailableInputDevices.types() & ~AUDIO_DEVICE_BIT_IN;
    for (size_t i = 0; i < mHwModules.size(); i++) {
        mHwModules[i]->mHandle = mpClientInterface->loadHwModule(mHwModules[i]->mName);
        if (mHwModules[i]->mHandle == 0) {
            ALOGW("could not open HW module %s", mHwModules[i]->mName);
            continue;
        }
        // open all output streams needed to access attached devices
        // except for direct output streams that are only opened when they are actually
        // required by an app.
        // This also validates mAvailableOutputDevices list
        for (size_t j = 0; j < mHwModules[i]->mOutputProfiles.size(); j++)
        {
            const sp<IOProfile> outProfile = mHwModules[i]->mOutputProfiles[j];

            ...
            /** 获取采样率、通道数、数据格式等各音频参数 */
            outputDesc->mDevice = profileType;
            audio_config_t config = AUDIO_CONFIG_INITIALIZER;
            config.sample_rate = outputDesc->mSamplingRate;
            config.channel_mask = outputDesc->mChannelMask;
            config.format = outputDesc->mFormat;
            audio_io_handle_t output = AUDIO_IO_HANDLE_NONE;

            /** 打开 outputStream 创建 playbackThread 线程 */
            status_t status = mpClientInterface->openOutput(outProfile->getModuleHandle(),
                                                            &output,
                                                            &config,
                                                            &outputDesc->mDevice,
                                                            String8(""),
                                                            &outputDesc->mLatency,
                                                            outputDesc->mFlags);

            ...
        }
        // open input streams needed to access attached devices to validate
        // mAvailableInputDevices list
        for (size_t j = 0; j < mHwModules[i]->mInputProfiles.size(); j++)
        {
            const sp<IOProfile> inProfile = mHwModules[i]->mInputProfiles[j];

            ...
            /** 获取采样率、通道数、数据格式等各音频参数 */
            audio_config_t config = AUDIO_CONFIG_INITIALIZER;
            config.sample_rate = inputDesc->mSamplingRate;
            config.channel_mask = inputDesc->mChannelMask;
            config.format = inputDesc->mFormat;
            audio_io_handle_t input = AUDIO_IO_HANDLE_NONE;

            /** 打开 inputStream 创建 recordThread  线程 */
            status_t status = mpClientInterface->openInput(inProfile->getModuleHandle(),
                                                           &input,
                                                           &config,
                                                           &inputDesc->mDevice,
                                                           address,
                                                           AUDIO_SOURCE_MIC,
                                                           AUDIO_INPUT_FLAG_NONE);

            ...
        }
    }
    ...
    /** 更新系统缓存的音频输出设备信息 */
    updateDevicesAndOutputs();
    ...
}
```

## 配置文件加载

`audio_policy.conf`文件用于配置系统的音频设备，在`AudioPolicyManager`的构造方法中加载解析，通过`ConfigParsingUtils`的`loadAudioPolicyConfig`函数，解析这个文件，获取所有配置的输入输出设备类型以及各项配置，然后对所有获取到的`HwModule`进行遍历，通过`AudioFlinger`加载对应的so，生成音频设备

```cpp
audio_module_handle_t AudioFlinger::loadHwModule_l(const char *name)
{
    ...
    audio_hw_device_t *dev;

    /** 加载so */
    int rc = load_audio_interface(name, &dev);

    ...

    /** 添加音频设备 */
    audio_module_handle_t handle = nextUniqueId();
    mAudioHwDevs.add(handle, new AudioHwDevice(handle, name, dev, flags));

    ...
    return handle;
}
```

在`load_audio_interface`函数中，会从`system/lib/hw`中找到对应于`name`的库，一般为`audio.[name].default.so`，这个库由音频设备商提供，系统默认的音频设备库由`hardware/mstar/audio`编译生成，对应`primary`的设备

`audio_hw_device_t`为音频设备的实现，结构体在`hardware/libhardware/include/hardware/audio.h`中定义，每一个音频设备驱动都需要实现这个结构体定义的各个函数，所有操作音频设备都是通过这个结构体生成的对象实现的

## 录音调用流程

![Audio 录音调用流程](../../out/source/_posts/Android%20音频框架分析/录音调用流程/Audio%20录音调用流程.png)

一般情况下，录音由Android App通过`AudioRecord`发起，`AudioRecord`通过jni调用Android框架libmedia中的`AudioRecord`，`AudioRecord`的构造函数直接调用`set`函数，在`set`函数中，会首先进行一些参数的设置，然后运行`recordThread`，打开录音设备(`IAudioRecord`)

```cpp
status_t AudioRecord::set(
        audio_source_t inputSource,
        uint32_t sampleRate,
        audio_format_t format,
        audio_channel_mask_t channelMask,
        size_t frameCount,
        callback_t cbf,
        void* user,
        uint32_t notificationFrames,
        bool threadCanCallJava,
        int sessionId,
        transfer_type transferType,
        audio_input_flags_t flags,
        int uid,
        pid_t pid,
        const audio_attributes_t* pAttributes)
{
    /** 设置参数 */
    ...

    /** 打开 recordThread */
    if (cbf != NULL) {
        mAudioRecordThread = new AudioRecordThread(*this, threadCanCallJava);
        mAudioRecordThread->run("AudioRecord", ANDROID_PRIORITY_AUDIO);
        // thread begins in paused state, and will not reference us until start()
    }

    // create the IAudioRecord
    status_t status = openRecord_l(0 /*epoch*/, mOpPackageName);

    ...

    return NO_ERROR;
}
```

`set`函数调用了`openRecord_l`函数，在`openRecord_l`函数中，首先获取了`AudioFlinger`的实例，然后通过构造`AudioRecord`时传进来的参数，获取对应的`audio_io_handle_t`句柄

```cpp
    audio_io_handle_t input;
    status_t status = AudioSystem::getInputForAttr(&mAttributes, &input,
                                        (audio_session_t)mSessionId,
                                        IPCThreadState::self()->getCallingUid(),
                                        mSampleRate, mFormat, mChannelMask,
                                        mFlags, mSelectedDeviceId);
```

然后，通过`AudioFlinger`打开录音

```cpp
    sp<IAudioRecord> record = audioFlinger->openRecord(input,
                                                       mSampleRate,
                                                       mFormat,
                                                       mChannelMask,
                                                       opPackageName,
                                                       &temp,
                                                       &trackFlags,
                                                       tid,
                                                       mClientUid,
                                                       &mSessionId,
                                                       &notificationFrames,
                                                       iMem,
                                                       bufferMem,
                                                       &status);
```

## 录音设备选择

录音调用流程中，在`AudioRecord`的构造函数中，通过`set`函数调用了`AudioSystem::getInputForAttr`以获得匹配的录音设备，我们着重关注第一个参数`mAttributes`

在Android App中，我们通过实例化一个`AudioRecord(java)`来进行录音，通常调用的构造方法为

```java
    public AudioRecord(int audioSource, int sampleRateInHz, int channelConfig, int audioFormat,
            int bufferSizeInBytes)
    throws IllegalArgumentException {
        this((new AudioAttributes.Builder())
                    .setInternalCapturePreset(audioSource)
                    .build(),
                (new AudioFormat.Builder())
                    .setChannelMask(getChannelMaskFromLegacyConfig(channelConfig,
                                        true/*allow legacy configurations*/))
                    .setEncoding(audioFormat)
                    .setSampleRate(sampleRateInHz)
                    .build(),
                bufferSizeInBytes,
                AudioManager.AUDIO_SESSION_ID_GENERATE);
    }
```

可以看到，这里需要传递`audioSource`，`sampleRateInHz`等参数，并将这些参数构造成了一个`AudioAttributes`的对象，其中`audioSource`就是选择的音频源，在Android App端，可以选择的源如下(`frameworks/base/media/java/android/media/MediaRecorder.java`)

```java
    public final class AudioSource {

        private AudioSource() {}

        /** @hide */
        public final static int AUDIO_SOURCE_INVALID = -1;

      /* Do not change these values without updating their counterparts
       * in system/media/audio/include/system/audio.h!
       */

        /** Default audio source **/
        public static final int DEFAULT = 0;

        /** Microphone audio source */
        public static final int MIC = 1;

        /** Voice call uplink (Tx) audio source */
        public static final int VOICE_UPLINK = 2;

        /** Voice call downlink (Rx) audio source */
        public static final int VOICE_DOWNLINK = 3;

        /** Voice call uplink + downlink audio source */
        public static final int VOICE_CALL = 4;

        /** Microphone audio source with same orientation as camera if available, the main
         *  device microphone otherwise */
        public static final int CAMCORDER = 5;

        /** Microphone audio source tuned for voice recognition if available, behaves like
         *  {@link #DEFAULT} otherwise. */
        public static final int VOICE_RECOGNITION = 6;

        /** Microphone audio source tuned for voice communications such as VoIP. It
         *  will for instance take advantage of echo cancellation or automatic gain control
         *  if available. It otherwise behaves like {@link #DEFAULT} if no voice processing
         *  is applied.
         */
        public static final int VOICE_COMMUNICATION = 7;

        /**
         * Audio source for a submix of audio streams to be presented remotely.
         * <p>
         * An application can use this audio source to capture a mix of audio streams
         * that should be transmitted to a remote receiver such as a Wifi display.
         * While recording is active, these audio streams are redirected to the remote
         * submix instead of being played on the device speaker or headset.
         * </p><p>
         * Certain streams are excluded from the remote submix, including
         * {@link AudioManager#STREAM_RING}, {@link AudioManager#STREAM_ALARM},
         * and {@link AudioManager#STREAM_NOTIFICATION}.  These streams will continue
         * to be presented locally as usual.
         * </p><p>
         * Capturing the remote submix audio requires the
         * {@link android.Manifest.permission#CAPTURE_AUDIO_OUTPUT} permission.
         * This permission is reserved for use by system components and is not available to
         * third-party applications.
         * </p>
         */
        public static final int REMOTE_SUBMIX = 8;

        // MStar Android Patch Begin
        /**
         * @hide
         * BT PCM data send from remote devices which needs to be processed as PCM input
         */
        public static final int BT_MIC = 9;
        // MStar Android Patch End

        /**
         * Audio source for capturing broadcast radio tuner output.
         * @hide
         */
        @SystemApi
        public static final int RADIO_TUNER = 1998;

        /**
         * Audio source for preemptible, low-priority software hotword detection
         * It presents the same gain and pre processing tuning as {@link #VOICE_RECOGNITION}.
         * <p>
         * An application should use this audio source when it wishes to do
         * always-on software hotword detection, while gracefully giving in to any other application
         * that might want to read from the microphone.
         * </p>
         * This is a hidden audio source.
         * @hide
         */
        @SystemApi
        public static final int HOTWORD = 1999;
    }
```

传递进来的参数jni，调用到`AudioRecord(cpp)`的`set`函数

```cpp
status_t AudioRecord::set(
        audio_source_t inputSource,
        uint32_t sampleRate,
        audio_format_t format,
        audio_channel_mask_t channelMask,
        size_t frameCount,
        callback_t cbf,
        void* user,
        uint32_t notificationFrames,
        bool threadCanCallJava,
        int sessionId,
        transfer_type transferType,
        audio_input_flags_t flags,
        int uid,
        pid_t pid,
        const audio_attributes_t* pAttributes)
```

在这里，`inputSource`即表示音频源，`audio_source_t`在`audio.h`中定义(`system/media/audio/include/system/audio.h`)且与`AudioSource`中的定义一一对应

```cpp
typedef enum {
    AUDIO_SOURCE_DEFAULT             = 0,
    AUDIO_SOURCE_MIC                 = 1,
    AUDIO_SOURCE_VOICE_UPLINK        = 2,
    AUDIO_SOURCE_VOICE_DOWNLINK      = 3,
    AUDIO_SOURCE_VOICE_CALL          = 4,
    AUDIO_SOURCE_CAMCORDER           = 5,
    AUDIO_SOURCE_VOICE_RECOGNITION   = 6,
    AUDIO_SOURCE_VOICE_COMMUNICATION = 7,
    AUDIO_SOURCE_REMOTE_SUBMIX       = 8, /* Source for the mix to be presented remotely.      */
                                          /* An example of remote presentation is Wifi Display */
                                          /*  where a dongle attached to a TV can be used to   */
                                          /*  play the mix captured by this audio source.      */
    // MStar Android Patch Begin
    AUDIO_SOURCE_BLUETOOTH_MIC       = 9,
    AUDIO_SOURCE_CAPTURE_DEVICE0     = 10,
    AUDIO_SOURCE_CAPTURE_DEVICE1     = 11,
    AUDIO_SOURCE_A2DP                = 12, /* restricted to internal audioflinger routing */
    // MStar Android Patch End
    AUDIO_SOURCE_CNT,
    AUDIO_SOURCE_MAX                 = AUDIO_SOURCE_CNT - 1,
    AUDIO_SOURCE_FM_TUNER            = 1998,
    AUDIO_SOURCE_HOTWORD             = 1999, /* A low-priority, preemptible audio source for
                                                for background software hotword detection.
                                                Same tuning as AUDIO_SOURCE_VOICE_RECOGNITION.
                                                Used only internally to the framework. Not exposed
                                                at the audio HAL. */
} audio_source_t;
```

我们再回头看`getInputForAttr`函数(`frameworks/av/media/libmedia/AudioSystem.cpp`)

```cpp
status_t AudioSystem::getInputForAttr(const audio_attributes_t *attr,
                                audio_io_handle_t *input,
                                audio_session_t session,
                                uid_t uid,
                                uint32_t samplingRate,
                                audio_format_t format,
                                audio_channel_mask_t channelMask,
                                audio_input_flags_t flags,
                                audio_port_handle_t selectedDeviceId)
{
    const sp<IAudioPolicyService>& aps = AudioSystem::get_audio_policy_service();
    if (aps == 0) return NO_INIT;
    return aps->getInputForAttr(
            attr, input, session, uid, samplingRate, format, channelMask, flags, selectedDeviceId);
}
```

可以看到，这里直接调用的`AudioPolicyService`的`getInputForAttr`函数

```cpp
status_t AudioPolicyManager::getInputForAttr(const audio_attributes_t *attr,
                                             audio_io_handle_t *input,
                                             audio_session_t session,
                                             uid_t uid,
                                             uint32_t samplingRate,
                                             audio_format_t format,
                                             audio_channel_mask_t channelMask,
                                             audio_input_flags_t flags,
                                             audio_port_handle_t selectedDeviceId,
                                             input_type_t *inputType)
{
    *input = AUDIO_IO_HANDLE_NONE;
    *inputType = API_INPUT_INVALID;
    audio_devices_t device;

    ...

    if (inputSource == AUDIO_SOURCE_REMOTE_SUBMIX &&
            strncmp(attr->tags, "addr=", strlen("addr=")) == 0) {
        status_t ret = mPolicyMixes.getInputMixForAttr(*attr, &policyMix);
        if (ret != NO_ERROR) {
            return ret;
        }
        *inputType = API_INPUT_MIX_EXT_POLICY_REROUTE;
        device = AUDIO_DEVICE_IN_REMOTE_SUBMIX;
        address = String8(attr->tags + strlen("addr="));
    } else {
        device = getDeviceAndMixForInputSource(inputSource, &policyMix);
        ...
    }

    ...

    return NO_ERROR;
}
```

这里先判断`inputSource`是否为`AUDIO_SOURCE_REMOTE_SUBMIX`，如果时其他源，则调用`getDeviceAndMixForInputSource`函数

这里我们注意一下`audio_devices_t`这个数据类型，在`audio.h`中定义，这个数据类型就表示一个具体类型的音频设备，我们再`audio_polic.conf`中定义音频设备时，`device`字段的取值就需要是这个数据类型中定义的值

再来看`getDeviceAndMixForInputSource`函数

```cpp
audio_devices_t AudioPolicyManager::getDeviceAndMixForInputSource(audio_source_t inputSource,
                                                                  AudioMix **policyMix)
{
    audio_devices_t availableDeviceTypes = mAvailableInputDevices.types() & ~AUDIO_DEVICE_BIT_IN;
    audio_devices_t selectedDeviceFromMix =
           mPolicyMixes.getDeviceAndMixForInputSource(inputSource, availableDeviceTypes, policyMix);

    if (selectedDeviceFromMix != AUDIO_DEVICE_NONE) {
        return selectedDeviceFromMix;
    }
    return getDeviceForInputSource(inputSource);
}
```

这里首先调用了`getDeviceAndMixForInputSource`函数，这个应该是混音通道，我们先不看，一般情况，我们调用录音，会走到`getDeviceForInputSource`

```cpp
audio_devices_t AudioPolicyManager::getDeviceForInputSource(audio_source_t inputSource)
{
    for (size_t routeIndex = 0; routeIndex < mInputRoutes.size(); routeIndex++) {
         sp<SessionRoute> route = mInputRoutes.valueAt(routeIndex);
         if (inputSource == route->mSource && route->isActive()) {
             return route->mDeviceDescriptor->type();
         }
     }
     return mEngine->getDeviceForInputSource(inputSource);

}
```

`getDeviceForInputSource`函数直接调用了`Engine`的`getDeviceForInputSource`函数(`frameworks/av/services/audiopolicy/enginedefault/src/Engine.cpp`)

```cpp
audio_devices_t Engine::getDeviceForInputSource(audio_source_t inputSource) const
{
    const DeviceVector &availableOutputDevices = mApmObserver->getAvailableOutputDevices();
    const DeviceVector &availableInputDevices = mApmObserver->getAvailableInputDevices();
    const SwAudioOutputCollection &outputs = mApmObserver->getOutputs();
    audio_devices_t availableDeviceTypes = availableInputDevices.types() & ~AUDIO_DEVICE_BIT_IN;

    uint32_t device = AUDIO_DEVICE_NONE;
    // MStar Android Patch Begin
    char value[PROPERTY_VALUE_MAX] = {0};
    // MStar Android Patch End
    switch (inputSource) {
    case AUDIO_SOURCE_VOICE_UPLINK:
      if (availableDeviceTypes & AUDIO_DEVICE_IN_VOICE_CALL) {
          device = AUDIO_DEVICE_IN_VOICE_CALL;
          break;
      }
      break;

    case AUDIO_SOURCE_DEFAULT:
    case AUDIO_SOURCE_MIC:
    if (availableDeviceTypes & AUDIO_DEVICE_IN_BLUETOOTH_A2DP) {
        device = AUDIO_DEVICE_IN_BLUETOOTH_A2DP;
    } else if ((mForceUse[AUDIO_POLICY_FORCE_FOR_RECORD] == AUDIO_POLICY_FORCE_BT_SCO) &&
        (availableDeviceTypes & AUDIO_DEVICE_IN_BLUETOOTH_SCO_HEADSET)) {
        device = AUDIO_DEVICE_IN_BLUETOOTH_SCO_HEADSET;
    } else if (availableDeviceTypes & AUDIO_DEVICE_IN_WIRED_HEADSET) {
        device = AUDIO_DEVICE_IN_WIRED_HEADSET;
    } else if (availableDeviceTypes & AUDIO_DEVICE_IN_USB_DEVICE) {
        device = AUDIO_DEVICE_IN_USB_DEVICE;
    } else if (availableDeviceTypes & AUDIO_DEVICE_IN_BUILTIN_MIC) {
        device = AUDIO_DEVICE_IN_BUILTIN_MIC;
    }
    break;

    case AUDIO_SOURCE_VOICE_COMMUNICATION:
        // Allow only use of devices on primary input if in call and HAL does not support routing
        // to voice call path.
        if ((getPhoneState() == AUDIO_MODE_IN_CALL) &&
                (availableOutputDevices.types() & AUDIO_DEVICE_OUT_TELEPHONY_TX) == 0) {
            sp<AudioOutputDescriptor> primaryOutput = outputs.getPrimaryOutput();
            availableDeviceTypes =
                    availableInputDevices.getDevicesFromHwModule(primaryOutput->getModuleHandle())
                    & ~AUDIO_DEVICE_BIT_IN;
        }

        switch (mForceUse[AUDIO_POLICY_FORCE_FOR_COMMUNICATION]) {
        case AUDIO_POLICY_FORCE_BT_SCO:
            // if SCO device is requested but no SCO device is available, fall back to default case
            if (availableDeviceTypes & AUDIO_DEVICE_IN_BLUETOOTH_SCO_HEADSET) {
                device = AUDIO_DEVICE_IN_BLUETOOTH_SCO_HEADSET;
                break;
            }
            // FALL THROUGH

        default:    // FORCE_NONE
            if (availableDeviceTypes & AUDIO_DEVICE_IN_WIRED_HEADSET) {
                device = AUDIO_DEVICE_IN_WIRED_HEADSET;
            } else if (availableDeviceTypes & AUDIO_DEVICE_IN_USB_DEVICE) {
                device = AUDIO_DEVICE_IN_USB_DEVICE;
            } else if (availableDeviceTypes & AUDIO_DEVICE_IN_BUILTIN_MIC) {
                device = AUDIO_DEVICE_IN_BUILTIN_MIC;
            }
            break;

        case AUDIO_POLICY_FORCE_SPEAKER:
            if (availableDeviceTypes & AUDIO_DEVICE_IN_BACK_MIC) {
                device = AUDIO_DEVICE_IN_BACK_MIC;
            } else if (availableDeviceTypes & AUDIO_DEVICE_IN_BUILTIN_MIC) {
                device = AUDIO_DEVICE_IN_BUILTIN_MIC;
            }
            break;
        }
        break;

    // MStar Android Patch Begin
    case AUDIO_SOURCE_BLUETOOTH_MIC:
        if (mForceUse[AUDIO_POLICY_FORCE_FOR_RECORD] == AUDIO_POLICY_FORCE_BT_MIC &&
                availableDeviceTypes & AUDIO_DEVICE_IN_BLUETOOTH_MIC) {
                device = AUDIO_DEVICE_IN_BLUETOOTH_MIC;
        } else if (availableDeviceTypes & AUDIO_DEVICE_IN_BACK_MIC) {
            device = AUDIO_DEVICE_IN_BACK_MIC;
        }
        break;
    // MStar Android Patch End
    case AUDIO_SOURCE_VOICE_RECOGNITION:
    case AUDIO_SOURCE_HOTWORD:
        if (mForceUse[AUDIO_POLICY_FORCE_FOR_RECORD] == AUDIO_POLICY_FORCE_BT_SCO &&
                availableDeviceTypes & AUDIO_DEVICE_IN_BLUETOOTH_SCO_HEADSET) {
            device = AUDIO_DEVICE_IN_BLUETOOTH_SCO_HEADSET;
        } else if (availableDeviceTypes & AUDIO_DEVICE_IN_WIRED_HEADSET) {
            device = AUDIO_DEVICE_IN_WIRED_HEADSET;
        } else if (availableDeviceTypes & AUDIO_DEVICE_IN_USB_DEVICE) {
            device = AUDIO_DEVICE_IN_USB_DEVICE;
        } else if (availableDeviceTypes & AUDIO_DEVICE_IN_BUILTIN_MIC) {
            device = AUDIO_DEVICE_IN_BUILTIN_MIC;
        }

        // MStar Android Patch Begin
        if (property_get("mstar.bt.driver", value, NULL)) {
            if (!strcmp(value, "bcm")) {
                if (mForceUse[AUDIO_POLICY_FORCE_FOR_RECORD] == AUDIO_POLICY_FORCE_BT_MIC &&
                    (inputSource == AUDIO_SOURCE_VOICE_RECOGNITION ||
                    inputSource == AUDIO_SOURCE_MIC)) {
                    if (availableDeviceTypes & AUDIO_DEVICE_IN_BACK_MIC) {
                        device = AUDIO_DEVICE_IN_BACK_MIC;
                    } else {
                        device = AUDIO_DEVICE_IN_BLUETOOTH_MIC;
                    }
                    ALOGE("picked the Broadcom MIC");
                }
            }else if (!strcmp(value, "mtk") || !strcmp(value, "rtk")) {
                if ((inputSource == AUDIO_SOURCE_VOICE_RECOGNITION ||
                    inputSource == AUDIO_SOURCE_MIC)) {
                    if (availableDeviceTypes & AUDIO_DEVICE_IN_BACK_MIC) {
                        device = AUDIO_DEVICE_IN_BACK_MIC;
                    } else {
                        device = AUDIO_DEVICE_IN_BLUETOOTH_MIC;
                    }
                    ALOGE("picked the MtkRc MIC");
                }
            }else {
                //other bt to do ???
            }
        }
        ALOGE("%s device:%x ", __FUNCTION__, device);
        // MStar Android Patch End
        break;
    case AUDIO_SOURCE_CAMCORDER:
        if (availableDeviceTypes & AUDIO_DEVICE_IN_BACK_MIC) {
            device = AUDIO_DEVICE_IN_BACK_MIC;
        } else if (availableDeviceTypes & AUDIO_DEVICE_IN_BUILTIN_MIC) {
            device = AUDIO_DEVICE_IN_BUILTIN_MIC;
        }
        break;
    case AUDIO_SOURCE_VOICE_DOWNLINK:
    case AUDIO_SOURCE_VOICE_CALL:
        if (availableDeviceTypes & AUDIO_DEVICE_IN_VOICE_CALL) {
            device = AUDIO_DEVICE_IN_VOICE_CALL;
        }
        break;
    case AUDIO_SOURCE_REMOTE_SUBMIX:
        if (availableDeviceTypes & AUDIO_DEVICE_IN_REMOTE_SUBMIX) {
            device = AUDIO_DEVICE_IN_REMOTE_SUBMIX;
        }
        break;
     case AUDIO_SOURCE_FM_TUNER:
        //if (availableDeviceTypes & AUDIO_DEVICE_IN_FM_TUNER) {
        //    device = AUDIO_DEVICE_IN_FM_TUNER;
        //}
        device = AUDIO_DEVICE_IN_BLUETOOTH_A2DP;
        break;
    default:
        ALOGW("getDeviceForInputSource() invalid input source %d", inputSource);
        break;
    }
    ALOGE("getDeviceForInputSource()input source %d, device %08x", inputSource, device);
    return device;
}
```

这个函数会根据给进来的音频源参数，选择合适并且可用的音频设备，由此，就完成了从Android App层面音频源(`AudioSource`)到Audio Hal(`audio_policy.conf`)的调用
