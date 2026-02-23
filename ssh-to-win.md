## SSH to Win11（给 Agent 的避坑技能）

目标：让 **Termux/WSL/Linux → Windows 11** 的 SSH 登录稳定可用，尤其是 **公钥免密** 场景（常见：走 Tailscale）。

这份文档写给“会执行命令的 Agent”，核心是避免 Windows OpenSSH 的 Win-only 坑。

---

### 0. 先记住最关键结论（90% 卡在这里）

如果你用的是 **管理员账号**（属于 `Administrators` 组）登录 Windows：

- **不要只写** `C:\Users\<user>\.ssh\authorized_keys`
- Windows OpenSSH 默认真正看的往往是：
  - `C:\ProgramData\ssh\administrators_authorized_keys`
  - 且必须有严格 ACL（只允许 Administrators + SYSTEM）

否则客户端会无限看到：

```text
Permission denied (publickey,password,keyboard-interactive).
```

---

### 1. Windows 端：确保 sshd 正在跑

在 Windows PowerShell：

```powershell
Get-Service sshd
```

如果没有 `sshd`，先安装 OpenSSH Server（略）。

---

### 2. 客户端（Termux）端：准备密钥 + 只测公钥（禁用密码回退）

Termux：

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub 2>/dev/null || true

# 强制：只允许公钥，禁止密码回退，避免“其实用密码登上了还以为 key OK”
ssh -i ~/.ssh/id_ed25519 \
  -o PubkeyAuthentication=yes \
  -o PasswordAuthentication=no \
  y9_win11@100.72.70.79 "echo WIN_OK && hostname"
```

---

### 3. Windows 端：写入用户 authorized_keys（普通用户用这个）

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.ssh" | Out-Null
# 注意：authorized_keys 文件里只能是一行行的 ssh-ed25519/ssh-rsa 公钥，别把 PowerShell 的 @" / "@ 写进去
@"
ssh-ed25519 AAAA... your-comment
"@ | Set-Content -Path "$env:USERPROFILE\.ssh\authorized_keys"
```

快速验收：

```powershell
type "$env:USERPROFILE\.ssh\authorized_keys"
```

---

### 4. Win-only 关键：管理员账号必须配置 administrators_authorized_keys

先确认用户是否是管理员：

```powershell
net localgroup administrators
```

如果目标登录用户在列表里，就按下面做（**需要管理员 PowerShell / UAC**，Agent 不能替用户点 UAC）。

在 **PowerShell（管理员）** 粘贴：

```powershell
$authorizedKey = Get-Content -Path "$env:USERPROFILE\.ssh\authorized_keys"
New-Item -ItemType Directory -Force -Path "$env:ProgramData\ssh" | Out-Null
Set-Content -Force -Path "$env:ProgramData\ssh\administrators_authorized_keys" -Value $authorizedKey

# 用 SID 避免系统语言导致的组名不匹配：
# S-1-5-32-544 = Administrators
icacls "$env:ProgramData\ssh\administrators_authorized_keys" /inheritance:r /grant "*S-1-5-32-544:F" /grant "SYSTEM:F"

Restart-Service sshd
```

要点：
- **UAC**：这一步基本必须用户手动开“管理员 PowerShell”执行一次。
- **ACL**：不正确就当没配置，继续 Permission denied。

---

### 5. 最终验收（回到 Termux）

还是用“禁用密码回退”的命令：

```bash
ssh -i ~/.ssh/id_ed25519 \
  -o PubkeyAuthentication=yes \
  -o PasswordAuthentication=no \
  y9_win11@100.72.70.79 "echo WIN_OK && hostname"
```

看到 `WIN_OK` 即通过。

---

### 6. 体验优化：Termux 里加别名 host

```bash
cat >> ~/.ssh/config <<EOF

Host win11
  HostName 100.72.70.79
  User y9_win11
  IdentityFile ~/.ssh/id_ed25519
EOF

chmod 600 ~/.ssh/config
```

以后直接：

```bash
ssh win11
```

---

### 7. 给 Agent 的“别踩坑清单”

- 管理员账号：优先检查 `C:\ProgramData\ssh\administrators_authorized_keys`
- 不要把 `@"` / `"@` 这类 shell 语法写进 `authorized_keys`
- 任何“写 ProgramData + 改 ACL”都要 UAC：Agent 只能提示用户手动执行
- 验收必须禁用密码回退：`-o PasswordAuthentication=no`

