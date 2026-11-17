# Docker Hub to Alibaba Cloud ACR

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

