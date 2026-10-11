# 光影暗房

世界书入口 `index.html` 按 `影像 / 学习` 分类。电场章节仍为 `electrostatics.html`；原首页的梯度下降实验保留为 `gradient-descent.html`，原有浏览器学习记录的 key 不变。

## 文件与运行

`photo-studio.html` 是完整的独立 HTML，包含 CSS、应用逻辑、着色器、regl、Lucide 和示例照片。不需要 API 密钥、服务器计算、付费服务或构建步骤。可以直接打开，也可以通过 GitHub Pages 使用。源码在该文件末尾；主要函数与类按名称可检索。

## 处理顺序

1. `decode` 使用 `createImageBitmap` 的 `from-image` 方向选项；不支持时退回浏览器 Image 解码。转换到 sRGB 工作画布，最长边不超过 4096 px 和设备纹理上限；原照片不被覆盖。
2. `analyze` 对最多 256 px 的样本计算亮度与各通道直方图、亮度分位数、近黑/近白通道比例、候选中性色平均对数值。
3. `recommend` 由明暗倾向和中位数建议曝光，按分位数温和调节暗部、亮部和对比度。曝光范围 -0.70 到 +1.05 EV。仅有足够候选中性色时进行有限的 RGB 增益校正。不是机器学习，不识别人物、场景或艺术意图。
4. `effective` 把基础修正与选定风格按强度叠加，避免切换风格时累积滤镜。原图、裁切和参数是独立状态；所有结果重新从原片渲染。
5. `PhotoEngine.render` 使用 regl WebGL。先做局部纹理处理，再进行线性曝光、相对色温/色调、显示参考的明暗调整、曲线、HSL、分离色调、黑白混合、褪色、暗角、颗粒。没有 WebGL 时使用 Worker CPU 后备路径。
6. 导出不包含对比线、近限标记或水印。JPEG 透明区域合成白底，PNG/WebP 保留 alpha。Canvas 重新编码不复制原 EXIF/GPS。导出 WebP 若被浏览器退回 PNG，实际扩展名与提示会相应更改。

## 数学与实现

- `linear / encoded`：sRGB 分段传递函数；曝光在线性光中乘以 `2 ** EV`，不是对 CSS brightness 的包装。
- 温度与 tint：有界的相对 RGB 指数增益，做亮度归一化。不等同于 RAW 白平衡或绝对 Kelvin。
- 明暗：以亮度与区域权重映射目标亮度，随后调整颜色比例。高光变暗不意味着恢复 JPEG 中已被剪掉的细节。
- 曲线：固定输入位置 `[0, .25, .5, .75, 1]`，允许分别编辑 RGB、R、G、B 的输出值。保持单调，使用线性插值。GPU 以 256 项 LUT 采样；CPU 为直接插值，因此可能有少量量化差异。
- HSL：通过环形色相距离和 smoothstep 对八个色域连续加权，再修改 HSL 色相、饱和度和 lightness。低饱和区域降低色域权重，避免中性色被某一个色域误选。HSL lightness 不是 CIE L*。
- 自然饱和度：降低对已高饱和区域和橙色范围的增强。不是人脸检测或肤色蒙版。
- 分离色调：从选定颜色扣除亮度后，按亮度区域权重叠加偏色。
- 锐化：有界的邻域差分增益。降噪：五点近似双边加权。清晰度：较大邻域的亮度局部对比。均为轻量近似，不等同于专业多尺度或 AI 降噪算法。
- 颗粒：与源坐标相关的确定性合成噪声。缩略预览和原尺寸输出的纹理细节、噪声与锐化观感可能有差别，应检查导出文件。
- 直方图：从调整后的缩小图统计。近限提示按任何 RGB 通道 ≥252 或 ≤3 计算；它是预警，不是对传感器 RAW 曝光的判定。

## 配方与视频扩展

配方 JSON 使用 `version: 1`、`engine: world-photo-srgb-v1` 与纯数据状态。不包含照片、脚本或文件路径。`validateRecipe` 检查版本、参数范围、曲线单调性、颜色与裁切比例。最多 30 份本地配方，另可导出备份。

`cubeLUT` 生成 17³ `.cube`，以红通道最快、绿次之、蓝最慢排列，包含纯颜色处理；它排除裁切、锐化、清晰度、降噪、颗粒和暗角。强调整可能在低分辨率 LUT 插值时有差异。

逐帧输入与参数计算分离，因此后续可将视频帧上传给相同 WebGL 纹理/着色器。正式视频版本仍需要单独实现：

- 视频解码与时间戳驱动，持续更新源纹理，防止逐帧重新创建 WebGL 对象。
- 帧率、分辨率、颜色空间与 HDR/SDR 映射约定，避免把 sRGB JPEG 算法当作 HDR 管线。
- WebCodecs 或 MediaRecorder 能力探测、编解码容器支持、进度与取消。
- 音轨保留、A/V 同步和移动端内存限制。

目前没有视频上传或视频导出功能；首页也不放不可操作的占位工具。

## 限制与隐私

当前支持浏览器能够解码的 JPEG、PNG、WebP、AVIF。HEIC/HEIF 取决于浏览器；失败时提示先转 JPEG，不包含 HEIC 转码器。单文件最多 40 MB，解码后超过 1 亿像素拒绝处理。RAW、内容修补、多帧 HDR、透视校正、镜头校准与生成式编辑不在本版本范围。

GPU 工作与导出长边最多 4096 px；CPU 预览 720 px、导出最多 2048 px。用户照片仅在浏览器内存与本机下载中存在；没有上传 API、追踪、远程图片分析或照片持久化。页面包含示例照片与依赖，离线也不需外部资源。浏览器关闭后照片不会自动保留。只保存用户主动保存的参数配方。

## 参考与许可证

- [Adobe 色调与色彩控制](https://helpx.adobe.com/lightroom-classic/desktop/process-and-develop-photos/image-tone-color.html)：参数职责与摄影工作流参考，不声称复刻其算法。
- [W3C sRGB 转换](https://www.w3.org/TR/css-color-4/#color-conversion-code)：分段传递函数。
- [MDN createImageBitmap](https://developer.mozilla.org/en-US/docs/Web/API/Window/createImageBitmap)：方向与浏览器解码。
- [regl 2.1.1](https://github.com/regl-project/regl)：MIT，WebGL 命令、纹理与着色器管理。许可证全文内嵌 HTML。
- [Lucide 0.468.0](https://github.com/lucide-icons/lucide)：ISC，界面图标。许可证全文内嵌 HTML。
- [GPUImage2](https://github.com/BradLarson/GPUImage2)：图像/视频共享滤镜架构参考，未移植其代码。
- 示例照片：[Luca Bravo / Lago di Braies](https://unsplash.com/photos/brown-house-near-body-of-water-zAjdgNXsMeg)，[Unsplash License](https://unsplash.com/license)。摄影作品许可与应用代码分开。
