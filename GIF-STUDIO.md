# 动图工坊

入口：`gif-studio.html`，位于世界书首页的“影像”分类，与光影暗房并列。

## 与原工具的对应

原 `create_gif.py` 使用 imageio，手动填写文件名，`duration=500`、`loop=0`。网页版保留默认 500 毫秒和无限循环，通过文件选择或拖放取得图片，不需要填写本机路径。原 Python 文件及用户提供的照片、GIF 均未修改，也未发布。

## 实现

独立 HTML 内嵌界面、Lucide、gifenc、Worker 和公开示例照片，无 CDN、后端、遥测或图片上传接口。程序实现位于 HTML 末尾，可检索 `GifCore`、`gifWorkerMain`、`prepare`、`generate`。

- `prepare`：通过浏览器解码 JPEG / PNG / WebP / AVIF 等静态图片，使用 `createImageBitmap` 的 `from-image` 方向选项，不支持则退回 Image。工作图片等比缩小到长边最多 1280 px，以 PNG Blob 暂存；不保存原 EXIF / GPS。
- `GifCore.dimensions`：按首张比例或指定比例统一输出画布，支持自定义尺寸。完整模式留边，填满模式居中裁切，所有透明区域先合成指定的背景色。
- `GifCore.sequence`：按缩略图顺序播放。往返模式追加不含首尾重复帧的反向序列，例如 `[A,B,C,B]`。
- 停留时长：50 到 10000 毫秒，按 GIF 的 10 毫秒精度取整。默认时长影响以后添加的图片；应用全部会修改现有帧。GIF 在不同播放器中的实际计时可能有差异。
- `gifWorkerMain`：gifenc 1.0.3 在单独 Web Worker 中进行 PNN 色彩量化、调色板映射、GIF / LZW 编码。每帧独立调色板，最多 64 / 128 / 256 色。不是 CSS 假动画，不是视频容器改扩展名。
- 主线程逐帧解码与统一尺寸，只传输一帧 RGBA 给 Worker，等待编码确认后继续，不同时囤积全序列的未压缩输出帧。生成中禁止编辑，支持终止 Worker 取消。
- gifenc `repeat=0` 对应无限，`repeat=-1` 不写循环扩展，对应一次；三次对应两次重复。生成结果预览直接显示 Blob GIF，下载保留同一 Blob 的原字节。
- 任何画面、时长、顺序变化都会使旧输出失效，避免下载过时结果。文件名改变不重编码，只修改下载名。文件名由 `GifCore.filename` 清理。

## 隐私

图片、缩略图和生成结果仅存在于使用者自己的浏览器内存、Blob URL 与主动下载的本机文件中。网站所有者、其他访问者、GitHub 和本工具的开发者都不会自动收到这些图片。没有 localStorage、照片持久化、访问分析脚本或第三方图像 API。刷新或关闭页面后需要重新添加图片。

静态网页本身由 GitHub Pages 提供，因此页面访问仍会向托管方发送正常网络请求，例如 IP、时间和所请求的页面路径；这不包含所选择图片的内容或文件名。用户主动点击来源链接时会打开对应网站。公开发布的是应用代码与许可允许的示例照片，不是访客照片。

## 能力边界

- 最多 40 张，单文件 20 MB，原文件合计 120 MB。解码后单张最多 5000 万像素，输出长边或自定义宽高最多 1280 px，含往返帧的输出总量不超过 4000 万像素。
- HEIC / HEIF 依赖浏览器解码能力；不支持时先转换 JPEG。已有 GIF、视频不接受；动画 WebP / AVIF 不展开，只作为浏览器解码出的静态图使用。
- 这是图片序列动图，不生成新动作、不做 AI 插帧、不保留声音。单张图片会得到静态 GIF。
- GIF 每帧最多 256 色，不支持半透明，当前采用背景色合成。色彩渐变可能出现色带；没有声称无损保留摄影图的全部颜色。
- 界面序列播放与输出采用相同顺序和时长。浏览器 GIF 播放器可能自行限制最短时长；序列预览不是逐毫秒计时基准。
- PNG 中间图和顺序编码降低内存峰值，但手机解码超大图片仍可能消耗较多内存，建议先缩小。没有跨设备同步或自动恢复。

## 来源与许可

- [gifenc 官方项目与 API](https://github.com/mattdesl/gifenc)：1.0.3，MIT。编码、PNN 量化与调色板映射由该项目实现，许可全文内嵌 HTML。
- [MDN createImageBitmap](https://developer.mozilla.org/en-US/docs/Web/API/Window/createImageBitmap)：浏览器图片与方向处理。
- [MDN Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers)：后台线程。
- [Lucide](https://github.com/lucide-icons/lucide)：0.468.0，ISC，沿用世界书的图标依赖。
- 示例：[Luca Bravo / Lago di Braies](https://unsplash.com/photos/brown-house-near-body-of-water-zAjdgNXsMeg)，[Unsplash License](https://unsplash.com/license)。示例序列是公开示例照片的不同取景，不包含用户本地的 team-pic 照片。
