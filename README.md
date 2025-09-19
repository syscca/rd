# RustDesk 自定义修改和构建指南

本指南基于提供的步骤，帮助您Fork、修改和构建RustDesk的自定义版本，包括更改API服务器、删除广告提示、修改子模块配置等。操作基于GitHub和Git命令，确保您已安装Git和必要的开发环境。所有步骤假设您有GitHub账户。

## 1. Fork 项目和子模块

- **主项目 (rustdesk)**：
  - 访问 [https://github.com/rustdesk/rustdesk](https://github.com/rustdesk/rustdesk)。
  - 点击 **Fork**，生成您的副本（例如：https://github.com/您的用户名/rustdesk）。

- **子模块 (hbb_common)**：
  - 访问 [https://github.com/rustdesk/hbb_common](https://github.com/rustdesk/hbb_common)。
  - 点击 **Fork**，生成您的副本（例如：https://github.com/您的用户名/hbb_common）。

## 2. 克隆主项目并配置子模块

在终端中执行以下命令：

```bash
# 克隆您的主项目副本（替换为您的Fork URL）
git clone https://github.com/您的用户名/rustdesk.git
cd rustdesk

# 修改 .gitmodules 中的子模块 URL（指向您的 hbb_common Fork）
git config -f .gitmodules submodule.libs/hbb_common.url https://github.com/您的用户名/hbb_common

# 同步和更新子模块
git submodule sync
git submodule update --init --recursive

# 验证子模块 URL
git config -f .gitmodules --get submodule.libs/hbb_common.url
```

## 3. 修改 API 服务器地址

- 打开文件：`src/common.rs`。
- 在大约第1027行，替换 `get_api_server_` 函数的内容为以下代码（自定义API地址）：

```rust
fn get_api_server_(api: String, custom: String) -> String {
    #[cfg(windows)]
    if let Ok(lic) = crate::platform::windows::get_license_from_exe_name() {
        if !lic.api.is_empty() {
            return lic.api.clone();
        }
    }
    if !api.is_empty() {
        return api.to_owned();
    }
    let api = option_env!("API_SERVER").unwrap_or_default();
    if !api.is_empty() {
        return api.into();
    }
    let s0 = get_custom_rendezvous_server(custom);
    if !s0.is_empty() {
        let s = crate::increase_port(&s0, -2);
        if s == s0 {
            return format!("http://{}:{}", s, config::RENDEZVOUS_PORT - 2);
        } else {
            return format!("http://{}", s);
        }
    }
    "https://admin.你的域名.com".to_owned()
}
```

## 4. 删除客户端广告提示

- 打开文件：`flutter/lib/desktop/pages/connection_page.dart`。
- 将第81-110行的 `setupServerWidget` 函数替换为以下代码（移除广告提示）：

```dart
Widget setupServerWidget() => Flexible(
   child: Offstage(
     offstage: !(!_svcStopped.value &&
         stateGlobal.svcStatus.value == SvcStatus.ready &&
         _svcIsUsingPublicServer.value),
     child: Row(
       crossAxisAlignment: CrossAxisAlignment.center,
       children: [], 
     ),
   ),
 );
```

## 5. 修改子模块配置

- 进入子模块目录：

```bash
cd libs/hbb_common
```

- 打开文件：`src/config.rs`。
- 在大约第103行，修改常量为以下内容（自定义服务器和公钥）：

```rust
pub const RENDEZVOUS_SERVERS: &[&str] = &["rd.你的域名.com"];
pub const RS_PUB_KEY: &str = "你的KEY";
```

## 6. 提交子模块更改

在子模块目录（`libs/hbb_common`）中执行：

```bash
# 创建新分支（推荐）
git checkout -b fix-branch

# 添加并提交修改
git add src/config.rs
git commit -m "修改 config.rs"

# 推送分支到您的 hbb_common 仓库
git push origin fix-branch
```

## 7. 同步子模块到主项目

- 返回主项目目录：

```bash
cd ../..
```

- 更新主项目对子模块的引用：

```bash
git add libs/hbb_common
git commit -m "更新子模块 hbb_common 到修改后的版本"
```

## 8. 创建 Tag

- 切换到主分支（确保所有修改已提交）：

```bash
git checkout master  # 或您的目标分支
```

- 创建带附注的标签：

```bash
git tag -a v1.4.2 -m "Release 1.4.2 with custom fixes"
```

- 验证标签：

```bash
git show v1.4.2  # 检查是否包含子模块更新和文件修改
```

## 9. 推送所有更改到远程仓库

- 推送主项目代码和标签：

```bash
git push origin master
git push origin v1.4.2
```

- 推送子模块分支（确保已被主项目引用）：

```bash
cd libs/hbb_common
git push origin fix-branch
```

## 10. 开始编译

- 访问您的主项目 GitHub 页面。
- 在 **Actions** 页面，选择 **Flutter Nightly Build** 工作流。
- 在编译标签中选择 `v1.4.2`，然后运行构建。

## 附录：RustDesk 中继服务器部署（使用 Docker Compose）

使用以下 `docker-compose.yml` 文件部署中继服务器。确保替换占位符（如 `<relay_server>`）为实际值。

```yaml
networks:
  rustdesk-net:
    external: false
services:
  rustdesk:
    ports:
      - 21114:21114
      - 21115:21115
      - 21116:21116
      - 21116:21116/udp
      - 21117:21117
      - 21118:21118
      - 21119:21119
    image: lejianwen/rustdesk-server-s6:latest
    environment:
      - RELAY=<relay_server[:port]>
      - ENCRYPTED_ONLY=1
      - MUST_LOGIN=Y
      - TZ=Asia/Shanghai
      - RUSTDESK_API_RUSTDESK_ID_SERVER=<id_server[:21116]>
      - RUSTDESK_API_RUSTDESK_RELAY_SERVER=<relay_server[:21117]>
      - RUSTDESK_API_RUSTDESK_API_SERVER=http://<api_server[:21114]>
      - RUSTDESK_API_KEY_FILE=/data/id_ed25519.pub
      - RUSTDESK_API_JWT_KEY=xxxxxx # 自己随机一个字符串出来
    volumes:
      - ./data/rustdesk/server:/data
      - ./data/rustdesk/api:/app/data # 将数据库挂载
    networks:
      - rustdesk-net
    restart: unless-stopped
```

### 参数说明

- `RELAY=<relay_server[:port]>`：中继服务器地址和端口（默认21117）。
- `MUST_LOGIN=N`：默认为N，设置为Y则必须登录才能连接。
- `RUSTDESK_API_RUSTDESK_ID_SERVER=<id_server[:21116]>`：ID服务器地址。
- `RUSTDESK_API_RUSTDESK_RELAY_SERVER=<relay_server[:21117]>`：中继服务器地址。
- `RUSTDESK_API_RUSTDESK_API_SERVER=http://<api_server[:21114]>`：API服务器地址。
- `RUSTDESK_API_JWT_KEY=xxxxxx`：随便设置一个字符串作为JWT密钥。
- `/data/rustdesk/server`：用于查看密钥文件。

### 查看管理员密码

运行以下命令查看容器日志，查找 `[INFO] Admin Password Is:` 这行：

```bash
docker logs -f rustdesk-rustdesk-1
```
