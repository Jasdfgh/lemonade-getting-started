# Lemonade 零基础部署

在 AMD Radeon GPU 上从零运行本地 AI — 中文实战教程。

## 内容

- [lemonade-getting-started.md](lemonade-getting-started.md) — 完整 Playbook（Markdown 格式）

## 环境

| 项目 | 值 |
|------|-----|
| 平台 | AMD Radeon Cloud (radeon.anruicloud.com) |
| GPU | AMD Radeon RX 7900 XTX (gfx1100, RDNA3, 24GB) |
| 镜像 | AMD OneClick Base (rocm7.2.1-py3.12) |
| Lemonade | v10.7.0 embeddable |
| 模型 | Gemma-4-E2B-it-GGUF (Q4_K_M, ~3.8GB) |

## 使用方法

1. 在 radeon.anruicloud.com 创建 Workspace（选 AMD OneClick Base 镜像）
2. 打开 Terminal
3. 按照 `lemonade-getting-started.md` 中的步骤从第一步开始执行

## 难度

⭐ （1 星，零基础）
