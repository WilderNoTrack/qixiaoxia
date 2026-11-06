# 栖小霞 · 栖霞山家庭游览助手

48 小时黑客松作品（2026「枫客松」· 赛道二 栖霞文旅智能体，二等奖）。
面向带娃、推车、带老人的家庭：走不动了能重排路线，每句讲解都讲得出出处。

本仓库只放**技术说明与演示视频**，不含完整项目代码（主办方资料包不对外，代码为比赛提交版本）。

## 演示视频

113 秒，含宣传片与真机 1080p 逐场景录屏（宣传片部分为 AI 生成画面，界面部分为真实录屏，片内已标注）。
视频体积 62 MB，不放在仓库里；需要时可直接联系我，或见简历投递邮件附件。

## 这个项目里与本岗位相关的部分：本机语音链路

语音问答全程在本机完成，不上传音频、不落盘。核心实现在 `server/speech.py`，要点：

- **模型**：faster-whisper（whisper-small 的 ctranslate2 版本），`local_files_only=True` 离线加载
- **设备自适应**：有 CUDA 走 `float16`，否则降到 CPU + `int8`，CPU 线程数可配；Windows 下自动挂 cuBLAS 目录
- **输入约束**：只接受 16kHz / 单声道 / 16bit WAV，VAD 过滤 + `no_speech_prob` 阈值，能量过低直接判「没听到人声」
- **术语热词**：`initial_prompt` 注入 8 个景点名（栖霞寺、明镜湖、千佛斋、太虚亭……），解码后过 OpenCC 繁转简
- **进程隔离**：识别放在独立子进程里跑，180 秒超时 + 崩溃隔离，模型挂掉不会拖死主服务
- **人工确认**：识别结果可逐字修改再提交，不指望模型一次就对

### 题外但相关：用 ASR 反查 TTS

产品要把讲解词念出来，而生成式 TTS 会自己加词、改词。做法是把生成的音频再喂回同一个 ASR，
用拼音序列算相似度，低于 0.9 就不允许播放——用 ASR 校验「念出来的」和「写出来的」是否一致。

```python
clean = lambda s: ''.join(c for c in OpenCC('t2s').convert(s) if c.isalnum())
a = lazy_pinyin(clean(text))        # 想让他念的
b = lazy_pinyin(clean(observed))    # ASR 听到的
if difflib.SequenceMatcher(None, a, b, autojunk=False).ratio() < .9:
    raise SpeechError('配音与文字未能通过核对')
```

## 其他

- 定位：高德 Web 服务做坐标转换，GPS 成功后匹配 16 个景点；精度不足时降级为景区总览并要求手动确认
- 数据：主线 SQLite，`/admin` 本机管理端；小红书、抖音按自然周采集并与历史库跨批去重
- 测试：31 项服务端测试
- 诚实边界：32 个点位的 `verified` 字段全为 `false`，资料没核实的厕所位置、台阶数、票价一律标注「查不到」

---

作者：冯译宽 · [WilderNoTrack.me](https://www.WilderNoTrack.me) · github.com/WilderNoTrack
