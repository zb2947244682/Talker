# Android C++ SDK 语音功能开发指南

本文档为Android C++ SDK语音功能提供详细指南，涵盖语音识别（ASR）和语音合成（TTS），包括接口、参数配置及调用流程。

## 1. SDK 概述

- **支持版本**: Android 4.0+ (API Level 14)
- **支持架构**: `armeabi-v7a`, `arm64-v8a`, `x86`, `x86_64`
- **功能列表**:
  - 一句话识别
  - 实时语音识别
  - 语音合成（短文本、长文本、流式）
  - 录音文件识别极速版
  - 听悟实时推流
  - *不支持*: 离线语音合成、唤醒及命令词

## 2. 前提条件与安装

### 前提条件
- 已阅读接口说明文档
- 已获取项目`Appkey`和`Access Token`

### 安装步骤
1. 从Android SDK页面下载SDK，包含头文件和链接库文件
2. 解压ZIP包：
   - `android_libs`: 获取动态库
   - `android_include`: 获取头文件
3. 在初始化代码中替换阿里云账号信息、`Appkey`和`Token`

## 3. 核心接口

### 3.1 SDK 初始化
`nui_initialize`用于初始化SDK，单例模式，需先释放资源再重新初始化，避免在UI线程调用。

```cpp
NuiResultCode nui_initialize(const char *parameters,
                             const NuiSdkListener listener,
                             const NuiAsyncCallback async_listener = nullptr,
                             NuiSdkLogLevel level = LOG_LEVEL_VERBOSE,
                             bool save_log = false);
```

- **parameters**: JSON格式，包含：
  - `workspace` (String, 必选): 工作目录路径，需读写权限
  - `app_key` (String, 必选): 项目Appkey
  - `token` (String, 必选): 有效Token
  - `device_id` (String, 必选): 唯一用户ID
  - `debug_path` (String, 可选): 日志存储目录
  - `log_level` (String, 可选): 日志级别
  - `url` (String, 可选): 服务端URL
  - `ssl_enable` (String, 可选): 是否开启SSL
  - `mode_type` (String, 必选): 识别模式（ASR设为`"MODE_P2T"`，TTS设为`"2"`）
  - `tts_version` (String, 必选): 语音合成版本（`"0"`短文本，`"1"`长文本）
  - `custom_params` (String, 可选): 自定义JSON参数
- **listener**: 事件监听回调
- **async_listener**: 异步回调，`nullptr`为同步
- **level**: 日志级别，值越小打印越多
- **save_log**: 是否保存日志（存储于`debug_path`）
- **return**: 错误码

### 3.2 SDK 监听器 (NuiSdkListener)
定义回调函数：

| 名称                       | 类型                       | 说明                     |
|----------------------------|----------------------------|--------------------------|
| `event_callback`           | `FuncDialogListenerOnEvent` | NUI事件回调             |
| `user_data_callback`       | `FuncDialogUserProvideData` | 麦克风数据请求回调      |
| `audio_state_changed_callback` | `FuncDialogAudioStateChange` | 麦克风状态回调         |
| `audio_extra_event_callback` | `FuncDialogAudioExtraEvent` | 特殊事件回调（暂不使用） |
| `user_data`                | `void *`                   | 用户数据                |

### 3.3 参数设置与获取
- **`nui_set_params`**: 设置JSON参数
```cpp
NuiResultCode nui_set_params(const char *params,
                             const NuiAsyncCallback listener = nullptr);
```

- **`nui_get_param`**: 获取参数值
```cpp
const char* nui_get_param(const char* param,
                          const NuiAsyncCallback listener = nullptr);
```

### 3.4 资源释放
`nui_release`释放SDK资源
```cpp
NuiResultCode nui_release(const NuiAsyncCallback async_listener = nullptr);
```

### 3.5 获取版本
`nui_get_version`获取SDK版本
```cpp
const char* nui_get_version(const char* module = nullptr);
```

## 4. 语音识别 (ASR)

支持一句话识别和实时语音识别。

### 4.1 监听器回调
- **`FuncDialogAudioStateChange`**: 录音状态回调
```cpp
typedef void (*FuncDialogAudioStateChange)(void *user_data, NuiAudioState state);
```
  - `state`: `STATE_OPEN`（打开录音），`STATE_CLOSE`/`STATE_PAUSE`（关闭/暂停）

- **`FuncDialogUserProvideData`**: 提供录音数据
```cpp
typedef int (*FuncDialogUserProvideData)(void *user_data, char *buffer, int len);
```
  - `buffer`: 填充语音数据
  - `len`: 需填充字节数
  - `return`: 实际填充字节数

- **`FuncDialogListenerOnEvent`**: 事件回调
```cpp
typedef void (*FuncDialogListenerOnEvent)(void *user_data,
                                         NuiCallbackEvent event,
                                         long dialog,
                                         const char *wuw,
                                         const char *asr_result,
                                         bool finish,
                                         int code,
                                         const char *all_response);
```

**ASR事件**:
- `EVENT_VAD_START`: 人声起点
- `EVENT_VAD_END`: 人声尾点
- `EVENT_ASR_PARTIAL_RESULT`: 中间结果
- `EVENT_ASR_RESULT`: 最终结果
- `EVENT_ASR_ERROR`: 错误
- `EVENT_MIC_ERROR`: 录音错误
- `EVENT_SENTENCE_START`: 句子开始（实时识别）
- `EVENT_SENTENCE_END`: 句子结束（实时识别）
- `EVENT_TRANSCRIBER_COMPLETE`: 识别停止

### 4.2 开始识别
`nui_dialog_start`开始语音识别
```cpp
NuiResultCode nui_dialog_start(NuiVadMode vad_mode,
                               const char *dialog_params,
                               const NuiAsyncCallback listener = nullptr);
```
- `vad_mode`: 识别模式，推荐`MODE_P2T`

### 4.3 结束识别
`nui_dialog_cancel`结束识别
```cpp
NuiResultCode nui_dialog_cancel(bool force,
                                const NuiAsyncCallback listener = nullptr);
```
- `force`: `false`等待完整结果，`true`强制结束

### 4.4 调用流程
1. 初始化SDK和录音实例
2. 配置参数
3. 调用`nui_dialog_start`开始识别
4. 根据`audio_state_changed_callback`打开录音
5. 在`user_data_callback`提供录音数据
6. 通过`EVENT_ASR_PARTIAL_RESULT`获取中间结果
7. 实时识别：`EVENT_SENTENCE_START`/`EVENT_SENTENCE_END`获取句子结果
8. 调用`nui_dialog_cancel`结束识别，获取`EVENT_ASR_RESULT`
9. 调用`nui_release`释放资源

## 5. 语音合成 (TTS)

支持短文本、长文本和流式语音合成。

### 5.1 TTS 初始化
`nui_tts_initialize`初始化TTS SDK，单例模式，避免UI线程调用。
```cpp
int nui_tts_initialize(const char *parameters,
                       const NuiTtsSdkListener listener,
                       const NuiAsyncCallback async_listener = nullptr,
                       NuiSdkLogLevel level = LOG_LEVEL_VERBOSE,
                       bool save_log = false);
```
- **parameters**: JSON格式，参考3.1

### 5.2 TTS 监听器 (NuiTtsSdkListener)
| 名称                     | 类型                      | 说明           |
|--------------------------|---------------------------|----------------|
| `tts_event_callback`     | `FuncNuiTtsListenerOnEvent` | SDK事件回调   |
| `tts_user_data_callback` | `FuncNuiTtsUserProvideData` | 合成数据回调  |

### 5.3 TTS 回调
- **`FuncNuiTtsListenerOnEvent`**: 事件回调
```cpp
typedef void (*FuncNuiTtsListenerOnEvent)(void *user_data,
                                         NuiSdkTtsEvent event,
                                         char *taskid,
                                         int code);
```

**TTS事件**:
- `TTS_EVENT_START`: 合成开始
- `TTS_EVENT_END`: 合成结束
- `TTS_EVENT_CANCEL`: 取消合成
- `TTS_EVENT_PAUSE`: 暂停合成
- `TTS_EVENT_RESUME`: 恢复合成
- `TTS_EVENT_ERROR`: 错误

- **`FuncNuiTtsUserProvideData`**: 合成数据回调
```cpp
typedef void (*FuncNuiTtsUserProvideData)(void *user_data,
                                         char *info,
                                         int info_len,
                                         char *buffer,
                                         int len,
                                         char *taskid);
```
  - `info`: 时间戳JSON（启用`enable_subtitle`）
  - `buffer`: 音频数据
  - `len`: 音频数据字节数
  - `taskid`: 任务ID

### 5.4 参数设置与获取
- **`nui_tts_set_param`**: 设置TTS参数
```cpp
int nui_tts_set_param(const char *param,
                      const char *value,
                      const NuiAsyncCallback listener = nullptr);
```

- **`nui_tts_get_param`**: 获取TTS参数
```cpp
const char* nui_tts_get_param(const char *param,
                              const NuiAsyncCallback listener = nullptr);
```

### 5.5 开始合成
`nui_tts_play`开始合成任务
```cpp
int nui_tts_play(const char *priority,
                 const char *taskid,
                 const char *text,
                 const NuiAsyncCallback listener = nullptr);
```
- `priority`: 任务优先级，推荐`"1"`
- `taskid`: 任务ID，可为空（SDK自动生成）
- `text`: 合成文本

### 5.6 暂停/取消合成
- **`nui_tts_pause`**: 暂停合成
```cpp
int nui_tts_pause(const NuiAsyncCallback listener = nullptr);
```

- **`nui_tts_cancel`**: 取消合成
```cpp
int nui_tts_cancel(const char *taskid,
                   const NuiAsyncCallback listener = nullptr);
```