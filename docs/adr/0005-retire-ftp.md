# 0005 · Retire the FTP service

**Status:** Accepted

## Context
The original stack included vsftpd for file access to the WordPress volume. It was a
42 bonus requirement. Plain FTP sends credentials in cleartext, needs a range of
passive ports open, and needs a stateful always-on host with a shared file system,
none of which exists in Phase 2.

## Decision
FTP is not migrated. File transfer uses Cloud Storage: `gcloud storage cp/rsync`
authorized by IAM, with every access in the audit logs. In Phase 1 it is also
removed from the compose file, so ports 21 and 21000–21010 are never opened in the
cloud firewall.

## Alternatives considered
- **vsftpd with FTPS on the VM.** Encrypted, but still a password-based service on an
  internet-facing port, and incompatible with Phase 2.
- **SFTP on a VM, through IAP.** Secure, but an always-on VM just for file transfer.

## Consequences
- Smaller attack surface and one service fewer to operate.
- Uploading through the WordPress admin panel still works, because it writes to the
  Cloud Storage volume.
- Retiring a component is a valid migration outcome, and this ADR shows the reasoning.
