# Canvas LMS IDP Token 自动刷新

通过模拟机构 IDP（CAS/SAML）登录流程，使用 RSA 加密凭据，自动刷新 Canvas LMS API Token。

[English](README.en.md)

## 概述

许多高校将 Canvas LMS 部署在机构 SSO（CAS/SAML）之后。通过 Canvas 设置页面手动生成的 API Token 可能每天或每次会话后被 SSO 层强制失效。本项目自动完成完整的登录链路，无需人工干预即可编程方式创建全新的 Canvas API Token。

## 工作原理

```
入口 URL → CAS/IDP（lck + authChainCode）→ RSA 加密密码 → authExecute
  → loginToken（JWT）→ authnEngine → SSO ticket → Canvas 会话 → 创建 API Token
  → （可选）按 purpose 删除旧 Token → 输出 NEW_TOKEN=xxx
```

## 快速开始

```bash
# 1. 创建虚拟环境
python3 -m venv .venv
source .venv/bin/activate  # 或：.venv\Scripts\activate（Windows）

# 2. 安装依赖
pip install -r requirements.txt

# 3. 配置凭据
cp .env.example .env
# 编辑 .env，填入 ELEARNING_USERNAME 和 ELEARNING_PASSWORD

# 4. 测试（dry-run —— 仅获取公钥，不执行登录）
python elearning_login.py --dry-run --debug

# 5. 完整流程：登录 → 创建 Token → 清理旧 Token
python elearning_login.py --cleanup-old-tokens
```

成功后：`NEW_TOKEN=<token值>` 将输出到标准输出。

## 配置说明

| 变量 | 是否必填 | 默认值 | 说明 |
|---|---|---|---|
| `ELEARNING_USERNAME` | ✅ | — | 学号/工号 |
| `ELEARNING_PASSWORD` | ✅ | — | 密码 |
| `ELEARNING_ENTRY_URL` | | `https://elearning.fudan.edu.cn/login/cas` | CAS 入口地址 |
| `ELEARNING_IDP_BASE_URL` | | `https://id.fudan.edu.cn` | IDP 基础 URL |
| `ELEARNING_ENTITY_ID` | | `https://elearning.fudan.edu.cn` | 服务提供方实体 ID |
| `ELEARNING_TOKEN_PURPOSE` | | `OpenClaw Auto Refresh Token` | 创建 Token 的标签 |
| `ELEARNING_CLEANUP_OLD_TOKENS` | | `0` | 自动删除同 purpose 的旧 Token |
| `ELEARNING_TIMEOUT_SECONDS` | | `20` | 请求超时时间（秒） |

## 命令行参数

```
--debug               开启详细 HTTP 日志，并将调试文件保存至 debug_output/
--dry-run             只测试到获取公钥为止（不执行实际登录）
--skip-token          登录后跳过 Token 创建
--cleanup-old-tokens  创建新 Token 后，删除同 purpose 的旧 Token
--cleanup-purpose     覆盖清理时匹配的 purpose
--cleanup-dry-run     预览将被删除的 Token，不实际删除
--dump-dir            自定义调试输出目录（默认：debug_output/）
```

## 与 Agent/工具集成

实现懒刷新（lazy-refresh）模式：

1. 从文件中读取已保存的 Token
2. 通过 `GET /api/v1/users/self` 校验 Token（仅检查 HTTP 状态码）
3. 返回 200 → 使用该 Token 进行 API 调用
4. 返回 401 → 运行本脚本，捕获 `NEW_TOKEN=` 并保存到文件，重试
5. 脚本失败 → 提醒用户（密码已更改、触发验证码等）

## 适配其他高校

本 IDP 流程基于中国许多高校通用的 CAS 模式：

1. 在 `.env` 中修改 URL（`ELEARNING_ENTRY_URL`、`ELEARNING_IDP_BASE_URL`、`ELEARNING_ENTITY_ID`）
2. 使用 `--dry-run --debug` 验证 lck/authChainCode 提取是否正常
3. 如认证方式不同：修改 `auth_session.py` 中的 `pick_auth_chain_code()`
4. 如加密方式不同：修改 `encrypt_password_rsa()` 和 `parse_public_key_payload()`

详细适配指南请参阅 [references/idp-adaptation.md](references/idp-adaptation.md)。

已测试：**复旦大学**（`id.fudan.edu.cn` → `elearning.fudan.edu.cn`）。

## 故障排查

使用 `--debug` 将调试产物保存至 `debug_output/`：

| 现象 | 检查项 |
|---|---|
| `未能从入口响应中解析到 lck` | `entry_response.html` —— 查找 `context_CAS_...` |
| `queryAuthMethods 未找到 userAndPwd` | `query_auth_methods.json` —— 检查 `moduleCode` |
| `未从 authExecute 提取到 loginToken` | `auth_execute.json` —— 检查 `code`/`message` |
| `未拿到关键会话 Cookie` | 检查是否触发验证码，核实凭据是否正确 |
| Token API 返回 401/422 | `cookies.txt` 中查找 `_csrf_token` / `_normandy_session` |
| 清理失败 | `cleanup_summary.json` 中查找 `failed` 条目 |

## 安全说明

- `.env` 包含凭据 —— **切勿提交到 git**
- `debug_output/` 可能包含会话数据 —— **分享前请清除敏感信息**
- 密码使用服务器提供的 RSA 公钥进行 PKCS1v1_5 加密
- Token 创建时带有 purpose 标签，便于安全的生命周期管理

## 已知限制

- **验证码/限流**：无法解决人机验证 —— 需要人工干预
- **MFA/2FA**：不支持（需要交互式流程）
- **IDP 接口变更**：若 JSON 字段或 HTML 结构发生变化，脚本可能失效
- **SAML 2.0（非 CAS）**：不直接支持

## 补充文档

- [项目背景与应用场景说明](docs/project-background.md) — 复旦 eLearning 平台架构背景、典型应用场景、技术选型说明

## 许可证

MIT

## 致谢

本项目为复旦大学 Canvas LMS Token 生命周期自动化管理而开发。
同时作为 [OpenClaw Skill](https://clawhub.com) 发布（`canvas-lms-idp-auto-refresh`）。
