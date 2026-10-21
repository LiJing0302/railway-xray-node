# Railway 变成 xray 代理节点（Skill）

一个可复用的 AI Skill：把 Railway 云主机部署成一个 xray 代理节点。

- xray 提供 **VMess**（给 Surge）与 **VLESS** 两条 WebSocket 通道
- Node 前门负责发 Surge 托管订阅、发 VLESS 链接、根路径 400 伪装，并把 WS 流量反代给本机 xray
- 两条部署路线：**CLI 直传**（设备码登录，代码不经 GitHub）与 **GitHub 集成**
- 内含：两把钥匙的生成、5 个部署文件全文、本地测试、验收清单、免费换区（新加坡/美西/美东/阿姆斯特丹）、Surge / Clash / v2rayNG 导入对照、费用与生命周期（$5 试用 → Free $1/月）说明、故障排查速查

## 文件

- [`SKILL.md`](SKILL.md) — 技能正文（完整方法）
- [`skill.yaml`](skill.yaml) — 技能元数据

## 怎么用这个 Skill

把 `SKILL.md` 交给你的 AI 助手（或放入其 skills 目录），然后说一句「按这个 skill 帮我在 Railway 上搭一个 xray 节点」，它就会照着步骤执行：生成钥匙 → 写部署文件 → 登录 Railway → 部署 → 验收 → 给你各客户端的导入链接。

## 架构速览

```
客户端 ──TLS/WS──> Node 前门（对外，监听 PORT）
                    ├─ /sub-<口令>   → Surge 订阅
                    ├─ /vless       → VLESS 链接
                    ├─ /tk-rail     → 反代 xray VMess (127.0.0.1:20000)
                    ├─ /tk-vless    → 反代 xray VLESS (127.0.0.1:20001)
                    └─ 其余路径     → 400 Bad Request（伪装）
```

详细步骤、命令与注意事项见 [`SKILL.md`](SKILL.md)。
