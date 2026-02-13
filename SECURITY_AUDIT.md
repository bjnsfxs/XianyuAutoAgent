# 安全审计报告（后门/漏洞/危险代码）

## 审计范围
- `main.py`
- `XianyuApis.py`
- `XianyuAgent.py`
- `context_manager.py`
- `utils/xianyu_utils.py`
- `requirements.txt` / `.env.example` / `Dockerfile`

## 结论（先给结果）
- **未发现明确后门行为**（例如远程命令执行、隐蔽回连、恶意下载执行、动态`eval/exec`代码注入）。
- 发现若干**中低风险安全问题**，主要集中在：
  1. 凭据处理与落盘（Cookie写入`.env`）；
  2. 输入体积缺乏限制导致潜在DoS；
  3. 对外部数据（消息/商品描述）缺乏边界校验；
  4. 供应链与运行时加固不足。

## 重点发现

### 1) 凭据落盘与格式注入风险（中风险）
- 代码会把会话Cookie自动写回`.env`，且通过正则整行替换：`COOKIES_STR=.*`。若Cookie中出现异常字符（如换行），存在污染配置文件结构的风险，并增加凭据泄露面。
- 位置：`XianyuApis.py` 的 `update_env_cookies()`。

**建议**
- 不自动落盘敏感Cookie（默认仅内存存活）；
- 若必须落盘，使用严格转义（如JSON/base64封装）并限制字符集；
- 将`.env`权限收紧为`600`。

### 2) 解码流程无大小限制，可能触发资源耗尽（中风险）
- 对外部消息进行 `base64.b64decode` 和自定义MessagePack解码时，未做最大长度校验；异常大包可导致CPU/内存消耗。
- 位置：`main.py` 中 `handle_message()` 的解码流程；`utils/xianyu_utils.py` 的 `decrypt()`。

**建议**
- 对 `sync_data["data"]` 增加长度上限（例如1MB）；
- 在 `decrypt()` 前做大小检查，超过阈值直接丢弃并记录摘要日志；
- 给解析流程增加超时/快速失败策略。

### 3) 日志中包含用户消息与业务内容（中风险，隐私合规）
- 日志会输出用户ID、会话ID和消息内容，若日志外泄会造成隐私数据暴露。
- 位置：`main.py` 中多处 `logger.info(...)`。

**建议**
- 默认降低日志级别到`INFO`以下的敏感字段脱敏；
- 对 user_id/chat_id/message 做部分掩码；
- 生产环境启用日志保留与访问控制策略。

### 4) 供应链与依赖完整性缺失（中风险）
- `requirements.txt`虽固定版本，但未启用哈希校验（`--require-hashes`），无法防止镜像源污染/投毒。

**建议**
- 生成带哈希锁定文件（如`pip-compile --generate-hashes`）；
- 在CI中加入依赖漏洞扫描（`pip-audit`/`safety`）。

### 5) 交互式凭据输入与退出路径（低风险）
- 程序在某些失败场景会要求手工输入Cookie并 `sys.exit(1)`；对守护进程友好性较差，可能被异常消息触发不可用（可用性风险）。
- 位置：`XianyuApis.py:get_token()`、`main.py:check_and_complete_env()`。

**建议**
- 非交互模式下禁用stdin输入，改为错误码+告警；
- 用指数退避和熔断代替递归重试+退出。

## 明确“未发现”的高危项
- 未发现 `eval/exec` 动态执行。
- 未发现 `subprocess/os.system` 命令执行链。
- 未发现反序列化RCE（如`pickle.loads`）。
- 未发现硬编码外传端点（webhook/c2）与数据回传后门逻辑。

## 修复优先级建议
1. **P1**：限制外部消息体大小 + 解码防护（DoS风险）。
2. **P1**：停止自动明文落盘Cookie或加密/转义落盘。
3. **P2**：日志脱敏与最小化。
4. **P2**：依赖哈希锁定与漏洞扫描接入CI。

## 备注
- 受环境限制，本次未能在线安装第三方安全扫描器；结论基于静态代码审阅与模式检查。
