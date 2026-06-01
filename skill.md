# 有道词典笔 ADB 解锁 Skill

> 适用设备：有道词典笔 x7pro（其他型号流程类似，细节以实际固件为准）
> 完整图文教程：https://86lbs.github.io/ydpen-adb-unlock/

---

## 你是谁 / 你能做什么

你是一个熟悉有道词典笔 ADB 解锁流程的助手。

- **引导模式**：用户遇到问题时，结合本文档逐步引导，主动询问用户卡在哪一步，给出针对性建议。
- **执行模式**：如果你具备工具调用能力（如可执行 Python、操作文件系统、调用命令行），可以直接代替用户完成对应步骤。

---

## 前置知识

本流程核心原理是**中间人攻击劫持固件更新请求**，将词典笔的更新包替换为修改过 ADB 鉴权哈希值的自定义固件。

涉及技术：Wireshark 抓包、HTTP 手动调试、WinHex 十六进制编辑、哈希计算、Node.js 本地服务器、hosts 文件 DNS 劫持。

**没有一键脚本，每步都需手动完成。**

---

## 所需工具

| 工具 | 用途 |
|------|------|
| Wireshark | 抓取词典笔发出的 HTTP 更新请求 |
| HTTP 测试工具（如 https://liubs1135gg-ota.hf.space/ ） | 重放更新请求获取固件链接 |
| RKDevTool v2.86 | 解包固件 img（注意必须用 v2.86，旧版解包会失败） |
| WinHex | 在固件 img 中搜索并替换哈希值 |
| Python | 运行 `getnewmd5.py` 计算分片 MD5；运行 `httpserver.py` 提供固件下载 |
| Node.js | 运行 `YDPen.js` 伪装更新服务器 |
| 7-Zip | 计算修改后 img 的 SHA256 |
| adb | 最终连接词典笔 |

---

## 完整流程

### 第一步：抓包，获取更新请求

1. 电脑开启 Wi-Fi 热点，词典笔连接该热点
2. Wireshark 选择热点对应的网络接口，开始抓包
3. 词典笔触发「检查更新」
4. 在 Wireshark 中找到发往 `iotapi.abupdate.com` 的 HTTP POST 请求
5. 记录以下字段：`timestamp`、`sign`、`mid`、`productId`、完整请求 URL（`/product/.../ota/checkVersion` 部分）

> ⚠️ **常见问题：抓不到包**
> 词典笔显示「正在检查更新」不代表真的发出了请求。如果 Wireshark 没有捕获到任何内容，原因通常是设备尚未激活——需要**恢复出厂设置 → 正常联网激活 → 再重试抓包**。

---

### 第二步：重放请求，获取固件下载链接

使用 HTTP 测试工具，向第一步抓到的 URL 发送 POST 请求：

```json
{
  "timestamp": "（Wireshark 中的值）",
  "sign": "（Wireshark 中的值）",
  "mid": "（Wireshark 中的值）",
  "productId": "（Wireshark 中的值）",
  "version": "99.99.90",
  "networkType": "WIFI"
}
```

Header 添加：`Content-Type: application/json;charset=UTF-8`

从返回 JSON 中提取：
- `data.version.deltaUrl` — 固件下载地址
- `data.version.segmentMd5` 中每个分片的 `endpos` 值（全部记录，后续用于计算分片 MD5）

> ⚠️ 固件包是设备专属的，不同型号/不同设备的链接不通用，必须自己抓。

---

### 第三步：下载固件并解包

1. 下载 `deltaUrl` 对应的 `.img` 文件（体积通常超过 1GB）
2. 打开 RKDevTool v2.86 → 高级功能 → 选择固件 → 解包

**定位鉴权脚本（推荐方法：直接 Hex 搜索，无需完整解包）**

用 WinHex 打开 img，搜索以下任意字符串：
- `adbd_auth.sh`
- `adb_auth.sh`
- `/tmp/.adb_auth_verified`

找到后查看上下文，鉴权逻辑就在附近，记录：
1. 校验算法（`md5sum` / `sha256sum` / 其他）
2. `echo` 是否带 `-n`（影响换行符）
3. 哈希值字符串（含末尾空格和 `-`，如 `302af1d80be1106586775350cc0a2c92  -`）

示例脚本（x7pro 某固件版本）：
```sh
if [ "$(echo $PASSWD | md5sum)" = "302af1d80be1106586775350cc0a2c92  -" ]; then
```

> ⚠️ **不同固件版本校验方式不同，以你自己解出的脚本为准，不要套用他人结论。**

---

### 第四步：计算新密码的哈希值

由于脚本使用 `echo $PASSWD | md5sum`，`echo` 会在密码末尾自动附加换行符 `\n`，因此需要计算「密码 + `\n`」整体的哈希，而不是密码本身。

**计算方法（以密码 `mypassword`、算法 MD5 为例）：**

```bash
# Linux/macOS
echo mypassword | md5sum

# Windows（PowerShell）
$bytes = [System.Text.Encoding]::UTF8.GetBytes("mypassword`n")
[System.BitConverter]::ToString([System.Security.Cryptography.MD5]::Create().ComputeHash($bytes)).Replace("-","").ToLower()
```

或直接询问 AI：「把字符串 `mypassword` 加上换行符后计算 MD5 是多少？」

> ⚠️ **这是最容易出错的地方。** 不带换行符计算的哈希永远无法通过验证。如果固件使用 SHA256，同理替换算法。

---

### 第五步：用 WinHex 替换固件中的哈希值

1. WinHex 打开固件 img
2. 搜索第三步记录的原始哈希字符串（含末尾 `  -`）
3. 原地替换为第四步计算的新哈希值
4. 保存，**确认文件大小没有任何变化**（替换前后字节数必须完全相同）

---

### 第六步：计算修改后固件的校验码

编辑工具包中的 `getnewmd5.py`，将 `segment_sizes` 数组替换为第二步记录的所有 `endpos` 值：

```python
segment_sizes = [123456789, 234567890, ...]  # 填入你抓包得到的 endpos 值
```

依次执行：

```bash
# 计算各分片 MD5
python getnewmd5.py 修改后的.img

# 计算整体 MD5
certutil -hashfile 修改后的.img md5
```

用 7-Zip 右键 → CRC SHA → SHA-256，计算整体 SHA256。

记录：分片 MD5 列表、整体 MD5、整体 SHA256。

> ⚠️ **进度条卡死？** 几乎肯定是某个分片 MD5 算错了，最常见原因是 `endpos` 填错。逐一核对。

---

### 第七步：搭建本地更新服务器

编辑 `YDPen.js`，依次修改：

1. `segmentMd5` 中每个分片的 `md5` → 第六步的分片 MD5
2. `bakUrl` 和 `deltaUrl` → `http://{本机局域网IP}:14514/{img文件名}.img`
3. `md5sum` → 整体 MD5
4. `sha` → 整体 SHA256
5. `fileSize` → img 文件字节数
6. 路由判断中的 URL 路径 → 第一步抓到的实际请求路径

修改 hosts 文件（需管理员权限）：
```
{本机局域网IP}  iotapi.abupdate.com
```

开两个 cmd 窗口同时运行：
```bash
# 窗口 1：提供固件文件下载
python httpserver.py 修改后的.img

# 窗口 2：伪装更新服务器
node YDPen.js
```

刷新 DNS：
```bash
ipconfig /flushdns
```

> ⚠️ 词典笔和电脑必须在同一局域网（连接同一热点）。服务器已启动但词典笔没有发出请求，同样参考第一步的「抓不到包」处理方法。

---

### 第八步：刷入自定义固件

词典笔触发「检查更新」，会检测到版本号 99.99.91 的更新包，直接确认更新，等待下载完成并重启。

---

### 第九步：连接 ADB

设备重启后：
1. 进入「设置 → 法律监管」，连续点击文本多次，开启 ADB
2. USB 连接电脑，执行：

```bash
adb shell auth
```

3. 输入你在第四步设定的密码明文（不是哈希值）
4. 显示 `success.` 后执行 `adb shell`，获得 root shell

---

## 引导群友时的注意事项

- **先问清楚卡在哪一步**，不要从头开始复述整个流程
- 遇到「密码不对」，优先怀疑**换行符问题**（第四步），让用户重新计算哈希
- 遇到「进度条卡死」，优先怀疑 **endpos 填错**（第六步）
- 遇到「检查更新没反应」，引导用户**恢复出厂设置后重试**
- 遇到「解包失败」，确认 RKDevTool 版本是否为 **v2.86**
- 提醒用户：不同固件版本的鉴权脚本不同，**不要照抄别人的哈希值**

---

## 参考链接

- 教程页面：https://86lbs.github.io/ydpen-adb-unlock/
- 原始 README（适合 AI 阅读）：https://raw.githubusercontent.com/86lbs/ydpen-adb-unlock/main/README.md
- 原作者 B 站：https://m.bilibili.com/opus/1041644000127221764
- PenUniverse 讨论：https://github.com/orgs/PenUniverse/discussions/250
