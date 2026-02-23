## 在 Windows 11 上安装 / 启用 OpenSSH Server（给 Agent 的技能说明）

目标：当用户说“帮我装好 Win11 的 SSH，让别的机器能连进来”，Agent 知道该做哪些检查和命令，并且**不乱改系统配置**。

> 默认场景：Win11 桌面版，PowerShell 环境，用户有本机管理员权限，但 Agent 无法点击 UAC 弹窗。

---

### 1. 先检查：是不是已经装好了？

在 PowerShell 中执行（普通窗口即可）：

```powershell
Get-Service sshd -ErrorAction SilentlyContinue
```

判断：

- 如果看到类似：

  ```text
  Status   Name               DisplayName
  ------   ----               -----------
  Running  sshd               OpenSSH SSH Server
  ```

  表示 **已经安装并在运行**，本技能可以直接结束，进入“如何配置公钥登录”的技能（例如本 repo 的 `ssh-to-win.md`）。

- 如果没有任何输出，说明 **OpenSSH Server 尚未安装或服务名不存在**，继续下面步骤。

---

### 2. 优先使用系统内置的 OpenSSH Server 能力（Add-WindowsCapability）

> 注意：这一段需要在**管理员 PowerShell**里执行，Agent 需要提示用户“以管理员身份运行 PowerShell”后再粘贴命令。

1. 让用户打开 **PowerShell（管理员）**。
2. 执行安装命令：

   ```powershell
   Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
   ```

3. 安装完成后，启动并设置 sshd 开机自启：

   ```powershell
   Start-Service sshd
   Set-Service -Name sshd -StartupType Automatic
   ```

4. 验证：

   ```powershell
   Get-Service sshd
   ```

   看到 `Status: Running` 即 OK。

> 若 `Add-WindowsCapability` 提示需要提升权限，那说明用户没在管理员窗口里；Agent 要求用户重新开“以管理员身份运行”的 PowerShell 再执行。

---

### 3. 备用方案：用 winget 安装 Win32-OpenSSH（Preview 包）

在某些 Win11 环境（或 Windows 版本）里，内置功能不可用 / 报错时，可以退而求其次使用 `winget` 安装 Win32-OpenSSH 包。

> 这一步不一定需要管理员，但安装过程中通常仍会弹 UAC，Agent 无法代点，只能提示用户确认。

在 PowerShell 中执行：

```powershell
winget install Microsoft.OpenSSH.Preview --accept-package-agreements --accept-source-agreements
```

安装完成后，再次检查服务：

```powershell
Get-Service sshd
```

如果服务存在但未启动：

```powershell
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

此时再 `Get-Service sshd` 确认为 `Running` 即表示安装 + 启动成功。

---

### 4. 防火墙 / 端口快速自检

默认情况下，安装 OpenSSH Server 时会自动添加防火墙规则。Agent 可做最小验证：

1. 在本机 PowerShell：

   ```powershell
   netstat -an | findstr ":22"
   ```

   如果看到：

   ```text
   TCP    0.0.0.0:22   0.0.0.0:0   LISTENING
   TCP    [::]:22      [::]:0      LISTENING
   ```

   说明 sshd 已经在 22 端口监听。

2. 若用户使用的是第三方防火墙（火绒、360 等），Agent 只能提示用户：

   - “请在防火墙里允许 `sshd.exe` / `OpenSSH SSH Server` 的入站连接（TCP 22）”

---

### 5. 连接测试（从另一台机器）

安装完成后，可以让用户从另一台机器（如 Termux、WSL、Linux）测试密码登录：

```bash
ssh y9_win11@<win11-ip>
```

例如在配合 Tailscale 时，用 Tailscale IP：

```bash
ssh y9_win11@100.72.70.79
```

如果密码登录正常，则说明：

- OpenSSH Server 安装成功
- 服务在运行
- 端口 / 防火墙没有拦截

后续可以交给 `ssh-to-win.md` 这个技能处理**公钥免密登录**的配置。

---

### 6. Agent 行为要点（总结）

1. **永远先查再装**：
   - 通过 `Get-Service sshd` 判断是否已经有可用的 SSH Server，避免重复安装 / 破坏现有配置。

2. **区分用户权限场景**：
   - 需要 `Add-WindowsCapability` 或修改服务启动类型时，必须提醒用户“以管理员身份运行 PowerShell”，Agent 无法绕过 UAC。

3. **尽量使用系统内置 OpenSSH**：
   - 优先尝试 `OpenSSH.Server~~~~0.0.1.0`；只有在失败时再建议使用 `winget` 的 Win32-OpenSSH 包。

4. **安装后一定要显式启动 + 设置自动启动**：
   - `Start-Service sshd` + `Set-Service -Name sshd -StartupType Automatic`

5. **把安装步骤和公钥配置步骤拆开**：
   - 本技能只负责“让 Win11 有一个运行中的 sshd，可以接收密码登录”；
   - 公钥免密、管理员账号 key 的各种坑交给 `ssh-to-win.md` 处理。

