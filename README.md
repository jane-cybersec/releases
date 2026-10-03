# JANE forensic agent: releases

This repository publishes the JANE forensic agent for Linux. Each [release](../../releases) contains the `.deb` and
`.rpm` packages, a tarball, the installer script, and a signed list of checksums.

## Install

In the JANE dashboard, go to **Hosts → Enroll host** and copy the command. It looks like this, with a one-time
enrollment token already filled in:

```sh
curl --proto '=https' --tlsv1.2 -fsSL https://github.com/jane-cybersec/releases/releases/download/v0.1.0/install.sh \
  | FORENSIC_TOKEN='fat1.…' sudo --preserve-env=FORENSIC_TOKEN bash
```

Run it on the host, as a user who can `sudo`. The installer:

1. Checks the host can run the agent and reports every problem at once.
2. Downloads the package for your distribution.
3. Verifies it against the release's signed checksums, using the public key built into the script.
4. Installs it with `apt` or `dnf`.
5. Enrolls the host with the token and starts the `forensic-agent` service.

The host appears in the dashboard within a minute. Running the installer again upgrades the agent in place.

The token stays out of the process list, and the installer never prints or stores it. Each token works once and
expires, so create a new one for each host.

### Requirements

- x86_64 Linux, kernel 5.8 or newer with BTF (`/sys/kernel/btf/vmlinux`), systemd, glibc 2.28 or newer.
- Tested end to end on Ubuntu 22.04, Ubuntu 24.04 (with a TPM), Debian 12 and Rocky Linux 9. Debian 11, other RHEL 9
  rebuilds and Amazon Linux 2023 meet the requirements but are not in that test run yet.
- Older systems are refused before anything is installed: for example Ubuntu 20.04 on its stock 5.4 kernel, RHEL 8 or
  Amazon Linux 2.
- Outbound access to the JANE ingest endpoint named in your enrollment token.

## Verify a release by hand

The installer does this for you. To check the files yourself:

```sh
gpg --import jane-release-key.asc
gpg --fingerprint "JANE release signing key"   # must be D875 B4D1 E49F 1B5B C178  9251 E00B B439 5BBF BB7E
gpg --verify SHA256SUMS.asc SHA256SUMS
sha256sum --ignore-missing -c SHA256SUMS
```

Check the fingerprint against this page, not only against the key that came with the download.

## Install by hand

```sh
sudo apt install ./forensic-agent_0.1.0-1_amd64.deb      # Debian, Ubuntu
sudo dnf install ./forensic-agent-0.1.0-1.x86_64.rpm     # RHEL, Rocky, Alma, Amazon Linux
sudo forensic-agent enroll -                             # paste the token, then press Ctrl-D
sudo systemctl enable --now forensic-agent
```

## How the agent protects its keys

The agent encrypts everything it keeps on disk (its credentials, its signing key, and evidence not yet delivered) with
a master key. At enrollment it picks the strongest protection the host supports, and the dashboard shows which one
each host has:

| Level | When | A copied disk can read the key |
|---|---|---|
| `tpm2` | `systemd-creds` and a usable TPM2 | No |
| `host_key` | `systemd-creds`, no usable TPM2 (most cloud VMs) | Yes, if the whole disk is copied |
| `unsealed` | systemd older than 250, no `systemd-creds` (Ubuntu 22.04) | Yes |

To move a host to `tpm2`, give it a TPM2 (for example a vTPM from your cloud provider, which needs UEFI boot), then
re-enroll it with a new token.

## Uninstall

```sh
sudo apt purge forensic-agent       # also deletes /etc/forensic-agent and /var/lib/forensic-agent
sudo dnf remove forensic-agent      # keeps them; delete both by hand to remove every trace
```

Then revoke the host in the dashboard.

## Security

Report vulnerabilities privately to cdavidsanchez054@gmail.com rather than in a public issue.
