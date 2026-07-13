# Issue: AAP Unable to SSH into abc-ant-sandbox as svc_ansible

## Error
```
svc_ansible@172.26.20.218: Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password)
```

## Investigation Steps Taken
1. Confirmed `svc_ansible` user exists on the sandbox server
2. Confirmed `.ssh/authorized_keys` exists with correct permissions (`600`) and ownership (`svc_ansible`)
3. Added new SSH public key to `authorized_keys` — confirmed 4 keys present
4. Verified public key in `authorized_keys` matches private key in AAP credential
5. Checked `sshd_config` — no obvious misconfigurations, `PubkeyAuthentication` defaults to yes

## Root Cause
```
sudo ls -laZ /home/svc_ansible/.ssh/
system_u:object_r:nfs_t:s0
```
The `svc_ansible` home directory is on an **NFS mount**. SSH refuses to use `authorized_keys` from NFS-mounted directories by default for security reasons, causing all public key authentication attempts to fail regardless of the key used.

## Fix Required (Admin Action Needed)
Two options:

**Option 1:** Move `authorized_keys` to a local path and update `sshd_config`:
```
AuthorizedKeysFile /etc/ssh/authorized_keys/%u
```

**Option 2:** Change `svc_ansible` home directory to a local path instead of NFS

Both options require root access and an `sshd` restart — escalate to server admin.
