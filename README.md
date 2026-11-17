# Docker Hub to Alibaba Cloud ACR

[简体中文](#简体中文) | [English](#english)

---

## 简体中文

将 `images.txt` 中列出的容器镜像手动同步到阿里云容器镜像服务（ACR）个人版。

### 功能

- **手动触发**：通过 GitHub Actions 的 `workflow_dispatch` 启动；不配置定时任务。
- **架构选择**：可选择 `linux/amd64`（默认）或 `linux/arm64`。
- **并行同步**：最多同时处理 4 个镜像。
- **逐镜像状态**：每个镜像独立运行，单个镜像失败不会取消其他镜像任务。
- **目标格式**：使用 Skopeo 将选定平台的镜像复制为 Docker V2 Schema 2 格式。
- **自动命名**：优先使用镜像路径的最后一级名称；发生重名时逐级添加上级路径，确保目标仓库名称唯一。

> 本工作流每次只复制所选的一个 CPU 架构，不会生成包含 amd64 和 arm64 的多架构 Manifest。需要另一种架构时，请再次手动运行并选择对应架构。

### 开始使用

1. 在仓库的 `images.txt` 中，每行填写一个镜像引用，例如：

   ```text
   henrygd/beszel:latest
   ghcr.io/fish2018/pansou-web:latest
   portainer/portainer-ce:latest
   ```

2. 在 GitHub 仓库打开 **Settings → Secrets and variables → Actions**，添加以下 Repository secrets：

   | Secret | 说明 |
   |---|---|
   | `ACR_REGISTRY` | ACR 实例的 Registry 地址或主机名 |
   | `ACR_NAMESPACE` | ACR 命名空间 |
   | `ACR_USERNAME` | ACR 登录用户名 |
   | `ACR_PASSWORD` | ACR 登录密码 |

   可选：配置 Docker Hub 登录凭据，以降低匿名拉取限制：

   | Secret | 说明 |
   |---|---|
   | `DOCKERHUB_USERNAME` | Docker Hub 用户名 |
   | `DOCKERHUB_TOKEN` | Docker Hub Access Token |

   `ACR_REGISTRY` 可以填写控制台提供的主机名，也可以填写带 `https://` 的地址。请以 ACR 控制台显示的地址为准。不要将密码或 Token 写入仓库文件。

3. 打开仓库的 **Actions** 页面，选择 **Sync container images to Alibaba Cloud ACR**，点击 **Run workflow**。
4. 选择目标架构（`amd64` 或 `arm64`），然后启动工作流。
5. 在 Actions 运行详情中查看各个镜像任务的状态和日志。

### 目标仓库命名规则

目标路径格式：

```text
<ACR_REGISTRY>/<ACR_NAMESPACE>/<目标仓库名>:<tag>
```

命名按以下顺序解析：

1. 首先只取源镜像路径的最后一级名称。
2. 如果多个不同源镜像得到相同名称，则为冲突项逐级增加上级路径。
3. 如果路径扩展后仍有极少数名称冲突（例如规范化后相同），才使用短哈希作为最终区分。

例如：

| 源镜像 | 无冲突时的目标仓库名 |
|---|---|
| `henrygd/beszel:latest` | `beszel` |
| `bililive/recorder:latest` | `recorder` |
| `cloudnas/clouddrive2:latest` | `clouddrive2` |
| `ghcr.io/fish2018/pansou-web:latest` | `pansou-web` |
| `portainer/portainer-ce:latest` | `portainer-ce` |

如果以后同时加入 `team-a/nginx:latest` 和 `team-b/nginx:latest`，目标仓库名会从原本的 `nginx` 自动扩展为 `team-a-nginx` 和 `team-b-nginx`。

命名由每次运行时的完整镜像清单动态计算。修改清单后，冲突名称可能随之变化；若目标仓库名发生变化，旧名称对应的 ACR 仓库不会自动删除。

### 修改镜像清单

- **添加镜像**：在 `images.txt` 末尾增加一行，下次运行自动纳入。
- **删除镜像**：删除对应行，下次运行不再同步该镜像。
- **修改 Tag**：修改镜像引用中的 Tag，下次运行同步新 Tag。
- **清单数量**：无需修改工作流；任务矩阵会根据文件内容自动生成。

从清单删除镜像不会删除 ACR 中已经存在的镜像或仓库。如需清理，请在确认不再需要后到 ACR 控制台手动删除。

### 重要说明

- 本仓库只提供手动触发，不会自动定时同步。
- 每个镜像任务有独立日志；部分镜像失败时，可在 Actions 中查看具体错误。
- ACR 个人版对镜像格式和功能有自身限制；遇到目标端不支持的镜像格式时，任务仍可能失败。
- GHCR 等源仓库中的私有镜像需要相应的源仓库认证配置；当前工作流只提供可选的 Docker Hub 登录配置。
- 请确保所选架构与实际运行容器的设备架构匹配。

### 项目文件

- `images.txt`：待同步的镜像清单。
- `.github/workflows/sync.yml`：GitHub Actions 同步工作流。

---

## English

Manually copy the container images listed in `images.txt` to Alibaba Cloud Container Registry (ACR) Personal Edition.

### Features

- **Manual execution only**: Start the workflow with GitHub Actions `workflow_dispatch`. No scheduled sync is configured.
- **Architecture selection**: Choose `linux/amd64` (default) or `linux/arm64`.
- **Parallel jobs**: Up to four images are processed concurrently.
- **Independent image jobs**: A failure in one image job does not cancel the other jobs.
- **Target format**: Skopeo copies the selected platform using Docker V2 Schema 2 manifest format.
- **Automatic naming**: Use the final path component by default. If names collide, parent path components are added until the repository names are unique.

> Each run copies only one selected CPU architecture. It does not create a multi-platform manifest containing both amd64 and arm64. Run the workflow again with the other architecture when needed.

### Getting started

1. Add one image reference per line in `images.txt`, for example:

   ```text
   henrygd/beszel:latest
   ghcr.io/fish2018/pansou-web:latest
   portainer/portainer-ce:latest
   ```

2. In the GitHub repository, open **Settings → Secrets and variables → Actions** and add these repository secrets:

   | Secret | Description |
   |---|---|
   | `ACR_REGISTRY` | ACR registry hostname or URL |
   | `ACR_NAMESPACE` | ACR namespace |
   | `ACR_USERNAME` | ACR login username |
   | `ACR_PASSWORD` | ACR login password |

   Optional Docker Hub credentials can help reduce anonymous pull limits:

   | Secret | Description |
   |---|---|
   | `DOCKERHUB_USERNAME` | Docker Hub username |
   | `DOCKERHUB_TOKEN` | Docker Hub access token |

   Use the registry address shown in the ACR console. `ACR_REGISTRY` accepts a hostname or an address prefixed with `https://`. Never commit passwords or tokens to the repository.

3. Open the repository's **Actions** tab, select **Sync container images to Alibaba Cloud ACR**, and click **Run workflow**.
4. Select the target architecture (`amd64` or `arm64`) and start the run.
5. Review each image job and its logs on the Actions run page.

### Target repository naming

The destination path has this format:

```text
<ACR_REGISTRY>/<ACR_NAMESPACE>/<target-repository>:<tag>
```

Names are resolved in this order:

1. Use only the final component of the source image path.
2. If different source images produce the same name, add parent path components to the colliding names.
3. Use a short hash only as a last-resort disambiguator for rare collisions that remain after path expansion (for example, after name normalization).

Examples when there is no collision:

| Source image | Target repository |
|---|---|
| `henrygd/beszel:latest` | `beszel` |
| `bililive/recorder:latest` | `recorder` |
| `cloudnas/clouddrive2:latest` | `clouddrive2` |
| `ghcr.io/fish2018/pansou-web:latest` | `pansou-web` |
| `portainer/portainer-ce:latest` | `portainer-ce` |

For example, if both `team-a/nginx:latest` and `team-b/nginx:latest` are listed, their target repository names are automatically expanded to `team-a-nginx` and `team-b-nginx`.

Names are recalculated from the complete image list on every run. Changing the list may therefore change names for colliding images. ACR repositories using old names are not deleted automatically.

### Editing the image list

- **Add an image**: append a line to `images.txt`. It is included in the next run.
- **Remove an image**: remove its line. It is no longer synchronized in subsequent runs.
- **Change a tag**: edit the tag in the image reference. The new tag is copied on the next run.
- **Change the number of images**: no workflow edit is required; the job matrix is generated from the file.

Removing an image from the list does not delete the image or repository already stored in ACR. Delete old content manually in the ACR console after confirming it is no longer needed.

### Notes and limitations

- This repository supports manual runs only; no scheduled sync is configured.
- Each image has its own job and logs. Check the Actions run for image-specific errors.
- ACR Personal Edition has its own image-format and feature limitations. A source image may still fail if its content or manifest is unsupported by the destination.
- Private images in GHCR or other source registries require appropriate source-registry authentication. The current workflow provides optional Docker Hub authentication only.
- Select an architecture that matches the devices where you intend to run the containers.

### Project files

- `images.txt`: list of source images to synchronize.
- `.github/workflows/sync.yml`: GitHub Actions workflow.
