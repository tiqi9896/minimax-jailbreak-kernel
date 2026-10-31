# Hermes 请求层插件写法（注入 system 内核 / 改思考参数）

## 一、插件能挂的三个位置

| API | 用途 |
|---|---|
| `ctx.register_system_prompt_section(id, content_or_callable, position=, max_chars=)` | 把一段文本冻进系统提示词（会话开始时渲染一次） |
| `ctx.register_middleware("llm_request", fn)` | **每次请求**改写 api_kwargs（改 system / 改 thinking / 加参数） |
| `ctx.register_command(name, handler=, description=, args_hint=)` | 会话级开关命令 |

★ **段预算**：单段 ≤ 4000 字符，全部插件段合计 ≤ 8000 字符。超限的段**被静默跳过**，
不报错——所以“段没生效”要去查字符数，不是查日志。

★ **段位置**只有 `after_memory` 一个槽位；渲染顺序按 section id 字典序。

★ **section id 规则**：小写字母/数字/`.`/`_`/`-`，1–128 字符。重复注册同一 id 会抛异常。

## 二、★ Anthropic 兼容模式的形状陷阱（最容易踩）

`llm_request` 中间件拿到的 `api_kwargs` 在不同 api_mode 下形状不同：

**anthropic_messages 模式**：
```python
{
  "model": ...,
  "messages": [{"role": "user", ...}],   # ← 没有 role=="system" 的消息
  "system": "<整段系统提示词字符串>",      # ← system 在【顶层】
  "max_tokens": N, "tools": [...],
  "thinking": {...}, "temperature": 1,
}
```
**OpenAI 兼容模式**：system 是 `messages` 里第一条 `role=="system"` 消息。

⇒ **只扫 `messages` 里 `role=="system"` 的插件，在 Anthropic 模式下永远找不到目标**：
注册成功、日志正常、请求照发、效果为零。它只能退而往 `messages` 里塞一条 system 消息，
把内核和原系统提示**混排**，强度打折。

⇒ **正确做法**：优先改写**顶层 `request["system"]`**（替换 / 追加 / 剥离三种模式都要覆盖），
并保留 messages 回退路径给 OpenAI 兼容模式。

★ 顶层 system 的内容结构（Hermes 组装）：
```
<stable 段>
<context 段>
# Supermemory
...
<!-- hermes-plugin-sections:start -->   ← 插件段容器（在 memory 段与 runtime 段之间）
## Plugin Context: <id>
...
<!-- hermes-plugin-sections:end -->
# Hermes runtime environment
...
```
★ **切段重建 system 时，切分点要同时认 `\n## `（标题）与 `\n<!--`（容器起点）**——
只认 `\n## ` 会把容器起始标签留在上一段里，重建后残留半截容器。

## 三、思考参数：框架默认发的不一定被 provider 认

`agent/anthropic_adapter.py::_thinking_kwargs` 按模型名判断走哪条分支：

- 自适应族（Claude 4.6+、Kimi/Moonshot）→ `thinking.type=adaptive` + `output_config.effort`
- 其余（含未在适配器里登记的 provider）→ `thinking.type=enabled` + `budget_tokens`
  + `temperature=1` + 上抬 `max_tokens`

★ **未在适配器里登记的 provider 会落到 manual 分支**。若该 provider 只认
`output_config.effort`，则 `budget_tokens` 被静默忽略 → 思考烧光预算 → 正文 0 字。

⇒ **修法**：在中间件里按 provider 改写，不要改适配器：
```python
if is_target_provider(request, context):
    request = dict(request)
    request["thinking"] = {"type": "adaptive"}
    request["output_config"] = {"effort": "low"}
    request.pop("temperature", None)
```
★ **只在目标 provider 上生效**，否则会误伤其它路由（Claude 系对 `output_config` 的
支持面不同，Kimi 的 disable 语义也不同）。

## 四、中间件签名与 fail-soft

```python
def _llm_request(request: dict, **context) -> dict:
    try:
        ...
        return {"request": request}
    except Exception:
        logger.warning("...fail-soft", exc_info=True)
        return {"request": request}     # ★ 任何异常都原样放行，绝不打断会话
```

`context` 里可拿到：`session_id` / `model` / `provider` / `base_url` / `api_mode` /
`task_id` / `turn_id`。

★ **改写前先 `dict(request)` 拷贝**，不要就地改入参——就地改会污染同一次调用里
其它中间件看到的对象。

★ **不要在 messages 里插 `role="system"`**（Anthropic 端点虽能收下，但会与顶层
system 重复且改变消息角色序列）。要改 system 就改顶层。

## 五、部署与验证

```bash
hermes plugins enable <name>       # 热加载进活体 gateway，不需要 restart
hermes plugins list                # 状态 enabled
hermes plugins show <name>         # 声明的能力
```
★ `enable` 的输出会区分**立即生效**（如命令/中间件）与**下个会话生效**（系统提示词段）。
★ **活体有会话时 `hermes gateway restart` 会被拒**（会打断 N 个在跑的会话）——
用 enable/disable 热加载，别为了验证去重启。
★ 验证要**看真实流量日志**里中间件的命中记录，才证明它挂在了生产路径上。

## 六、自测（改插件必跑）

插件目录内放一个 `selftest.py`：不触网、不调模型，直接构造**宿主真实形状**的
api_kwargs，断言中间件改写结果。覆盖：
- 目标 provider 上 thinking/output_config 被改；**非目标 provider 上不被改**
- system 落在顶层而非 messages
- 旧插件段容器被剥离干净
- 各模式（替换/追加/剥离）行为正确
- 段字符数在预算内
- 畸形请求不抛异常（fail-soft）
