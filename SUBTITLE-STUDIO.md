# 字幕工坊

地址：https://zoara111.github.io/world-notebook/subtitle-studio.html

## 目录约定

世界书一级分类保持「影像」「学习」。影像细分「照片」「动图」「视频」，已有 URL 不变，主页支持 `#image`、`#photo`、`#gif`、`#video`、`#learn` 深链接。新视频能力作为独立工具加入视频分类，避免让字幕编辑器承担全部影像工作。当前没有视频调色功能或后端服务。

## 数据与隐私

视频、声音、字幕在访问者的浏览器内处理，不发送到本站、GitHub、Hugging Face 或其他识别服务器。网站所有者不能看到访问者选中的视频。页面不配置账号、遥测、上传接口、云端字幕存储。

首次识别会从 jsDelivr 下载固定版本的 Transformers.js / ONNX Runtime，从 Hugging Face 下载 Whisper 权重、词表及配置。模型和运行库请求会暴露通常的网络信息（如 IP 和请求资源），但请求中没有媒体内容或字幕。模型可能缓存在浏览器 Cache API 中。工程只在主动下载时保存到磁盘；刷新页面会清除未下载的工作，不自动持久化私人视频。

## 实现

- Transformers.js 3.8.1，模块 Worker 中推理，WASM 单线程以兼容 GitHub Pages 无跨源隔离的环境。优先尝试 WebGPU，初始化或推理失败时切换 WASM。
- 模型 `onnx-community/whisper-base_timestamped`，固定 revision `608c49e61301901684bc36cac8f74b95ff6b5a8e`。GPU: encoder fp32 / merged decoder q4。CPU: q8。下载体积依设备路径而异，界面展示各文件的真实进度。
- Web Audio `decodeAudioData` 读取完整视频文件中的声音，混为单声道，重采样至 16 kHz。视频只用本地 object URL 播放。
- `return_timestamps: 'word'`, 30 秒分块、5 秒重叠，输出词级时间戳。词内汉字按照词的起止时间均分，**不等于精确汉字或音素强制对齐**。纯静音先拦截，背景音乐、方言、人名仍可能识别错误。
- 自动分句兼顾标点、停顿、最多 5.5 秒和每句字数。支持逐字出现、逐字高亮、整句三种显示模式。完整句子先布局，再逐字绘制，避免不断跳动。
- 手动文本修正后该句逐字时间按整句重新均分；修改起止时间按比例拉伸原逐字时间。整体偏移、重新分句、撤销操作只处理字幕。
- 预览和成片共用 Canvas 字幕渲染器。位置、宽度、字号、安全边界、自动换行用于横屏与竖屏；长句必要时缩小字号。字体使用本机系统字体，换设备可能有细微差异。
- 导出通过 `canvas.captureStream(30)` + Web Audio 的 `MediaStreamDestination` + `MediaRecorder`，把声音和烧录字幕的画面合成视频。用 `MediaRecorder.isTypeSupported` 选择 MP4 或 WebM；不保证原视频编码格式或无损。导出按实际时长运行，离开前台时主动停止以防缺帧。
- SRT/VTT 保存整句起止时间和文本，不保存字体或逐字效果；JSON 工程保存样式与逐字时间，不包含原视频。工程导入需先载入对应视频。
- 模型下载失败、无声、格式不支持等情况给出可恢复错误；支持取消、手动添加字幕或导入现有字幕，不制造假识别结果。

## 当前限制

- 视频上限 250 MB、5 分钟。浏览器需能播放画面并用 Web Audio 解码音轨，推荐 MP4（H.264 + AAC）或 WebM。MOV/HEVC 支持取决于设备；不支持时需要先转换。
- 首次模型下载约 80–220 MB（CPU 模型约 77 MB，GPU 模型约 206 MB；不含可选 GPU 失败后追加的 CPU 模型）。本地推理占内存和计算资源；弱性能手机建议短片或电脑处理。无需 API key，不使用服务器版付费识别。
- 中文 base 模型是轻量取舍，不保证准确率；应校对结果。词级时间戳也可能不准确。
- 音视频导出属于浏览器实时录制，帧率和同步精度受设备负载影响，不能替代专业离线编码器。离开页面会中止录制。不支持的浏览器仍可导出字幕文件。
- 没有说话人分离、翻译、云端队列、高清视频离线编码或视频调色。

## 后续边界

未来若增加云端高精度识别，应作为明确可选入口，单独说明接收方、费用、保留策略，并在上传前获得同意。不要把目前「本机处理」默默改成上传。视频调色可以复用画面渲染输出链，但应有独立处理模块；目录中增加独立视频工具即可。

## 依赖与来源

- [Transformers.js 官方 ASR API](https://huggingface.co/docs/transformers.js/v3.8.1/api/pipelines) / [源码与 Apache-2.0](https://github.com/huggingface/transformers.js/tree/3.8.1)
- [官方逐词时间戳示例](https://github.com/huggingface/transformers.js/tree/v3/examples/whisper-word-timestamps)
- [timestamped ONNX 模型](https://huggingface.co/onnx-community/whisper-base_timestamped) / [基础 Whisper 模型 MIT](https://huggingface.co/openai/whisper-base)
- [ONNX Runtime MIT](https://github.com/microsoft/onnxruntime/blob/main/LICENSE)
- [Web Audio 解码](https://developer.mozilla.org/en-US/docs/Web/API/BaseAudioContext/decodeAudioData) / [Canvas 录制流](https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/captureStream) / [MediaRecorder 格式判断](https://developer.mozilla.org/en-US/docs/Web/API/MediaRecorder/isTypeSupported_static)
- Lucide 0.468.0，ISC；完整许可证包含在字幕页面 HTML 的注释内。
- 演示照片：Luca Bravo，[Unsplash 原图](https://unsplash.com/photos/brown-house-near-body-of-water-zAjdgNXsMeg)，[Unsplash License](https://unsplash.com/license)。演示中动态平移由浏览器绘制，不是用户上传的视频。
- 演示声音：John F. Kennedy 1961 就职演讲片段，公共领域美国政府录音，[JFK Library](https://www.jfklibrary.org/learn/about-jfk/historic-speeches/inaugural-address)，使用 [Transformers.js 官方演示音频](https://huggingface.co/datasets/Xenova/transformers.js-docs/resolve/main/jfk.wav)。演示同样走实际视频解码与模型识别流程，没有预置假字幕。
