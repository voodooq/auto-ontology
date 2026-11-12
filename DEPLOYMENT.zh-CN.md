<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 在 Kubernetes 上部署 Auto-Ontology

> 本文档为 [`DEPLOYMENT.md`](./DEPLOYMENT.md) 的简体中文翻译版。

## 已发布制品（Released artifacts）

正式发布版本均托管在公开的 NGC Catalog `nvidia` 组织下：

| 制品 | 仓库地址 |
| --- | --- |
| 后端 + 摄取服务镜像 | `nvcr.io/nvidia/auto-ontology` |
| 前端（Next.js）镜像 | `nvcr.io/nvidia/auto-ontology-frontend` |
| MCP 服务器镜像 | `nvcr.io/nvidia/auto-ontology-mcp` |
| Helm Chart | 位于 `https://helm.ngc.nvidia.com/nvidia` 仓库中的 `auto-ontology`（同样收录于 NGC Catalog） |

每次发布都会使用 Chart 版本号作为镜像标签，并且 Chart 会默认将内部引用的镜像版本锁定为该标签。

## 部署发布版本

安装最新发布的版本步骤如下：

1. 添加并更新 NGC Helm 仓库：

   ```bash
   helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
   helm repo update
   ```

   执行 `helm search repo nvidia/auto-ontology --versions` 可列出所有可用发布版本。

2. 将敏感密钥写入单独的 values 文件。请直接在文本编辑器中创建该文件，而不要在命令行参数中明文传入，从而避免密钥留在 Shell 历史记录和进程参数中；同时使用 `chmod 600` 限制访问权限，并切勿提交至 Git：

   ```yaml
   # auto-ontology-secrets.yaml
   defaultModelsApiKey: <API-KEY>
   postgresPassword: <POSTGRES-PASSWORD>
   adminEmail: <ADMIN-EMAIL>
   adminPassword: <ADMIN-PASSWORD>
   connectionStrings: <CONNECTION-STRINGS>   # 可选；也可后续在 Web UI 中添加连接
   ```

3. 安装 Helm Chart：

   ```bash
   helm install auto-ontology nvidia/auto-ontology -f auto-ontology-secrets.yaml
   ```

   以上命令默认安装最新版本；如需固定版本，可添加 `--version <version>` 参数。通过 `helm list` 可查看当前已安装的 Chart 版本。

   `adminEmail` 与 `adminPassword` 为必填项：由于系统禁用了自助注册，这两个字段用于创建初始管理员账号，缺省时安装将会失败。如果用户将通过除下方 `http://localhost:3000`（端口转发地址）之外的域名或地址访问应用，请务必将 `appUrl` 设置为对应的精确 Origin。

4. 暴露 Web UI：

   ```bash
   kubectl port-forward svc/frontend 3000:3000
   ```
