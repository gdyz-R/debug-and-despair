# 科大讯飞 TTS API - Opus 音频帧前的长度字段未在文档说明的问题

**标签：** `#TTS` `#Opus` `#科大讯飞` `#音频处理` `#ESP32` `#帧解析` `#坑点记录`

---

## 问题描述

在接入科大讯飞 TTS（Text-to-Speech）API 时，选择返回 `opus-wb` 格式的音频流。官方文档中说明“每一帧为 20ms”，但并**没有提到每一帧前存在额外的长度字段**。

我直接将接收到的音频数据解码失败，怀疑是编码参数、缓冲区大小、播放器问题，甚至一度以为是音频设备损坏。

---

## 原因分析

### 起初的猜测：
- 是不是 Opus 编码格式的问题？
- 是不是缓冲区大小设置不对？
- 是不是播放器参数配置错了？

### 后来发现：
通过抓包并查看原始字节流，发现：
- **每一帧前有两个额外的字节（2 Bytes）**
- 这两个字节代表该帧的长度（大端存储）

### 这个行为完全没有在科大讯飞的官方 API 文档中说明。

最终是在一个 Star 不到 10 的 GitHub 仓库中找到如下描述：

> “Opus 宽带：每一帧前用 2 Byte 保存这一帧的数据长度，保存的格式为大端。”

---

## 解决方法

### ✅ 正确的解码流程：

1. 接收整段 TTS 音频流；
2. 按照以下方式逐帧解析：

```python
offset = 0
while offset < len(data):
    # 读取两个字节（大端），作为帧长
    frame_len = int.from_bytes(data[offset:offset+2], byteorder='big')
    offset += 2

    # 读取这一帧的 Opus 数据
    frame_data = data[offset:offset+frame_len]
    offset += frame_len

    # 将 frame_data 送入解码器处理
