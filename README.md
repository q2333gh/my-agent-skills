## Guide: Making SSH Key Auth Work to Windows 11 (For LLM Agents)

This note is for agents that need to help users get **Termux → Windows 11** SSH key auth working. Windows OpenSSH has several **Windows-only gotchas** that differ from Linux; this guide is the “avoid-all-the-pain” path.

---

### 1. Environment Assumptions

- Windows 11 host with:
  - `OpenSSH SSH Server` installed and running (`sshd`)
  - Tailscale installed (optional but common in the setup)
- Client is Termux on Android:
  - Has `openssh` installed
  - Has an Ed25519 keypair at `~/.ssh/id_ed25519` (or you can generate it)

You, the agent, can run commands on Windows via a shell tool, but **cannot interact with UAC dialogs** and **cannot type passwords into TTY prompts**.

---

### 2. Understand the Big Windows-Specific Trap

On Linux, for *any* user you can just put the key in:

- `~/.ssh/authorized_keys`

On Windows OpenSSH:

- For a **normal (non-admin) user**:
  - Keys live in: `C:\Users\<user>\.ssh\authorized_keys`
- For an **administrator** (user is in `Administrators` group), by default the server really expects:
  - `C:\ProgramData\ssh\administrators_authorized_keys`
  - With very strict ACLs (only `Administrators` + `SYSTEM`)

In this session, the user is logging in as **`y9_win11`**, which **is an Administrator**:

```text
net localgroup administrators
  ...
  Administrator
  y9_win11
```

So:

- Putting the Termux public key into `C:\Users\y9_win11\.ssh\authorized_keys` is **necessary but not sufficient**.
- You must also handle the **`administrators_authorized_keys`** file with correct ACL.

If you skip this, you get:

```text
Permission denied (publickey,password,keyboard-interactive).
```

even though the key is present.

---

### 3. Termux Side: Key and Permissions

On Termux, normalize permissions and test pure key auth (no password fallback):

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub 2>/dev/null || true

echo "[Termux] testing key-based SSH to Windows..."
ssh -i ~/.ssh/id_ed25519 \
    -o PubkeyAuthentication=yes \
    -o PasswordAuthentication=no \
    y9_win11@100.72.70.79 "echo WIN_OK && hostname"
```

Interpretation:

- If it prints `WIN_OK` and the hostname (e.g. `btwly9w11`), key auth works.
- If it still says `Permission denied (publickey,...)`, Windows hasn’t accepted the key yet.

Later, once it works, add a host alias in Termux:

```bash
cat >> ~/.ssh/config <<EOF

Host win11
    HostName 100.72.70.79
    User y9_win11
    IdentityFile ~/.ssh/id_ed25519
EOF

chmod 600 ~/.ssh/config
```

Then the user can just run:

```bash
ssh win11
```

---

### 4. Windows Side: User-Level authorized_keys

First, write the Termux public key into the user’s `authorized_keys`. In this session the user explicitly provided the key:

```powershell
@"
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICZx7IaRcvFLD7KA4o2t+PFkjQCbb+7h3/KgVP+qjtLr u0_a383@localhost
"@ | Set-Content -Path "$env:USERPROFILE\.ssh\authorized_keys"
```

Important points for agents:

- **Do not** write the PowerShell here-string markers (`@"` / `"@`) into `authorized_keys`.  
  The *file content* must be just the `ssh-ed25519 ...` line.
- In this session we verified the file content via:

```powershell
type "$env:USERPROFILE\.ssh\authorized_keys"
```

and it correctly contained just:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICZx7IaRcvFLD7KA4o2t+PFkjQCbb+7h3/KgVP+qjtLr u0_a383@localhost
```

We also checked ACLs:

```powershell
icacls "$env:USERPROFILE\.ssh\authorized_keys"
```

which showed:

```text
NT AUTHORITY\SYSTEM:(I)(F)
BUILTIN\Administrators:(I)(F)
Y9WIN11\y9_win11:(I)(F)
```

This is acceptable.

---

### 5. The Real Fix: administrators_authorized_keys for Admin Users

Because `y9_win11` is an Administrator, OpenSSH does **not** actually use the user profile `authorized_keys` for admin sessions by default. Instead it expects:

- File: `C:\ProgramData\ssh\administrators_authorized_keys`
- ACL: only `Administrators` and `SYSTEM` with FullControl

As an agent, you **cannot bypass UAC**, so you cannot fully automate this part. You must:

1. Ask the user to open an **elevated** PowerShell (Run as Administrator).
2. Provide them a **single block of commands** to paste.

The working one-liner-block from this session:

```powershell
$authorizedKey = Get-Content -Path "$env:USERPROFILE\.ssh\authorized_keys"
New-Item -ItemType Directory -Force -Path "$env:ProgramData\ssh" | Out-Null
Set-Content -Force -Path "$env:ProgramData\ssh\administrators_authorized_keys" -Value $authorizedKey
icacls "$env:ProgramData\ssh\administrators_authorized_keys" /inheritance:r /grant "*S-1-5-32-544:F" /grant "SYSTEM:F"
Restart-Service sshd
```

Notes:

- `*S-1-5-32-544` is the SID for the `Administrators` group; using the SID avoids localization issues.
- `Restart-Service sshd` is needed so the server picks up the new file.
- After running this in an elevated shell, the file may not be visible to a non-elevated process in some security contexts, but OpenSSH can see it.

After this, rerunning on Termux:

```bash
ssh -i ~/.ssh/id_ed25519 \
    -o PubkeyAuthentication=yes \
    -o PasswordAuthentication=no \
    y9_win11@100.72.70.79 "echo WIN_OK && hostname"
```

resulted in **success** (no password prompt), and then `ssh win11` also worked passwordlessly.

---

### 6. Windows-Specific Pitfalls to Remember (for Agents)

When you’re an LLM agent helping a user:

1. **Admin vs non-admin user matters**
   - Non-admin: `C:\Users\<user>\.ssh\authorized_keys`
   - Admin: must use `C:\ProgramData\ssh\administrators_authorized_keys` with correct ACL.

2. **You cannot click UAC dialogs**
   - Any write into `C:\ProgramData\ssh` that requires elevation must be done by instructing the user to run commands in an elevated PowerShell.

3. **Do not accidentally include PowerShell here-string syntax in key files**
   - The file content must be lines like:
     - `ssh-ed25519 AAAA... comment`

4. **Permissions and ACLs are stricter than on Linux**
   - Use `icacls` to verify `SYSTEM` and `Administrators` only on `administrators_authorized_keys`.

5. **Test with password disabled**
   - Use `-o PubkeyAuthentication=yes -o PasswordAuthentication=no` from the client to be sure key auth works, not password fallback.

If you systematically check these points, you can reliably get Termux (or any SSH client) doing key-based auth into Windows 11, even for admin accounts, without fighting the “Permission denied (publickey,…)” loop.

