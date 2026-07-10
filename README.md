# gdrive-upload-action

使用 Google 用户 OAuth 授权（而非服务账号），将文件上传/覆盖到指定 Google Drive 文件夹的 GitHub Composite Action。纯 bash + curl 实现，不依赖任何现场构建的 Docker 镜像，避免了因基础镜像拉取失败导致的 CI 不稳定问题。

> 为什么不用服务账号？服务账号本身没有 Google Drive 存储配额，即使把文件夹共享给服务账号，新建文件时依然会报 `storageQuotaExceeded`。对**个人 Gmail 账号**（非 Google Workspace）而言，唯一可行的方式是用真实用户的 OAuth 授权（refresh_token）以该用户身份上传，文件计入该用户自己的 Drive 配额。

## 前置准备（一次性）

### 1. 创建 OAuth 客户端

1. 打开 [Google Cloud Console](https://console.cloud.google.com/apis/credentials)，选择/创建一个项目
2. 启用 **Google Drive API**（APIs & Services → Library → 搜索 Google Drive API → Enable）
3. 如果项目没有配置过 OAuth 同意屏幕（OAuth consent screen），先按提示配置一个，User Type 选 **External**，测试阶段把你自己的 Gmail 加入 Test users 即可，无需提交审核
4. Credentials → Create Credentials → OAuth client ID → Application type 选 **Desktop app**
5. 创建后得到 `Client ID` 和 `Client secret`

### 2. 用你的 Gmail 账号做一次性授权，换取 refresh_token

1. 把下面 URL 中的 `YOUR_CLIENT_ID` 替换成上一步的 Client ID，在浏览器中打开（用你要上传文件所属的那个 Gmail 账号登录）：

   ```
   https://accounts.google.com/o/oauth2/v2/auth?client_id=YOUR_CLIENT_ID&redirect_uri=http://localhost&response_type=code&scope=https://www.googleapis.com/auth/drive&access_type=offline&prompt=consent
   ```

2. 登录并同意授权后，浏览器会跳转到一个打不开的 `http://localhost/?code=XXXX&scope=...` 页面（这是预期的，本地并没有服务在监听），**从地址栏里复制 `code=` 后面的那一段值**（到下一个 `&` 为止，记得先做一次 URL 解码，比如 `%2F` 要还原成 `/`）

3. 用这个 code 换取 refresh_token（把 `YOUR_CLIENT_ID`、`YOUR_CLIENT_SECRET`、`YOUR_CODE` 替换成实际值）：

   ```bash
   curl -s -X POST https://oauth2.googleapis.com/token \
     --data-urlencode "client_id=YOUR_CLIENT_ID" \
     --data-urlencode "client_secret=YOUR_CLIENT_SECRET" \
     --data-urlencode "code=YOUR_CODE" \
     --data-urlencode "grant_type=authorization_code" \
     --data-urlencode "redirect_uri=http://localhost"
   ```

4. 返回的 JSON 里的 `refresh_token` 字段就是长期使用的凭证，**只在第一次授权（`prompt=consent`）时返回，务必保存好**

### 3. 存入调用方仓库的 Secrets

在**调用方仓库** Settings → Secrets and variables → Actions 里添加：

| Secret | 值 |
|---|---|
| `GDRIVE_OAUTH_CLIENT_ID` | 第 1 步的 Client ID |
| `GDRIVE_OAUTH_CLIENT_SECRET` | 第 1 步的 Client secret |
| `GDRIVE_OAUTH_REFRESH_TOKEN` | 第 2 步换到的 refresh_token |
| `GDRIVE_FOLDER_ID` | 目标文件夹 ID（从文件夹 URL 中获取，该文件夹需属于做授权的这个 Gmail 账号，或已被其共享为可编辑） |

## 用法

```yaml
- name: 上传 AAB 到 Google Drive
  id: gdrive_upload
  uses: starwaveai-dev/gdrive-upload-action@v1
  with:
    oauth_client_id: ${{ secrets.GDRIVE_OAUTH_CLIENT_ID }}
    oauth_client_secret: ${{ secrets.GDRIVE_OAUTH_CLIENT_SECRET }}
    oauth_refresh_token: ${{ secrets.GDRIVE_OAUTH_REFRESH_TOKEN }}
    folder_id: ${{ secrets.GDRIVE_FOLDER_ID }}
    file_path: build/app/outputs/bundle/release/app-release.aab
    overwrite: "true"
```

## 输入参数

| 参数 | 必填 | 默认值 | 说明 |
|---|---|---|---|
| `oauth_client_id` | 是 | - | Google OAuth 客户端 ID（Desktop app 类型） |
| `oauth_client_secret` | 是 | - | Google OAuth 客户端密钥 |
| `oauth_refresh_token` | 是 | - | 一次性授权换取的 refresh_token |
| `folder_id` | 是 | - | 目标 Google Drive 文件夹 ID |
| `file_path` | 是 | - | 要上传的本地文件路径 |
| `file_name` | 否 | `file_path` 的文件名 | 上传到 Drive 后使用的文件名 |
| `overwrite` | 否 | `true` | 文件夹中已存在同名文件时是否覆盖，否则会新建一个同名文件 |

## 输出参数

| 输出 | 说明 |
|---|---|
| `file_id` | 上传/更新后的 Google Drive 文件 ID |
| `web_view_link` | 文件在 Google Drive 中的查看链接 |

## 实现原理

1. 用 `refresh_token` + `client_id` + `client_secret` 向 `https://oauth2.googleapis.com/token` 换取短期 access_token（refresh_token 长期有效，除非被撤销）
2. `overwrite=true` 时，先按文件名在目标文件夹内搜索是否已存在同名文件
3. 用 `multipart/related` 请求上传（已存在则 `PATCH` 覆盖，否则 `POST` 新建），文件的存储配额计入做授权的这个 Google 账号

## 注意事项

- refresh_token 一旦被用户在 [Google 账号权限管理页](https://myaccount.google.com/permissions) 手动撤销，或超过 6 个月未使用，就会失效，需要重新走一遍授权流程
- OAuth 同意屏幕如果长期停留在 "Testing" 状态且超过 100 个 test users 限制会有问题，但对单一 CI 账号场景不受影响；如果担心过期策略，也可以后续把同意屏幕发布为 "In production"（无需 Google 审核，除非申请敏感/受限 scope）
