### 给 LLM Agent 的 Win11 SSH 避坑速查（已实战踩过）

**场景**：Termux（或任意 Linux 客户端）用 Ed25519 公钥登录一台 Win11，本地账号是本机管理员（`Administrators` 组），SSH 由 Windows OpenSSH 提供。

---

### 1. 最大坑：管理员账号不看用户目录的 `authorized_keys`

- 对 **管理员账号**，Win OpenSSH 默认配置是：

  ```text
  Match Group administrators
      AuthorizedKeysFile __PROGRAMDATA__/ssh/administrators_authorized_keys
  ```

- 这意味着：
  - `C:\Users\<user>\.ssh\authorized_keys` **不会被用来认证管理员登录**
  - 你往用户目录写再多 key，都会得到：
    - `Permission denied (publickey,password,keyboard-interactive).`
    - 然后继续要密码

**Agent 必须首先判断：**

```powershell
net localgroup administrators
```

如果目标账号在列表里，就要走 **ProgramData 那条路**，而不是只改用户目录。

---

### 2. 正确的管理员公钥文件与权限

**目标文件**：`C:\ProgramData\ssh\administrators_authorized_keys`

**要求：**

1. 内容：**只有纯粹的公钥行**，例如：

   ```text
   ssh-ed25519 AAAA... u0_a383@localhost
   ```

   - 不要带 PowerShell here-string 的 `@"` / `"@`
   - 不要带多余注释行（除非你确定不会破坏格式）

2. ACL 必须非常严格，否则 OpenSSH 会直接忽略该文件：

   ```powershell
   icacls "C:\ProgramData\ssh\administrators_authorized_keys" `
     /inheritance:r `
     /grant "*S-1-5-32-544:F" `
     /grant "SYSTEM:F"
   ```

   - `*S-1-5-32-544` 是 `Administrators` 组的 SID，避免本地化名字差异
   - 只有 `Administrators` + `SYSTEM` 拥有权限

---

### 3. Agent 在 Win11 上应该怎么干（管理员账号场景）

**前提**：公钥已经写进了用户目录的 `authorized_keys`（例如你之前用脚本加进去）。

**在提升权限的 PowerShell 中（必须 Admin）执行一条多行命令：**

```powershell
$authorizedKey = Get-Content -Path "$env:USERPROFILE\.ssh\authorized_keys"
New-Item -ItemType Directory -Force -Path "$env:ProgramData\ssh" | Out-Null
Set-Content -Force -Path "$env:ProgramData\ssh\administrators_authorized_keys" -Value $authorizedKey
icacls "$env:ProgramData\ssh\administrators_authorized_keys" /inheritance:r /grant "*S-1-5-32-544:F" /grant "SYSTEM:F"
Restart-Service sshd
```

**对 LLM Agent 的要点：**

- 你可以生成上面这段命令，但**无法替用户点击 UAC**，所以要明确提示：
  - “请在 *管理员* PowerShell 里执行这整段”
- 执行后再自动检查一次：

  ```powershell
  type C:\ProgramData\ssh\administrators_authorized_keys
  icacls C:\ProgramData\ssh\administrators_authorized_keys
  ```

  确认文件存在且 ACL 正确。

---

### 4. Termux / Linux 侧的常规事项（相对简单）

- 权限：

  ```bash
  mkdir -p ~/.ssh
  chmod 700 ~/.ssh
  chmod 600 ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub 2>/dev/null || true
  ```

- 测试纯公钥登录（必须禁止密码回退，否则会误判）：

  ```bash
  ssh -i ~/.ssh/id_ed25519 \
      -o PubkeyAuthentication=yes \
      -o PasswordAuthentication=no \
      y9_win11@100.72.70.79 "echo WIN_OK && hostname"
  ```

- 通过后再写 `~/.ssh/config`：

  ```bash
  Host win11
      HostName 100.72.70.79
      User y9_win11
      IdentityFile ~/.ssh/id_ed25519
  ```

---

### 5. 总结给 Agent 的“一句话原则”

- **如果目标是 Windows 管理员账号**：
  - **永远不要只改用户目录的 `authorized_keys`**
  - 必须改 `C:\ProgramData\ssh\administrators_authorized_keys`，并设置严格 ACL
  - 所有对该文件的操作，都需要通过“用户确认 UAC 的管理员 PowerShell”来完成  
- 否则，你会一直看到 `Permission denied (publickey,...)`，然后回退到密码登录，看起来像“key 完全没生效”。
