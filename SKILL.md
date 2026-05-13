---
name: interview-debrief
description: 面试复盘助手，处理面试录音或转录文本。触发场景：用户提到"面试复盘"、上传音频文件（.wav/.m4a/.mp3/.mp4）需要压缩或转录、上传 PDF 转录文件需要整理、或说"帮我整理这场面试"。
---

# 面试复盘

根据传入的文件类型自动判断执行阶段：
- 传入音频文件（.wav / .m4a / .mp3）或视频文件（.mp4）→ 执行阶段一：本地转录
- 传入 PDF 文件 → 执行阶段二：整理并生成飞书文档

**多文件处理：** 如果用户传入多个音频/视频文件，在阶段一完成后自动执行阶段零（多文件合并排序），再进入阶段二。单文件则跳过合并直接处理。

---

## 前置依赖

使用前确保以下工具已安装：

```bash
# 转录（二选一，按平台选）
pip install mlx-whisper          # Apple Silicon Mac 用，GPU 加速，推荐
pip install openai-whisper        # 其他平台 fallback

# 音视频处理（whisper 内部依赖 ffmpeg 解码音频）
# macOS
brew install ffmpeg
# Ubuntu / Debian
sudo apt install ffmpeg
# Windows
# 前往 https://ffmpeg.org/download.html 下载，或用 winget install ffmpeg
```

**平台说明：**
- Apple Silicon Mac（M1/M2/M3/M4）→ 使用 mlx-whisper，速度快 5-10 倍
- Intel Mac / Windows / Linux → 自动 fallback 到 openai-whisper（CPU 模式）

---

## 飞书配置

使用前需要创建自己的飞书应用，完成后将凭证填入 skill。

### 第一步：创建飞书自建应用

1. 打开 [open.feishu.cn](https://open.feishu.cn)，登录你的飞书账号
2. 点击右上角「开发者后台」→「创建应用」→ 选「自建应用」
3. 填写应用名称（随意，如 `interview-debrief`）和描述，创建完成

### 第二步：开启权限

1. 进入应用后，左侧菜单点「权限管理」
2. 搜索并开启以下两个权限：
   - `docx:document`（以应用身份读写文档内容）
   - `bitable:app`（以应用身份读取多维表格）
3. 点击「申请权限」，选择「无需审核，直接开启」

### 第三步：添加回调地址

1. 左侧菜单点「安全设置」
2. 找到「重定向 URL」，点击添加，填入：
   ```
   http://localhost:9998/callback
   ```
3. 保存

### 第四步：发布应用

1. 左侧菜单点「应用发布」→「版本管理与发布」
2. 创建一个版本，点击「申请线上发布」
3. 如果是个人自用，选择「仅对内测用户可见」即可，不需要审核

### 第五步：获取凭证并填入 skill

1. 左侧菜单点「凭证与基础信息」，复制 **App ID** 和 **App Secret**
2. 在 SKILL.md 中找到以下两处，替换成你自己的值：

   **飞书配置章节：**
   ```
   APP_ID = "your_app_id"      ← 替换这里
   APP_SECRET = "your_app_secret"  ← 替换这里
   ```

   **「获取飞书 user_access_token」代码块（`# 从 skill 配置中读取` 注释下方）：**
   ```python
   APP_ID = "your_app_id"      ← 替换这里
   APP_SECRET = "your_app_secret"  ← 替换这里
   ```

配置完成后，第一次使用时会自动打开浏览器完成飞书 OAuth 授权，之后 token 自动缓存，无需重复操作。

```
TOKEN_CACHE = os.path.expanduser("~/.claude/feishu_token.json")
```

---

## 阶段一：音频/视频处理

**触发条件：** 文件扩展名为 .wav、.m4a、.mp3、.mp4

### 转录

先检测平台，选择对应的转录方式：

```python
import platform, subprocess, os

def is_apple_silicon():
    return platform.system() == "Darwin" and platform.machine() == "arm64"

def transcribe(files):
    for f in files:
        out = f.rsplit('.', 1)[0] + '.txt'
        if is_apple_silicon():
            # mlx-whisper：Apple Silicon GPU 加速，41 分钟音频约 3-5 分钟
            import mlx_whisper
            result = mlx_whisper.transcribe(
                f, path_or_hf_repo="mlx-community/whisper-large-v3-turbo", language="zh"
            )
        else:
            # openai-whisper：有 NVIDIA GPU 自动用 CUDA，否则 CPU
            import whisper, torch
            model = whisper.load_model("small")
            fp16 = torch.cuda.is_available()
            result = model.transcribe(f, language="zh", fp16=fp16)
        open(out, 'w', encoding='utf-8').write('\n'.join(s['text'].strip() for s in result['segments']))
        print('完成:', out)
```

注意：
- mlx-whisper 首次运行会自动下载模型（约 800MB），之后缓存本地
- **转录命令启动后，立即在对话框中询问阶段二所需的信息**（复盘类型、日期、公司名等），无需等转录完成。转录结束后直接用已收集的信息进入阶段零（多文件）或阶段二（单文件）。

---

## 阶段零：多文件合并排序（自动触发）

**触发条件：** 阶段一产出 2 个及以上 .txt 转录文件时自动执行，无需用户操作。

执行步骤：

1. 读取所有 .txt 文件内容
2. 根据文件名时间戳 + 内容语义判断顺序：
   - 文件名含时间戳（如 1733、215639）→ 按时间升序排列
   - 内容含开场白（自我介绍、你好）→ 排最前
   - 内容含道别（拜拜、谢谢）→ 排最后
   - 内容有明显上下文衔接 → 按语义连贯性判断
3. 将排序结果告知用户确认（列出文件顺序和判断依据），确认后进入阶段二
4. 若顺序不确定，询问用户

---

## 阶段二：面试整理 → 生成飞书文档

**触发条件：** 文件扩展名为 .pdf，或阶段一/阶段零完成后的转录文本

启动前先收集以下信息（如果用户没有提供，主动询问）：
- **复盘类型**（面试 / 导师指导 / 其他）→ 决定后续文档模板和信息头
- **日期**（格式：YYYY-MM-DD）

若类型为**面试**，追加询问：
- 公司名
- 岗位名
- 面试轮次（如：一面、二面、终面）

若类型为**导师指导**，追加询问：
- 指导人姓名
- 主题（如：作品集、求职方向、项目复盘）

若类型为**其他**，追加询问：
- 对方身份
- 主题

根据类型生成对应的文档标题和信息头：

**面试：**
```
{公司名} 完整对话
{公司名} 问答整理
{公司名} 面试总评

信息头：公司 / 岗位 / 轮次 / 日期
```

**导师指导：**
```
{指导人} 完整对话
{指导人} 问答整理
{指导人} 指导总评

信息头：指导人 / 性质：{主题} / 日期
```

**其他：**
```
{对方身份} 完整对话
{对方身份} 问答整理
{对方身份} 总评

信息头：对方身份 / 主题 / 日期
```

### 1. 获取飞书 user_access_token

从配置中读取 APP_ID / APP_SECRET，TOKEN_CACHE 路径做 `~` 展开：

```python
import json, time, os, subprocess, tempfile
import http.server, threading, webbrowser, urllib.parse

# 从 skill 配置中读取，用户需自行填写
APP_ID = "your_app_id"
APP_SECRET = "your_app_secret"
TOKEN_CACHE = os.path.expanduser("~/.claude/feishu_token.json")
REDIRECT_URI = "http://localhost:9998/callback"

def load_token():
    try:
        with open(TOKEN_CACHE) as f:
            return json.load(f)
    except:
        return None

def save_token(data):
    os.makedirs(os.path.dirname(TOKEN_CACHE), exist_ok=True)
    with open(TOKEN_CACHE, "w") as f:
        json.dump(data, f)

def curl_post_auth(url, data, headers=None):
    payload = json.dumps(data, ensure_ascii=False)
    with tempfile.NamedTemporaryFile(mode='w', suffix='.json', delete=False, encoding='utf-8') as f:
        f.write(payload)
        fname = f.name
    cmd = ["curl", "-sk", "-X", "POST", url, "--noproxy", "*",
           "-H", "Content-Type: application/json"]
    if headers:
        for h in headers:
            cmd += ["-H", h]
    cmd += ["--data", f"@{fname}"]
    try:
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=30)
    finally:
        os.unlink(fname)
    return json.loads(result.stdout)

def get_app_token():
    d = curl_post_auth(
        "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal",
        {"app_id": APP_ID, "app_secret": APP_SECRET})
    return d.get("tenant_access_token")

def oauth_flow():
    app_token = get_app_token()
    code_holder = {}
    class Handler(http.server.BaseHTTPRequestHandler):
        def do_GET(self):
            parsed = urllib.parse.urlparse(self.path)
            params = urllib.parse.parse_qs(parsed.query)
            code_holder["code"] = params.get("code", [None])[0]
            self.send_response(200)
            self.end_headers()
            self.wfile.write(b"<html><body>Done! You can close this tab.</body></html>")
        def log_message(self, *args): pass
    server = http.server.HTTPServer(("localhost", 9998), Handler)
    t = threading.Thread(target=server.handle_request)
    t.start()
    auth_url = (f"https://open.feishu.cn/open-apis/authen/v1/authorize"
                f"?app_id={APP_ID}&redirect_uri={urllib.parse.quote(REDIRECT_URI)}"
                f"&scope=docx%3Adocument%20bitable%3Aapp&response_type=code")
    print("正在打开飞书授权页面...")
    webbrowser.open(auth_url)
    t.join(timeout=120)
    code = code_holder.get("code")
    if not code:
        raise Exception("未获取到授权码，请重试")
    d = curl_post_auth(
        "https://open.feishu.cn/open-apis/authen/v1/access_token",
        {"grant_type": "authorization_code", "code": code},
        headers=[f"Authorization: Bearer {app_token}"])
    d = d.get("data", {})
    token_data = {
        "access_token": d["access_token"],
        "refresh_token": d.get("refresh_token", ""),
        "expires_at": int(time.time()) + d.get("expires_in", 7200) - 300
    }
    save_token(token_data)
    return token_data["access_token"]

def get_token():
    cached = load_token()
    if cached and time.time() < cached.get("expires_at", 0):
        return cached["access_token"]
    return oauth_flow()
```

首次运行会打开浏览器授权，之后 token 自动缓存复用，无需重复操作。

### 2. 读取 PDF

用 Read 工具读取 PDF 全文。

### 3. 识别发言人

根据对话内容判断：
- 做自我介绍、介绍项目、回答问题的是**我**
- 提问、追问、给反馈的是**面试官**

### 4. 生成三个飞书文档

**重要注意事项（调试确认）：**
- 用 subprocess curl 调用飞书 API，绕过 Python SSL 栈（Python 3.14 + 本地代理会导致 SSL handshake 失败）
- 不要使用 `text_color` 参数（会报 99992402 field validation failed）
- 批量写入每批不超过 20 块，超过容易超时
- token 获取方式：先用 app_id/app_secret 换 tenant_access_token，再用它换 user_access_token

```python
import json, time, subprocess, tempfile, os

def curl_post(url, data, token):
    payload = json.dumps(data, ensure_ascii=False)
    with tempfile.NamedTemporaryFile(mode='w', suffix='.json', delete=False, encoding='utf-8') as f:
        f.write(payload)
        fname = f.name
    try:
        result = subprocess.run(
            ["curl", "-sk", "-X", "POST", url, "--noproxy", "*",
             "-H", "Content-Type: application/json",
             "-H", f"Authorization: Bearer {token}",
             "--data", f"@{fname}"],
            capture_output=True, text=True, timeout=60)
    finally:
        os.unlink(fname)
    if not result.stdout.strip():
        return {}
    return json.loads(result.stdout)

def create_doc(token, title):
    d = curl_post("https://open.feishu.cn/open-apis/docx/v1/documents", {"title": title}, token)
    if d.get("code") != 0:
        print(f"create_doc failed: {d}")
        return None
    return d["data"]["document"]["document_id"]

def write_blocks(token, doc_id, blocks):
    for i in range(0, len(blocks), 20):
        batch = blocks[i:i+20]
        for attempt in range(3):
            r = curl_post(
                f"https://open.feishu.cn/open-apis/docx/v1/documents/{doc_id}/blocks/{doc_id}/children",
                {"children": batch, "index": -1}, token)
            if r.get("code") == 0:
                break
            print(f"  batch {i} attempt {attempt+1} failed: {r.get('code')}, retrying...")
            time.sleep(1.5)
        time.sleep(0.5)

def p(text):
    return {"block_type": 2, "text": {"elements": [{"text_run": {"content": text}}], "style": {}}}

def bold(text):
    return {"block_type": 2, "text": {"elements": [{"text_run": {"content": text, "text_element_style": {"bold": True}}}], "style": {}}}

def mixed_p(parts):
    # parts = [("文字", is_bold), ...]，用于发言人加粗+正文不加粗的混排段落
    elements = []
    for text, is_bold in parts:
        el = {"text_run": {"content": text}}
        if is_bold:
            el["text_run"]["text_element_style"] = {"bold": True}
        elements.append(el)
    return {"block_type": 2, "text": {"elements": elements, "style": {}}}

def h1(text):
    return {"block_type": 3, "heading1": {"elements": [{"text_run": {"content": text}}], "style": {}}}

def h2(text):
    return {"block_type": 4, "heading2": {"elements": [{"text_run": {"content": text}}], "style": {}}}

def divider():
    return {"block_type": 22, "divider": {}}
```

#### 每个文档开头写入基础信息块

```
公司：{公司名}
岗位：{岗位名}
轮次：{面试轮次}
日期：{面试日期}
────────────────────────
```

---

#### 文档一：完整对话

- 全程对话按时间顺序，**面试官：** 加粗，**我：** 加粗（不加颜色）
- 保留完整内容，不删减

---

#### 文档二：问答整理

- 每组问答独立分块，加二级标题（Q1、Q2……）
- 标题概括该组问答的核心话题
- 包含候选人的反问部分
- 每组问答之间空一行

---

#### 文档三：面试总评

根据对话内容灵活调整结构，基础框架：

**整体印象**（一句话定调）

**做得好的地方**（具体举例，引用原文）

**明显问题**（具体举例，说明为什么是问题）

**优化建议**（可操作，针对这场面试的具体情况）

如有特殊情况（面试官明确反馈、压力性问题、明显失误等），单独增加板块说明。

### 5. 面试总评同时在对话框内输出

三个文档创建完毕后，将面试总评内容直接输出在对话框中，方便即时查看。

### 6. 完成提示

告知用户三个飞书文档已创建，并输出每个文档的链接（格式：`https://bytedance.feishu.cn/docx/{document_id}`）。
