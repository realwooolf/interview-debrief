# interview-debrief

面试复盘 skill for Claude Code。上传录音或转录文本，自动转录、整理问答、生成总评，保存到本地或飞书文档。

## 安装

```bash
npx skills add realwooolf/interview-debrief
```

## 使用方法

把录音文件路径发给 Claude，说「帮我整理这场面试」：

```
帮我整理这场面试 /Users/xxx/Desktop/面试录音.mp3
```

或者直接上传已有的 PDF 转录文本，skill 会自动识别并跳过转录阶段。

## 前置依赖

```bash
# Apple Silicon Mac（推荐）
pip install mlx-whisper
brew install ffmpeg

# 其他平台
pip install openai-whisper
brew install ffmpeg  # macOS
# sudo apt install ffmpeg  # Ubuntu
```

## 功能

- **自动转录**：本地运行 Whisper，支持 .wav / .m4a / .mp3 / .mp4
- **多文件合并**：多段录音自动按时间排序合并
- **整理输出**：生成完整对话、问答整理、面试总评三份内容
- **双路径保存**：选本地保存为 .md 文件，或保存到飞书文档 + 多维表格

## 飞书配置（可选）

选择保存到飞书时需要配置，选本地保存可跳过。

首次使用时 skill 会引导完成飞书自建应用配置，约 5 分钟。配置完成后运行 skill，浏览器会自动打开完成 OAuth 授权，之后无需重复操作。

## 支持平台

- Apple Silicon Mac（M1/M2/M3/M4）：mlx-whisper GPU 加速，41 分钟音频约 3-5 分钟
- Intel Mac / Windows / Linux：openai-whisper CPU 模式
