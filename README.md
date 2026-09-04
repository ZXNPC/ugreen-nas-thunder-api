# 通过 API 操作 UGREEN NAS 内置迅雷应用

> **字太多不看？翻到最下面看[懒人版](#10-懒人版照着复制就行)。**
> **但至少看看[安全模型](#1-安全模型)一章，以防隐私泄露和安全问题。**
> **看不懂的话可以把全文喂给 AI，让它一步步教你。**

适用于 NAS 应用中心安装的**绿联自带迅雷应用**（非 Docker 版 xunlei/Thor）。
目标：不登录 Web UI、不用账号密码，纯 HTTP 接口提交下载任务并指定落盘目录。
**以下方案实测在 DH4300 Plus 机器上可行。** 其他机型若遇到接口或路径差异，以「故障排查」表逐项对照。

## 0. 变量约定

全流程只改这一处，后续命令与脚本都复用这些值。

```bash
NAS_IP="<你的NAS局域网IP>"              # 例:192.168.x.x
NAS_PORT="<NAS服务端口>"                # 绿联 Web 入口端口，默认 9999；改过就以实际为准
BASE="http://${NAS_IP}:${NAS_PORT}/ugreen/v1/thunder"
SPACE="DEVICESPACE:1000"               # 本地设备空间，固定常量，不是凭证
XUNLEI_DEFAULT="<迅雷默认下载目录>"      # 例:/volume1/迅雷下载/
MOUNT_POINT="<宿主机上的下载目录>"       # 例:/volume1/Downloads/（你想让文件落这里）
```

## 1. 安全模型

迅雷应用本身只靠 `pan-auth` 一个 JWT 鉴权，`device-space` 是公开常量。谁持有有效 token，谁就能读写这个下载空间。整套方案的安全边界完全取决于**你用什么方式让脚本拿到 token**，而不是 token 本身有多强。

风险面：持有有效 token 即可列目录（读到你的下载清单）、提交任意 URL、删除任务（含删文件）。**注意：走本文 API 直接提交不受迅雷免费账号的每日任务配额约束，也就是说一旦被外人拿到 token，灌满你的磁盘没有配额这道兜底。** 因此必须确认 `${NAS_PORT}` 没有被端口转发、UPnP 或反向代理导出到公网。**若已导出，方案 A 等同于把迅雷开放给互联网，请只用方案 B 或 C。**

| 方案 | 做法 | 谁能取到 token | 适用 |
|---|---|---|---|
| C（最安全） | NAS 本机 cron/定时任务抓 token 写文件，只读挂载进容器 | 只有 NAS 本机进程与挂载该文件的容器 | 服务跑在 NAS 上的容器里 |
| B（折中） | 加 `raw/` 入口 + 密钥头 + IP 白名单 | 仅白名单内且持密钥的调用方 | 局域网内可控环境 |
| A（最省事） | 裸加 `raw/` location | 任何能访问 `${NAS_PORT}` 的设备 | 完全隔离的可信内网 |

## 2. 步骤 1：准备免会话的 token 入口

`GET ${BASE}/` 会被绿联会话校验拦截，没有浏览器 Cookie 时返回 `code:1024 Login expired`，而 token 就嵌在这个首页里。按上表选一个方案。

### 方案 A：裸 `raw/` 入口（仅可信内网）

可访问：局域网内任意设备、任意容器、以及公网（若 `${NAS_PORT}` 已导出）。
不可访问：无限制 —— 这是它的优点也是它的风险。

```bash
sudo cp /etc/nginx/conf.d/xunlei_serv.conf /etc/nginx/conf.d/xunlei_serv.conf.bak-$(date +%Y%m%d)

sudo tee -a /etc/nginx/conf.d/xunlei_serv.conf > /dev/null <<'EOF'
location = /ugreen/v1/thunder/raw/ {
    proxy_pass http://127.0.0.1:5050/;
    proxy_set_header Host $host;
}
EOF

sudo nginx -t && sudo nginx -s reload
curl -s "${BASE}/raw/" | head -c 300     # 应返回 HTML，不是 Login expired
```

### 方案 B：`raw/` 入口 + 密钥头 + IP 白名单（推荐）

默认配置下的可达性矩阵（容器网络以 `docker-compose.yml` 的默认 bridge 网段为例）：

| 调用方 | 能否取 token | 说明 |
|---|---|---|
| NAS 本机进程（`127.0.0.1`） | 能 | 需带密钥头 |
| NAS 上的 Docker 容器 | 能 | 默认 bridge 网段 `172.16.0.0/12`；compose 自定义网络落在此段内，若改过 `subnet` 要同步改 `allow` |
| 绿联系统自带的迅雷 Web UI | 不受影响 | 走原有会话鉴权，本 location 是精确匹配，不改变其行为 |
| 局域网里其他电脑上的脚本 | **不能** | 被 `deny all` 拦下。要在 PC 上直接跑，把 `allow` 加上该 PC 的 IP |
| 公网 | **不能** | 除非你已把 `${NAS_PORT}` 导出，此时仅多一道密钥头 |

```bash
RAW_KEY="$(openssl rand -hex 24)"; echo "记住这个密钥: ${RAW_KEY}"

sudo tee -a /etc/nginx/conf.d/xunlei_serv.conf > /dev/null <<EOF
location = /ugreen/v1/thunder/raw/ {
    if (\$http_x_thunder_raw_key != "${RAW_KEY}") { return 403; }
    allow 127.0.0.1;
    allow 172.16.0.0/12;      # Docker 默认私有网段，按实际 compose 网络调整
    # allow <你电脑的局域网IP>;   # 需要在 PC 上直接跑脚本时再放开
    deny all;
    proxy_pass http://127.0.0.1:5050/;
    proxy_set_header Host \$host;
}
EOF

sudo nginx -t && sudo nginx -s reload
curl -s -o /dev/null -w '%{http_code}\n' "${BASE}/raw/"                    # 期望 403
curl -s -H "X-Thunder-Raw-Key: ${RAW_KEY}" "${BASE}/raw/" | head -c 300    # 期望 HTML
```

> 密钥头只保护 `raw/` 这一个 URL。取到 token 之后调 `drive/v1/*` 不受此限制，因为那些接口本来只认 `pan-auth`。

### 方案 C：不碰 nginx，由 NAS 本机投喂 token

可访问：只有 NAS 本机与挂载了该文件的容器。对外零新增暴露面。
不可访问：局域网其他机器（需要 token 时得从 NAS 上取文件）。

```bash
mkdir -p /etc/thunder-token && chmod 700 /etc/thunder-token

cat > /usr/local/bin/refresh-thunder-token.sh <<'EOF'
#!/bin/sh
curl -s http://127.0.0.1:5050/ \
  | grep -oP 'function\s+uiauth\([^)]*\)\s*\{\s*return\s*"\K[^"]+' \
  > /etc/thunder-token/pan_auth.tmp && mv /etc/thunder-token/pan_auth.tmp /etc/thunder-token/pan_auth
chmod 600 /etc/thunder-token/pan_auth
EOF
chmod +x /usr/local/bin/refresh-thunder-token.sh
/usr/local/bin/refresh-thunder-token.sh          # 先手工跑一次，确认文件非空
head -c 32 /etc/thunder-token/pan_auth

(crontab -l 2>/dev/null; echo '0 */6 * * * /usr/local/bin/refresh-thunder-token.sh') | crontab -
```

容器侧只读挂载：`-v /etc/thunder-token/pan_auth:/run/secrets/pan_auth:ro`。

> 若 `/etc/nginx/conf.d/xunlei_serv.conf` 不存在，用 `sudo grep -rl "127.0.0.1:5050" /etc/nginx/` 定位实际承载迅雷反代的 conf。NAS 开了 HTTPS 时 `BASE` 改 `https://`，自签证书给 curl 加 `-k`。部分机型 `/bin/sh` 无 crontab，方案 C 可改用绿联「计划任务」UI 调同一个脚本。

---

## 3. 步骤 2：取 pan-auth

它是迅雷 Web UI 自带的 UIAuth JWT，嵌在首页 HTML 的这段 JS 里：

```javascript
function uiauth() { return "eyJhbGciOi..." }   // 引号里的值就是 pan-auth
```

```bash
UIAUTH=$(curl -s "${BASE}/raw/" \
  | grep -oP 'function\s+uiauth\([^)]*\)\s*\{\s*return\s*"\K[^"]+')
echo "${UIAUTH:0:24}..."
```

方案 B 加 `-H "X-Thunder-Raw-Key: ${RAW_KEY}"`；方案 C 改成 `UIAUTH=$(cat /etc/thunder-token/pan_auth)`。
手工兜底：浏览器打开迅雷 → DevTools → Network → 任意 XHR → Request Headers 里的 `pan-auth`。

token 有效期约 3 天且无刷新接口，不要长期缓存。后续请求统一带：

```bash
-H "device-space: ${SPACE}" -H "pan-auth: ${UIAUTH}"
```

## 4. 步骤 3：把自定义下载目录挂进迅雷设备树

迅雷只能往**它自己设备树里认识的目录**写文件，随便指一个宿主目录会被拒。

```bash
curl -s -X POST "${BASE}/device/v1/vfs" \
  -H "device-space: ${SPACE}" -H "pan-auth: ${UIAUTH}" -H "Content-Type: application/json" \
  -d "{\"type\":\"mount_fs\",\"config\":{\"mount_path\":\"${MOUNT_POINT}\"}}"
```

成功响应里带 `file_id`，它就是提交任务时用的 `parent_folder_id`，记下来。

## 5. 步骤 4：给 thunder 用户授权

只挂载不授权，列目录会报 `open ${MOUNT_POINT}: permission denied`。

**关键点：必须用绿联自带的 `/bin/ugacltool`，POSIX `setfacl` 在这里无效。**

先查迅雷默认下载目录上现成的 ACE，把 `user:1001` 那一条原样抄下来：

```bash
sudo /bin/ugacltool getace "${XUNLEI_DEFAULT}"
```

DH4300 Plus 实测输出（其他机型 uid 排列可能不同，以你自己的输出为准）：

```
[0]    user:1000:allow:rwxpdDaARWc--:-fd-  (level:0)
[1]    user:1001:allow:rwxpdDaARWc--:-fd-  (level:0)
[2]    group:10:allow:rwxpdDaARWc--:-fd-   (level:0)
```

从上例第 `[1]` 行取出的 ACE 字符串即 `user:1001:allow:rwxpdDaARWc--:-fd-`。`1001` 是迅雷应用运行用户 `thunder` 的 uid（别抄成相邻的 `user:1000`；要确认本机 uid 可 `grep -E 'thunder|xunlei' /etc/passwd`）。结尾 `-fd-` 是继承标志，照抄别改。

```bash
sudo /bin/ugacltool addace "${MOUNT_POINT}" "user:1001:allow:rwxpdDaARWc--:-fd-"

# 验证已生效
sudo /bin/ugacltool getace "${MOUNT_POINT}" | grep 'user:1001'
```

## 6. 步骤 5：提交下载任务

```bash
# 6.1 拿本地空间 target（形如 <device_id>#...）
TARGET=$(curl -s -X POST "${BASE}/device/info/watch?space=$(printf %s "$SPACE" | jq -sRr @uri)" \
  -H "device-space: ${SPACE}" -H "pan-auth: ${UIAUTH}" -d '{}' | jq -r '.target')
echo "target=${TARGET}"

# 6.2 提交 URL 到指定目录
curl -s -X POST "${BASE}/drive/v1/task?pan_auth=${UIAUTH}&device_space=${SPACE}" \
  -H "device-space: ${SPACE}" -H "pan-auth: ${UIAUTH}" -H "Content-Type: application/json" \
  -d '{
    "type": "user#download-url",
    "name": "demo.mp4",
    "file_name": "demo.mp4",
    "file_size": "0",
    "space": "'"$TARGET"'",
    "params": {
      "target": "'"$TARGET"'",
      "url": "https://example.com/demo.mp4",
      "total_file_count": "1",
      "parent_folder_id": "<步骤3的file_id>",
      "parent_folder_path": "'"$MOUNT_POINT"'",
      "mime_type": "application/octet-stream",
      "file_id": ""
    }
  }'
```

成功判据：响应 `HttpStatus == 0`。**`space` 和 `params.target` 必须都填 `target` 的值（`device_id#...`），并且与 `params.parent_folder_path` 指向同一位置，否则报 `code:1005`** —— 该码字面像缺 Cookie，实际是 space 填错。

不填 `parent_folder_id` / `parent_folder_path` 时，回退到 `${XUNLEI_DEFAULT}` 与顶层目录发现。

**关于每日配额：** 迅雷免费账号在 Web UI 上有每日任务数限制，但**实测通过本文 API 直连提交可以绕过该配额**，适合批量投递。因此批量任务不必再为省配额而排队，请自行留意磁盘容量与带宽。

## 7. 任务查询与控制

```bash
# 列表。phase 过滤值：PENDING / RUNNING / PAUSED / ERROR / COMPLETE
FILTERS=$(jq -cn --arg p RUNNING '{phase:{eq:"PHASE_TYPE_\($p)"}}' | jq -sRr @uri)
curl -s "${BASE}/drive/v1/tasks?space=${TARGET}&page_token=&filters=${FILTERS}&limit=100&pan_auth=${UIAUTH}&device_space=${SPACE}" \
  -H "device-space: ${SPACE}" -H "pan-auth: ${UIAUTH}"
```

进度从每条任务的 `params.real_path`、`params.speed`、`file_size` 读。列表接口有 `expires_in` 缓存，状态滞后 1~3 秒，轮询要留间隔。

绿联网关用**伪方法路由**转发 PATCH/DELETE：

```bash
# 暂停 / 恢复
curl -s -X POST "${BASE}/method/patch/drive/v1/task" \
  -H "device-space: ${SPACE}" -H "pan-auth: ${UIAUTH}" -H "Content-Type: application/json" \
  -d '{"set_params":{"spec":{"phase":"pause"}}}'      # 恢复用 "running"

# 删任务并删文件：PATCH action=delete

# 删任务但保留文件（只对已完成任务安全）
curl -s -X DELETE "${BASE}/drive/v1/tasks?task_ids=<ID>&pan_auth=${UIAUTH}" \
  -H "device-space: ${SPACE}" -H "pan-auth: ${UIAUTH}"
```

两个坑：进行中的任务用 `DELETE` 会**连文件一起删**；多个 `task_ids` 要重复该参数，逗号分隔返回 `invalid_argument`。

## 8. Python 客户端 `ugreen_thunder.py`

```python
"""UGREEN NAS 内置迅雷应用 API 客户端（仅依赖 requests）。

前置：已完成文档步骤 1~4（raw/ 入口、目录挂载、ugacltool 授权）。
"""
import json
import re
import time
from pathlib import Path
from urllib.parse import quote

import requests

# ========== 只需要改这一段 ==========
BASE_URL = "http://<你的NAS局域网IP>:<NAS服务端口>/ugreen/v1/thunder"
MOUNT_POINT = "<宿主机上的下载目录>"       # 例：/volume1/Downloads
XUNLEI_DEFAULT = "<迅雷默认下载目录>"      # 例：/volume1/迅雷下载，仅用于自查授权
RAW_KEY = None                            # 方案 B：nginx 密钥头；A/C 留 None
PAN_AUTH_FILE = None                      # 方案 C：例 /run/secrets/pan_auth；A/B 留 None
POLL_INTERVAL = 5                         # 秒；列表接口有 1~3 秒缓存
# ====================================

DEVICE_SPACE = "DEVICESPACE:1000"         # 本地设备空间，固定常量
TIMEOUT = 30
UIAUTH_RE = re.compile(r'function\s+uiauth\([^)]*\)\s*\{\s*return\s*"([^"]+)"')


class UgreenThunder:
    def __init__(self, base_url=BASE_URL, raw_key=RAW_KEY, pan_auth=None,
                 pan_auth_file=PAN_AUTH_FILE, session=None):
        self.base_url = base_url.rstrip("/")
        self.raw_key = raw_key
        self.pan_auth = pan_auth
        self.pan_auth_file = Path(pan_auth_file) if pan_auth_file else None
        self.s = session or requests.Session()
        self.target = None                # 形如 <device_id>#...
        self.default_path = None          # 迅雷自带默认下载目录

    # ---------- 鉴权 ----------
    def login(self) -> str:
        """解析 pan-auth。已有注入值/注入文件时不抓页面，可完全不依赖 raw/ 入口。"""
        if self.pan_auth:
            return self.pan_auth
        if self.pan_auth_file and self.pan_auth_file.exists():
            token = self.pan_auth_file.read_text(encoding="utf-8").strip()
            if token:
                self.pan_auth = token
                return token
        headers = {"device-space": DEVICE_SPACE}
        if self.raw_key:
            headers["X-Thunder-Raw-Key"] = self.raw_key
        resp = self.s.get(f"{self.base_url}/raw/", headers=headers, timeout=TIMEOUT)
        resp.raise_for_status()
        m = UIAUTH_RE.search(resp.text)
        if not m:
            raise RuntimeError("未找到 pan-auth：检查步骤 1 的 raw/ 入口与密钥头是否生效")
        self.pan_auth = m.group(1)
        return self.pan_auth

    def _headers(self) -> dict:
        return {"device-space": DEVICE_SPACE, "pan-auth": self.login()}

    def _auth_params(self) -> dict:
        # 部分接口除 header 外还要求 query 带 pan_auth
        return {"pan_auth": self.pan_auth, "device_space": DEVICE_SPACE}

    # ---------- 设备与目录 ----------
    def watch_info(self) -> dict:
        """本地下载空间 target + 迅雷默认下载目录。"""
        resp = self.s.post(f"{self.base_url}/device/info/watch",
                           params={"space": quote(DEVICE_SPACE, safe="")},
                           json={}, headers=self._headers(), timeout=TIMEOUT)
        resp.raise_for_status()
        data = resp.json()
        self.target = str(data.get("target") or "") or None
        downloads = data.get("downloads") or []
        if downloads:
            self.default_path = str(downloads[0].get("path") or "").rstrip("/") or None
        return data

    def list_mounts(self) -> list:
        resp = self.s.get(f"{self.base_url}/device/v1/vfs", headers=self._headers(), timeout=TIMEOUT)
        resp.raise_for_status()
        data = resp.json()
        return data.get("files") or data.get("items") or data.get("mounts") or []

    def mount_host_dir(self, host_path: str) -> str:
        """补做步骤 3：把宿主目录挂进迅雷设备树，返回 file_id。

        步骤 4 的 ugacltool 授权没做时，这里会成功但列目录 permission denied。
        """
        resp = self.s.post(f"{self.base_url}/device/v1/vfs",
                           json={"type": "mount_fs",
                                 "config": {"mount_path": host_path.rstrip("/")}},
                           headers=self._headers(), timeout=TIMEOUT)
        resp.raise_for_status()
        file_id = str(resp.json().get("file_id") or "").strip()
        if not file_id:
            raise RuntimeError(f"挂载 {host_path} 失败：{resp.text[:200]}")
        return file_id

    def folder_id_for_path(self, abs_path: str):
        """按宿主绝对路径反查 folder id：先查挂载表，再扫顶层目录 RealPath。"""
        if not self.target:
            self.watch_info()
        want = abs_path.rstrip("/")
        for entry in self.list_mounts():
            mounted = str((entry.get("config") or {}).get("mount_path") or "").rstrip("/")
            fid = str(entry.get("file_id") or entry.get("id") or "").strip()
            if mounted == want and fid and self._real_path(fid) == want:
                return fid
        filters = quote(json.dumps({"kind": {"eq": "drive#folder"}}), safe="")
        resp = self.s.get(f"{self.base_url}/drive/v1/files",
                          params={"space": self.target, "limit": 100, "parent_id": "",
                                  "filters": filters,
                                  "with": quote("withCategoryDownloadPath,withCategoryDiskMountPath", safe="")},
                          headers=self._headers(), timeout=TIMEOUT)
        resp.raise_for_status()
        data = resp.json()
        # 返回键是 files，不是 items；读错键会静默查不到目录
        for item in data.get("files") or []:
            real = str((item.get("params") or {}).get("RealPath") or "").rstrip("/")
            if real == want:
                return str(item.get("id") or "") or None
        return None

    def _real_path(self, file_id: str):
        try:
            resp = self.s.get(f"{self.base_url}/drive/v1/files/{file_id}",
                              headers=self._headers(), timeout=TIMEOUT)
            resp.raise_for_status()
            return str((resp.json().get("params") or {}).get("RealPath") or "").rstrip("/") or None
        except Exception:
            return None

    # ---------- 任务 ----------
    def submit_url(self, url: str, filename: str, download_root: str = None) -> dict:
        """提交下载 URL。download_root 传宿主绝对路径即为自定义下载位置。

        API 直提不受迅雷免费账号的每日任务配额限制，可批量投递。
        """
        if not self.target:
            self.watch_info()
        root = (download_root or self.default_path or "").rstrip("/")
        parent_folder_id = self.folder_id_for_path(root)
        if not parent_folder_id:
            raise RuntimeError(f"迅雷设备树里找不到 {root}；先 mount_host_dir() 并确认步骤 4 授权已完成")
        payload = {
            "type": "user#download-url",
            "name": filename, "file_name": filename, "file_size": "0",
            # space 与 params.target 必须一致且与 parent_folder_path 对应，否则 code:1005
            "space": self.target,
            "params": {"target": self.target, "url": url, "total_file_count": "1",
                       "parent_folder_id": parent_folder_id, "parent_folder_path": root,
                       "mime_type": "application/octet-stream", "file_id": ""},
        }
        resp = self.s.post(f"{self.base_url}/drive/v1/task", json=payload,
                           params=self._auth_params(), headers=self._headers(), timeout=TIMEOUT)
        resp.raise_for_status()
        data = resp.json()
        if data.get("HttpStatus") != 0:
            raise RuntimeError(f"提交任务失败：{data.get('msg') or data}")
        return data

    def list_tasks(self, phase: str = None, limit: int = 100) -> list:
        """phase: PENDING / RUNNING / PAUSED / ERROR / COMPLETE"""
        if not self.target:
            self.watch_info()
        params = {"space": self.target, "page_token": "", "limit": limit, **self._auth_params()}
        if phase:
            params["filters"] = json.dumps({"phase": {"eq": f"PHASE_TYPE_{phase}"}})
        resp = self.s.get(f"{self.base_url}/drive/v1/tasks", params=params,
                          headers=self._headers(), timeout=TIMEOUT)
        resp.raise_for_status()
        data = resp.json()
        return data.get("tasks") or data.get("files") or []

    def wait_done(self, filename: str, timeout: int = 3600):
        """轮询直到出现同名完成任务；返回该任务，超时返回 None。"""
        deadline = time.monotonic() + timeout
        while time.monotonic() < deadline:
            for task in self.list_tasks("COMPLETE"):
                p = task.get("params") or {}
                if str(p.get("real_path") or "").endswith(filename):
                    return task
            time.sleep(POLL_INTERVAL)
        return None

    def set_phase(self, task_id: str, phase: str) -> dict:
        """phase: 'pause' 或 'running'。UGREEN 用伪方法路由转发 PATCH。"""
        resp = self.s.post(f"{self.base_url}/method/patch/drive/v1/task",
                           json={"set_params": {"spec": {"phase": phase}}, "task_ids": task_id},
                           params=self._auth_params(), headers=self._headers(), timeout=TIMEOUT)
        resp.raise_for_status()
        return resp.json()

    def delete_task_keep_file(self, task_ids: list) -> dict:
        """仅对已完成任务安全；进行中删除会连带删文件。多 id 重复参数，逗号分隔报 invalid_argument。"""
        resp = self.s.delete(f"{self.base_url}/drive/v1/tasks",
                             params=[("task_ids", t) for t in task_ids] + list(self._auth_params().items()),
                             headers=self._headers(), timeout=TIMEOUT)
        resp.raise_for_status()
        return resp.json()


if __name__ == "__main__":
    # 步骤 1~4 已完成，这里直接调用：提交一个 URL 到自定义目录
    t = UgreenThunder()
    print("pan-auth:", t.login()[:24], "...")
    t.watch_info()
    print("target:", t.target, "| 迅雷默认目录:", t.default_path)

    folder_id = t.folder_id_for_path(MOUNT_POINT)
    if folder_id is None:
        folder_id = t.mount_host_dir(MOUNT_POINT)   # 补做步骤 3；步骤 4 授权须在 NAS 上手工做
        print("已挂载，file_id:", folder_id)
    print("parent_folder_id:", folder_id)

    t.submit_url("https://example.com/a.mp4", "demo.mp4", download_root=MOUNT_POINT)
    for task in t.list_tasks("RUNNING"):
        p = task.get("params") or {}
        print("下载中:", p.get("real_path"), p.get("speed"), task.get("file_size"))
```

## 9. Python token 自动续期 `thunder_token_daemon.py`

pan-auth 约 3 天过期且没有刷新接口。常态化运行时把它写进本地缓存，业务侧只读缓存，避免每次提交都抓页面；nginx 密钥头也集中在这个进程里，不必散落到各处。

```python
"""迅雷 pan-auth 常驻续期：定时抓 token 写入本地缓存，供业务进程读取。

配合方案 A/B：本进程负责抓 token。
配合方案 C：把抓 token 交给 NAS 上的 cron，本进程不需要。
"""
import argparse
import base64
import json
import os
import re
import sys
import time
from pathlib import Path

import requests

# ========== 只需要改这一段 ==========
BASE_URL = "http://<你的NAS局域网IP>:<NAS服务端口>/ugreen/v1/thunder"
RAW_KEY = None                        # 方案 B 填密钥头；方案 A 留 None
CACHE_FILE = ".pan_auth.cache"        # 业务进程读同一个路径
REFRESH_INTERVAL = 6 * 3600           # token 约 3 天过期，6 小时刷一次足够
# ====================================

UIAUTH_RE = re.compile(r'function\s+uiauth\([^)]*\)\s*\{\s*return\s*"([^"]+)"')


def fetch(base_url, raw_key):
    headers = {"device-space": "DEVICESPACE:1000"}
    if raw_key:
        headers["X-Thunder-Raw-Key"] = raw_key
    resp = requests.get(f"{base_url.rstrip('/')}/raw/", headers=headers, timeout=30)
    resp.raise_for_status()
    m = UIAUTH_RE.search(resp.text)
    if not m:
        raise RuntimeError("页面中未找到 pan-auth，检查 raw/ 入口与密钥头")
    return m.group(1)


def jwt_exp(token):
    """解出 JWT 的 exp，用于日志与告警。失败返回 None。"""
    try:
        seg = token.split(".")[1]
        seg += "=" * (-len(seg) % 4)
        return int(json.loads(base64.urlsafe_b64decode(seg)).get("exp") or 0) or None
    except Exception:
        return None


def write_cache(path, token):
    path.parent.mkdir(parents=True, exist_ok=True)
    tmp = path.with_suffix(path.suffix + ".tmp")
    tmp.write_text(token, encoding="utf-8")
    os.replace(tmp, path)          # 原子替换，避免业务读到半截
    try:
        os.chmod(path, 0o600)
    except OSError:
        pass


def run_forever(args):
    cache = Path(args.cache_file)
    while True:
        try:
            token = fetch(args.base_url, args.raw_key)
            write_cache(cache, token)
            exp = jwt_exp(token)
            left = f"，剩余约 {(exp - time.time()) / 86400:.1f} 天" if exp else ""
            print(f"[ok] pan-auth 已更新 -> {cache}{left}", flush=True)
        except Exception as exc:
            print(f"[warn] 刷新失败：{exc}；沿用旧缓存", flush=True)
        time.sleep(args.interval)


def check(args):
    cache = Path(args.cache_file)
    if not cache.exists():
        print(f"[fail] 缓存不存在：{cache}")
        return 1
    token = cache.read_text(encoding="utf-8").strip()
    if not token:
        print("[fail] 缓存为空")
        return 1
    exp = jwt_exp(token)
    if exp is None:
        print("[ok] 缓存有值，但不是可解析的 JWT")
        return 0
    left = (exp - time.time()) / 86400
    if left < 1:
        print(f"[warn] token 将在 {left * 24:.1f} 小时后失效，续期进程可能已停")
        return 1
    print(f"[ok] token 有效，剩余约 {left:.1f} 天")
    return 0


if __name__ == "__main__":
    p = argparse.ArgumentParser(description=__doc__)
    p.add_argument("--base-url", default=os.environ.get("THUNDER_BASE_URL", BASE_URL))
    p.add_argument("--raw-key", default=os.environ.get("THUNDER_RAW_KEY") or RAW_KEY)
    p.add_argument("--cache-file", default=os.environ.get("THUNDER_TOKEN_FILE", CACHE_FILE))
    p.add_argument("--interval", type=int, default=REFRESH_INTERVAL)
    p.add_argument("--once", action="store_true", help="只刷一次，适合放进 crontab")
    p.add_argument("--check", action="store_true", help="检查缓存是否有效，适合做健康检查")
    a = p.parse_args()

    if a.check:
        sys.exit(check(a))
    if a.once:
        try:
            write_cache(Path(a.cache_file), fetch(a.base_url, a.raw_key))
            print(f"[ok] 已写入 {a.cache_file}")
        except Exception as exc:
            print(f"[fail] {exc}")
            sys.exit(1)
        sys.exit(0)
    sys.exit(run_forever(a))
```

业务侧接上它，只需把 `UgreenThunder` 指到同一个缓存文件：

```python
from ugreen_thunder import UgreenThunder

t = UgreenThunder(pan_auth_file=".pan_auth.cache")
t.submit_url("https://example.com/b.mp4", "b.mp4", download_root="/volume1/Downloads")
```

三种部署形态：

```bash
# 常驻进程
python thunder_token_daemon.py

# 只刷一次，放进 crontab 每 6 小时
0 */6 * * * cd /srv/app && python thunder_token_daemon.py --once && python main.py

# docker-compose：sidecar + 共享卷
#   services:
#     token-refresh:
#       command: python thunder_token_daemon.py --cache-file /tokens/pan_auth
#       volumes: [tokens:/tokens]
#     app:
#       environment:
#         THUNDER_TOKEN_FILE: /tokens/pan_auth
#       volumes: [tokens:/tokens:ro]
#   volumes:
#     tokens: {}
```

`POLL_INTERVAL`、`REFRESH_INTERVAL` 属经验值，不是协议要求。

## 10. 懒人版：照着复制就行

SSH 进 NAS 执行，把开头三个占位符换掉，其余原样粘贴。每步后面紧跟验证命令，看到预期输出再继续。

```bash
# ==== 第 0 步：设好这几个变量 ====
NAS_PORT="9999"                                                   # 默认 9999，改过就填实际值
XUNLEI_DEFAULT="/volume1/迅雷下载/"                                # 迅雷默认下载目录
MOUNT_POINT="/volume1/Downloads/"                                 # 你想让文件落的目录
BASE="http://127.0.0.1:${NAS_PORT}/ugreen/v1/thunder"             # 在 NAS 本机跑，用 127.0.0.1 最省事
```

```bash
# ==== 第 1 步：开 raw/ 免会话入口（更安全做法见正文方案 B/C） ====
sudo cp /etc/nginx/conf.d/xunlei_serv.conf /etc/nginx/conf.d/xunlei_serv.conf.bak-$(date +%Y%m%d)
sudo tee -a /etc/nginx/conf.d/xunlei_serv.conf > /dev/null <<'NGINXEOF'
location = /ugreen/v1/thunder/raw/ {
    proxy_pass http://127.0.0.1:5050/;
    proxy_set_header Host $host;
}
NGINXEOF
sudo nginx -t && sudo nginx -s reload

# ==== 验证 1：应看到 HTML，而不是 Login expired ====
curl -s "${BASE}/raw/" | head -c 300
```

```bash
# ==== 第 2 步：取 pan-auth ====
UIAUTH=$(curl -s "${BASE}/raw/" | grep -oP 'function\s+uiauth\([^)]*\)\s*\{\s*return\s*"\K[^"]+')

# ==== 验证 2：打印出 eyJ... 开头即成功 ====
echo "pan-auth = ${UIAUTH:0:24}..."
```

```bash
# ==== 第 3 步：把目录挂进迅雷设备树，抓 file_id ====
curl -s -X POST "${BASE}/device/v1/vfs" \
  -H "device-space: DEVICESPACE:1000" -H "pan-auth: ${UIAUTH}" -H "Content-Type: application/json" \
  -d "{\"type\":\"mount_fs\",\"config\":{\"mount_path\":\"${MOUNT_POINT}\"}}"
# 从上面的输出里复制 file_id 的值，粘到下一行
FILE_ID="<把上一步的 file_id 粘这里>"

# ==== 验证 3：非空即成功 ====
echo "file_id = ${FILE_ID}"
```

```bash
# ==== 第 4 步：授权（ACE 抄迅雷默认目录 getace 输出里 user:1001 那条） ====
sudo /bin/ugacltool getace "${XUNLEI_DEFAULT}"          # 先看一眼，确认 user:1001 那行内容
sudo /bin/ugacltool addace "${MOUNT_POINT}" "user:1001:allow:rwxpdDaARWc--:-fd-"

# ==== 验证 4：能 grep 出 user:1001 即已生效 ====
sudo /bin/ugacltool getace "${MOUNT_POINT}" | grep 'user:1001'
```

```bash
# ==== 第 5 步：拿 target 并提交一个测试任务 ====
SPACE="DEVICESPACE:1000"
TARGET=$(curl -s -X POST "${BASE}/device/info/watch?space=$(printf %s "$SPACE" | jq -sRr @uri)" \
  -H "device-space: ${SPACE}" -H "pan-auth: ${UIAUTH}" -d '{}' | jq -r '.target')
echo "target = ${TARGET}"

curl -s -X POST "${BASE}/drive/v1/task?pan_auth=${UIAUTH}&device_space=${SPACE}" \
  -H "device-space: ${SPACE}" -H "pan-auth: ${UIAUTH}" -H "Content-Type: application/json" \
  -d '{"type":"user#download-url","name":"ugreen-api-test.mp4","file_name":"ugreen-api-test.mp4","file_size":"0","space":"'"${TARGET}"'","params":{"target":"'"${TARGET}"'","url":"https://proof.ovh.net/files/1Mb.dat","total_file_count":"1","parent_folder_id":"'"${FILE_ID}"'","parent_folder_path":"'"${MOUNT_POINT}"'","mime_type":"application/octet-stream","file_id":""}}'

# ==== 验证 5：响应 HttpStatus 为 0 即成功；再确认文件落地 ====
ls -l "${MOUNT_POINT}"
```

懒人版收尾提醒：测试 URL 换成任意可下载的直链即可；API 直提不受免费账号每日配额限制，但请留意磁盘容量；`space` 里的 `:` 无需转义；跑通之后如果不想每次手工抓 token，回去看第 9 节的续期脚本。

## 11. 故障排查

| 现象 | 原因与处理 |
|---|---|
| `code:1024 Login expired` | 步骤 1 入口没生效或路径打错。查 `sudo nginx -t` 与是否 reload |
| 方案 B 取 token 返回 403 | 密钥头名/值不匹配，或来源 IP 不在 `allow` 内。`docker network inspect` 看实际子网；局域网 PC 需显式 `allow` |
| 连不上 `${BASE}` | 端口不对。迅雷只监听 `127.0.0.1:5050`，必须经 `${NAS_PORT}`（默认 9999）的 nginx 入口 |
| 找不到 pan-auth | 首页结构随固件变过。用 `curl -s "${BASE}/raw/"` 抓下页面后搜 `uiauth`，看 token 实际嵌法 |
| `code:1005` | `space` 填错，不是缺 Cookie。`space` 和 `params.target` 都要用 `device_id#...` 形式 |
| 目录能挂但列不出来 | 漏了步骤 4。必须用 `ugacltool addace`，`setfacl` 无效；确认抄的是 `user:1001` 那条而非 `user:1000` |
| 任务没落到指定目录 | 定位靠 `parent_folder_id` / `parent_folder_path`，不是改迅雷默认目录。校验 folder id 的 `RealPath` |
| 提交任务静默找不到目录 | 读了 `items` 键；接口返回的是 `files` |
| 文件落成 `xxx(1).mp4` | 目标目录已有同名文件，迅雷加 `(1)` 后缀而不覆盖 |
| 任务状态不更新 | 列表接口 `expires_in` 缓存，滞后 1~3 秒 |
| 跑几天后全部请求失败 | pan-auth 过期（约 3 天）。启用第 9 节的续期进程或方案 C 的 cron |

## 12. 已知限制

- 实测机型 DH4300 Plus。其他机型的 nginx conf 路径、uid、迅雷默认目录名可能不同，以 `getace` 与 `/etc/passwd` 的实际输出为准。
- `pan-auth` 约 3 天过期且无刷新接口，需定期重取（方案 A/B 自动，方案 C 靠 cron）。
- 该接口是绿联 Web UI 的内部接口，无稳定性承诺，固件升级后可能变动。首次接入建议打印一次原始响应确认字段名。
- 自定义目录需一次性 NAS 侧操作（nginx 与 ugacltool），纯 API 无法完成。
- Docker 版第三方迅雷不走这套网关，本文不适用。

## 看不懂就交给 AI

这篇文档写得偏工程化，如果你按步骤操作时卡住了，最省事的做法是把整篇 README 复制下来发给你喜欢的 AI 助手，再告诉它你的机型和具体报错，让它针对你的情况逐步带你做。文中的验证命令输出（例如 `getace` 的结果、`code:1005` 这类错误码）原样贴给它，通常一轮就能定位问题。
