# gdrive-upload-action

使用 Google 服务账号，将文件上传/覆盖到指定 Google Drive 文件夹的 GitHub Composite Action。纯 bash + curl + openssl 实现，不依赖任何现场构建的 Docker 镜像，避免了因基础镜像拉取失败导致的 CI 不稳定问题。

## 前置准备

1. 在 Google Cloud Console 创建一个服务账号，并启用 **Google Drive API**
2. 为该服务账号生成一个 JSON 格式的密钥
3. 在目标 Google Drive 文件夹上，把该服务账号的邮箱（`client_email`）添加为"编辑者"协作者（否则服务账号无权限读写该文件夹）
4. 把以下两项存进**调用方仓库**的 Settings → Secrets and variables → Actions：
   - `GDRIVE_CREDENTIALS`：服务账号 JSON 的**原始内容**（不需要 base64 编码）
   - `GDRIVE_FOLDER_ID`：目标文件夹 ID（从文件夹 URL 中获取）

## 用法

```yaml
- name: 上传 AAB 到 Google Drive
  id: gdrive_upload
  uses: starwaveai-dev/gdrive-upload-action@v1
  with:
    credentials: ${{ secrets.GDRIVE_CREDENTIALS }}
    folder_id: ${{ secrets.GDRIVE_FOLDER_ID }}
    file_path: build/app/outputs/bundle/release/app-release.aab
    overwrite: "true"
```

## 输入参数

| 参数 | 必填 | 默认值 | 说明 |
|---|---|---|---|
| `credentials` | 是 | - | Google 服务账号 JSON 内容（原始 JSON 文本） |
| `folder_id` | 是 | - | 目标 Google Drive 文件夹 ID |
| `file_path` | 是 | - | 要上传的本地文件路径 |
| `file_name` | 否 | `file_path` 的文件名 | 上传到 Drive 后使用的文件名 |
| `overwrite` | 否 | `true` | 文件夹中已存在同名文件时是否覆盖，否则会新建一个同名文件 |

## 输出参数

| 输出 | 说明 |
|---|---|
| `file_id` | 上传/更新后的 Google Drive 文件 ID |
| `web_view_link` | 文件在 Google Drive 中的查看链接（需要该文件/文件夹已对目标用户开放访问权限） |

## 实现原理

1. 从服务账号 JSON 中取出 `client_email` 与 `private_key`
2. 构造 JWT（header + claims），用 `openssl dgst -sha256 -sign` 做 RS256 签名
3. 用签名后的 JWT 向 `https://oauth2.googleapis.com/token` 换取 access_token
4. `overwrite=true` 时，先按文件名在目标文件夹内搜索是否已存在同名文件
5. 用 `multipart/related` 请求上传（已存在则 `PATCH` 覆盖，否则 `POST` 新建）

## 注意事项

- 服务账号本身没有 Google Drive 存储配额，文件会计入**文件夹所有者**（或该文件夹所在共享云端硬盘）的配额，请确保目标文件夹已正确共享给服务账号且所有者有足够空间
- `web_view_link` 指向的文件默认仅服务账号与被显式授权的账号可访问；如需公开访问，需要额外设置该文件/文件夹的共享权限
