# rh-send 服务与环境（RH 短剧通用）

> 提交视频前通读。含接口、故障排查、图像生成环境。

---

## rh-send 本地服务

| 项 | 值 |
|---|---|
| 服务地址 | `http://127.0.0.1:8282` |
| 代码位置 | `H:\_Project\Python\AITest\RH_Send\main.py` |
| 启动方式 | **由用户手动启动**：`cd H:\_Project\Python\AITest\RH_Send && python main.py` |
| 落盘目录 | `H:\_Project\Python\AITest\RH_Send\save\<YYYY-MM-DD>\<task_id>.mp4` |
| 任务映射 | `H:\_Project\Python\AITest\RH_Send\task_map.json`（本地 task_id → 上游 taskId） |

> 🔴 **服务由用户启动，不要代劳**，重复启动会抢端口。
> 提交前只探测 `GET /docs`（HTTP 200 = 在线）；不通就打印启动提示并停止。

---

## 🔴 代理陷阱（最高频故障）

环境变量里常设 `HTTP_PROXY=http://127.0.0.1:8026`。**urllib 会把 localhost 请求也送进代理**，
代理连不上本地服务 → 返回 **502 Bad Gateway**。

**症状极具迷惑性**：连 `/docs` 都 502，看起来像服务彻底坏了；socket 能连上但 HTTP 超时，
像进程挂死。真因是代理拦截。

**修法**：所有访问本地服务的请求显式禁用代理。

```python
NO_PROXY_OPENER = urllib.request.build_opener(urllib.request.ProxyHandler({}))
req = urllib.request.Request(f'{SERVICE}/docs')
with NO_PROXY_OPENER.open(req, timeout=6) as r: ...
```

**排查顺序**：502 → 先看代理环境变量 → 直连 socket 测端口 → 都通才怀疑服务本身。

---

## 接口

### POST /file_upload — 上传参考图

- `multipart/form-data`，字段名 **`file`**（+ 可选 query 参数 `api_key`）
- 返回**上游原始 JSON**（结构 `{code, message, data}`，`code == 0` 为成功）：

```json
{ "code": 0, "message": "success",
  "data": { "type": "image",
            "fileName": "openapi/61432ac1….png",
            "download_url": "https://…?q-sign-algorithm=…",
            "size": "3490" } }
```

- 🔴 **取 `data.fileName` 填进 `/gen_video` 的 `images`** —— 不是 `image_id`，不是 `download_url`
- `fileName` 是服务端相对路径，**不能拼成外链访问**；`download_url` 带签名且只有一天有效期
- 每张图上传约 1-3 秒；一个单元 5-7 张时上传耗时可观
- 上传失败（上游非 200）返回 `502`，上游响应体在 `detail` 里

### POST /gen_video — 提交生成

```json
{
  "task_id": "EP01U04",            // 本地命名，也是落盘文件名（不含扩展名）
  "prompt": "<完整提示词>",
  "images": ["openapi/….png", …],  // /file_upload 的 data.fileName；有图走 I2VA，无图走 T2VA
  "duration": "12",                // ⚠️ 字符串
  "aspect_ratio": "16:9 (Widescreen)"   // ⚠️ 传完整枚举，见下
}
```

- **图片上限 = 服务端 `NODE_IMAGES` 节点数（当前 7 张）**，超出返回 400
  `最多支持 7 张输入图片`；图片按顺序 zip 到 nodeId `585 / 587 / 573 / 564 / 565 / 566 / 567`
- 🔴 **`aspect_ratio` 要传完整枚举值，不要传裸冒号形式**（实测踩坑）：
  上游 `ResolutionSelector` 节点（node 572）只认枚举：
  `1:1 (Square)` / `2:3 (Portrait Photo)` / `3:2 (Photo)` / `3:4 (Portrait Standard)` /
  `4:3 (Standard)` / `9:16 (Portrait Widescreen)` / `16:9 (Widescreen)` / `21:9 (Ultrawide)`。
  传裸 `16:9` 会被原样透传 → 上游报
  `aspect_ratio: '16:9' not in [... ]` → 任务 **0 秒即 FAILED**。
  `main.py` 里虽有 `resolve_aspect_ratio()` 负责转译，但**运行中的服务进程可能落后于源码**；
  **传完整枚举值在两种版本下都正确**（新版 `split()[0]` 取回 `16:9` 再转译，结果一致），
  所以一律传完整枚举，不依赖服务端版本。
- 🔴 **提交必须校验响应体，不能只看 HTTP 状态码**：
  上游出错时服务端**直接透传上游原始响应**，HTTP 仍 200。
  判据：响应里必须有 **`ok: true` 且 `upstream_task_id` 非空**，否则按失败处理并把 `msg` 打出来。
- 🔴 **`code 605` 的 `msg` 是误导性的，它写 `NOT_ENOUGH_BALANCE`，但实测不是余额问题。**
  真因是**同时运行的任务数超过上游并发上限（约 5 个）**。
  实测证据：5 个任务在跑时提交第 6 个 → 返回 605；
  等手上全部跑完、0 个在跑时提交**同样的请求** → 直接成功。
  **不要去充值，要限流。** 处理方式见下「并发调度」。
- `duration` 服务端**不校验整数**，但仍按整数秒传、小数向上取整
  （6.5→7、11.5→12），宽松不代表安全，上游节点怎么处理小数无从得知
- `task_id` 命名约定：`<集号><单元>`，如 `EP01U04`；重生成加后缀 `EP01U04b`
- 返回 `{"ok": true, "task_id": "...", "upstream_task_id": "..."}`，映射写入 task_map.json

> ⚠️ **`task_id` 与 `upstream_task_id` 无关，别混用**，前者是调用方传入的本地标识，
> 后者才是 RunningHub 返回的。后续 `/task_query` 和 `/save_video` 一路带**本地 `task_id`**。

---

### POST /task_query — 查询

```json
{ "task_id": "EP01U04" }
```

**单次查询，不阻塞**，一次请求只向上游查一次就返回，节奏由调用方控制。

返回：

```json
{ "task_id": "…", "upstream_task_id": "…",
  "status": "RUNNING", "finished": false,
  "raw": {…}, "saved": null, "size": null }
```

- `status` → 上游原始状态（`RUNNING` / `SUCCESS` / `FAILED`），**仅作参考，不代表任务成功**
- `finished` → 上游是否已结束（`status` 存在且 ≠ `RUNNING`）
- `saved` / `size` → **仅当 `finished == true` 时才出现**，从 save 目录查该 task_id 的文件
- 查不到映射 → 404「本地没有 task_id=… 对应的任务记录，请先用 /gen_video 提交」
- **轮询间隔 20 秒**，超时设 30 分钟

> ⚠️ **服务端不做任何自动落盘。** `finished=true` 且 `saved=null` 时，
> 代码**不会**替你调 `/save_video`，必须由调用方从 `raw` 里挖出视频直链，
> 自己调 `/save_video` 补落盘。

> 🔴 **`saved` 字段不可直接采信，必须校验时间戳！**
> `find_saved_video()` 的实现是**遍历 save/ 下所有日期目录**，返回第一个匹配
> `{task_id}.*` 的文件。所以同名 task_id 只要历史上跑过一次（哪怕在别的日期），
> `saved` 就会指到**旧文件**上。
>
> **实测踩坑**：提交后 **0 秒** 就打印"✅ 落盘"，其实拿到的是 `save/2026-09-16/` 的
> 昨天成片，归档了一版废片，还以为成功了。注意此时 `status` 常是 `FAILED`，
> 而"`FAILED` 也可能落盘成功"的旧经验会让这个错误更容易蒙混过关。
>
> **正确判据**：落盘文件的 `st_mtime` 必须 **≥ 提交时刻**（留 5 秒容差）。
> 不满足就明确告警后忽略，继续轮询。

> 🔴 **404 要立刻中止，不要重试。**
> `task_query` 对未知 task_id 返回 404「本地没有 task_id=… 对应的任务记录」。
> 这是**提交根本没成功**的信号（余额不足 / 参数被拒），空转轮询 30 分钟毫无意义。
> 遇到 404 立刻抛出并检查提交环节。

### POST /save_video — 下载落盘

```json
{ "task_id": "EP01U04", "url": "<视频直链>" }
```

落盘到 `save/<今天日期 YYYY-MM-DD>/<task_id>.mp4`（相对 `main.py` 所在目录），
目录自动创建，**同名文件直接覆盖**。返回 `{"ok": true, "path": …, "size": …}`。

> 🔴 **这是唯一会写出成片文件的接口**，它跑没跑成功，直接决定任务成败。

---

## 🔴 并发调度：滑动窗口，不是分批

上游**并发上限约 5 个**（超了返回 `code 605`）。调度必须用**滑动窗口**：

```text
✅ 正确：最多 5 个在飞，**任何一个落盘就立刻补上下一个**
❌ 错误：每批 5 个、等这一批**全部**落盘再开下一批
```

**为什么不能分批**：视频生成耗时差异大（7s 片约 5 分钟，14s 片约 10 分钟）。
分批模式下每批都要等最慢的那个，池子会出现空转；
滑动窗口则始终把 5 个槽位填满，整体更快。

**实现**：`concurrent.futures.ThreadPoolExecutor(max_workers=5)`
，一次性把所有单元 `submit` 进去，池子自动保持 5 个在跑、
某线程一结束就取下一个排队任务。**语义天然就是滑动窗口，不用自己维护队列。**

```python
with ThreadPoolExecutor(max_workers=MAX_JOBS) as ex:
    futs = {ex.submit(run_unit, uid, ...): uid for uid in targets}
    for f in as_completed(futs):
        ...
```

**并发时的日志**：多单元进度会交错，每个单元必须加 `[U06]` 前缀，
并给 `print` 加锁，否则输出会串行错乱、无法判断是谁在说话。

---

## 🔴 成功判据：唯一看文件

**只看 `save/<日期>/<task_id>.mp4` 是否存在。**

- 上游 `status: SUCCESS`，不算成功
- `results` 里有视频直链，不算成功
- `/task_query` 返回 `finished: true`，只代表上游任务结束了

**实测反复出现：上游报 `status=FAILED`，但文件正常落盘且完整可播。**
所以 `FAILED` ≠ 失败，必须落到磁盘看文件。

> 因此轮询逻辑应为：`finished=true` 时**先检查 saved 路径/文件是否存在**，
> 不存在再挖直链补调 `/save_video`，最后仍以磁盘文件为准。

---

## 提交前健康检查

```python
def check_service():
    try:
        NO_PROXY_OPENER.open(f'{SERVICE}/docs', timeout=6).close()
        return True
    except Exception as e:
        print(f'❌ rh-send 不可达（{SERVICE}）：{e}')
        print('   先启动服务：cd H:\\_Project\\Python\\AITest\\RH_Send && python main.py')
        return False
```

不先探测就提交，会白传一轮图片（每张 1-3 秒）才在提交时报错。

---

## 并行与耗时

| 项 | 实测 |
|---|---|
| 单个视频生成 | 5-9 分钟（与时长、图片数相关） |
| 7 秒片 | 约 5 分钟 |
| 12 秒片 | 约 9 分钟 |
| 14 秒片 | 约 10 分钟 |
| **上游并发上限** | **约 5 个**（超了返回 `code 605`，见上「并发调度」） |
| 关键帧（gpt-image-2）单张 | 29-70 秒 |
| 关键帧并行 | 可 10 个并行，无显著拖慢 |

**调度纪律**：视频用**滑动窗口**（最多 5 个在飞、任一落盘即补下一个），
**不要分批**（每批等最慢的会空转）、也不要一次全丢（会 605）。
关键帧上限宽松，可 10 并行。

生成完**立即复制归档**到 `EP##/视频/`，按命名规范重命名（见 naming.md）。

---

## 故障速查

| 现象 | 真因 | 处理 |
|---|---|---|
| 一切请求都 502 | HTTP_PROXY 劫持 localhost | `ProxyHandler({})` 禁用代理 |
| 端口通但 HTTP 超时 | 同上（socket 能连的是代理） | 同上 |
| connection refused (10061) | 服务没起 | 探测 `/docs`，提示用户启动（R13） |
| 提交 404「没有任务记录」 | task_map 里无映射 | 先 `/gen_video` 提交 |
| 提交 400「最多支持 7 张」 | 图片超 NODE_IMAGES 数 | 减到 ≤7 |
| 提交 400「不支持的 aspect_ratio」 | 传了表外值 | 传完整枚举值 |
| **任务 0 秒就 FAILED，`ResolutionSelector` 报 `'16:9' not in [...]`** | aspect_ratio 传了裸冒号形式，服务端没转译 | **传完整枚举 `16:9 (Widescreen)`** |
| **提交返 `code 605` / `NOT_ENOUGH_BALANCE`** | **不是余额！是并发超限（约 5 个）** | 用滑动窗口限流，不要充值 |
| **提交看似成功但没有 upstream_task_id** | 上游报错被服务端透传，HTTP 仍 200 | 校验响应体 `ok` + `upstream_task_id` |
| **0 秒就"落盘成功"，文件却是旧的** | `find_saved_video` 扫所有日期目录，命中历史同名文件 | 校验 mtime ≥ 提交时刻 |
| **404 后空转轮询 30 分钟** | 把"无任务记录"当成可重试的网络异常 | 404 立即中止 |
| duration 不生效/异常 | 传了小数 | 字符串整数，向上取整 |
| status=FAILED 但文件在 | 上游状态不可信 | **正常**，以文件为准（但仍要校验时间戳） |
| finished=true 但 saved=null | 直链没自动落盘 | 从 raw 挖直链调 `/save_video` |
| 重生成后旧文件没了 | 被覆盖 | 永不覆盖，task_id 加后缀 |
| 图片传进去了但画面没参考上 | 错把 `image_id` / `download_url` 填进 `images` | 必须填 `/file_upload` 的 `data.fileName` |

---

## ⚠️ 改动 main.py 前必读

`main.py` 里可能已被用户手动改过。**动代码前先读一遍当前内容**，不要凭记忆改。
本文件记录的默认值（apiKey / workflowId / instanceType / duration 默认值 / NODE_IMAGES 节点数）
都只是写作时的实测快照，以代码为准。
