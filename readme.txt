# GAN

## 服务器无交互场景获取 GitHub Token

在服务器没有浏览器、不能弹出登录页面、也不适合人工交互输入验证码的场景下，**不要直接在服务器上执行交互式登录**。推荐做法是：

1. 在一台可正常访问 GitHub 的电脑上创建 Token
2. 通过安全方式下发到服务器
3. 在服务器上使用环境变量或密钥文件读取 Token

下面给出两种常见方案。

---

## 方案一：使用 Fine-grained Personal Access Token（最简单）

适用场景：

- 个人服务器
- 小规模自动化任务
- 只需要访问少量仓库

### 操作步骤

#### 第 1 步：在有浏览器的电脑上登录 GitHub

访问：

`https://github.com/settings/tokens?type=beta`

如果页面跳转，也可以按下面路径进入：

`GitHub -> Settings -> Developer settings -> Personal access tokens -> Fine-grained tokens`

#### 第 2 步：创建新 Token

点击 **Generate new token**，然后按需填写：

- **Token name**：例如 `server-deploy-token`
- **Expiration**：建议设置有效期，例如 30 天、90 天
- **Resource owner**：选择个人账号或组织
- **Repository access**：选择 `Only select repositories`
- **Permissions**：按最小权限原则分配

常见权限示例：

- 只读代码：`Contents: Read-only`
- 拉取/推送代码：`Contents: Read and write`
- 读取 Actions：`Actions: Read-only`
- 发布包：按需开启 `Packages` 权限

> 建议只开必须权限，不要直接给全部仓库和全部权限。

#### 第 3 步：生成并复制 Token

点击 **Generate token** 后，GitHub 只会显示一次完整 Token。

需要立即保存到安全位置，例如：

- 企业密钥管理系统
- 服务器密文配置
- 本地密码管理器

不要把 Token：

- 提交到代码仓库
- 写进 README 示例中的真实值
- 直接明文写入脚本并提交

#### 第 4 步：安全下发到服务器

推荐方式：

- 写入服务器环境变量
- 放入 CI/CD Secret
- 放入 `systemd` 的环境文件
- 放入云厂商 Secret Manager / Vault

例如，在 Linux 服务器临时设置：

```bash
export GITHUB_TOKEN="YOUR_TOKEN_HERE"
```

如果前一步已经设置好 `GITHUB_TOKEN`，使用 `gh` 命令时也可继续设置：

```bash
export GH_TOKEN="$GITHUB_TOKEN"
```

如果希望服务启动时自动加载，可写入仅管理员可读的**专用**环境文件，例如：

```bash
sudo touch /etc/github-token.env
sudo chmod 600 /etc/github-token.env
sudo sh -c 'echo GITHUB_TOKEN="YOUR_TOKEN_HERE" > /etc/github-token.env'
```

上面示例假设 `/etc/github-token.env` 专门用于保存这个 Token。

然后在 `systemd` 服务中引用：

```ini
[Service]
EnvironmentFile=/etc/github-token.env
```

#### 第 5 步：在服务器上验证 Token 是否可用

```bash
curl -H "Authorization: Bearer $GITHUB_TOKEN" \
     -H "Accept: application/vnd.github+json" \
     https://api.github.com/user
```

如果返回当前账号信息，说明 Token 可用。

#### 第 6 步：在常见工具中使用 Token

**1）调用 GitHub API**

```bash
curl -H "Authorization: Bearer $GITHUB_TOKEN" \
     -H "Accept: application/vnd.github+json" \
     https://api.github.com/repos/OWNER/REPO
```

**2）使用 Git 克隆私有仓库**

如果对安全要求更高，建议使用权限受限的凭据文件，而不是把 Token 直接放进命令参数。示例：

```bash
cat > ~/.netrc <<EOF
machine github.com
login x-access-token
password ${GITHUB_TOKEN}
EOF
chmod 600 ~/.netrc
git clone https://github.com/OWNER/REPO.git
```

完成后如无需长期保留，可及时删除 `~/.netrc` 或改用专门的密钥管理系统。

**3）使用 GitHub CLI**

```bash
export GH_TOKEN="$GITHUB_TOKEN"
gh auth status
```

---

## 方案二：使用 GitHub App（更适合长期无人值守）

适用场景：

- 生产服务器长期运行
- 需要定期自动换取短期 Token
- 多仓库、多组织自动化
- 希望权限更细、审计更清晰

### 操作步骤

#### 第 1 步：创建 GitHub App

在浏览器访问：

`GitHub -> Settings -> Developer settings -> GitHub Apps -> New GitHub App`

填写：

- **GitHub App name**
- **Homepage URL**
- **Webhook**：如果暂时不用，可先关闭
- **Permissions**：按需授予仓库权限

#### 第 2 步：安装 GitHub App

创建完成后，点击 **Install App**，安装到目标账号或组织，并选择可访问的仓库。

#### 第 3 步：下载私钥

在 GitHub App 页面生成并下载私钥文件（`.pem`）。

将以下信息安全保存到服务器：

- `APP_ID`
- `INSTALLATION_ID`
- 私钥文件内容

#### 第 4 步：服务器上动态换取 Installation Token

GitHub App 不是直接长期使用固定 Token，而是：

1. 使用私钥生成 JWT
2. 用 JWT 调用 GitHub API
3. 换取短期有效的 Installation Access Token

这种方式更适合无人值守服务，因为短期 Token 过期后可以再次自动申请。

---

## 选择建议

- **只想快速完成服务器认证**：优先使用 **Fine-grained PAT**
- **要做长期自动化、生产部署、定期轮换**：优先使用 **GitHub App**

---

## 安全建议

1. 只授予必须的最小权限
2. 给 Token 设置过期时间
3. 不要把 Token 提交到仓库
4. 不要把 Token 写入日志
5. 优先使用环境变量、密钥管理系统或受限权限文件
6. 定期轮换 Token
7. 人员变更或服务器下线时立即废弃旧 Token
