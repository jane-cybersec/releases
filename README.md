# JANE forensic agent: releases

This repository publishes the JANE forensic agent for Linux and for Windows. Each [release](../../releases) contains,
for Linux, the `.deb` and `.rpm` packages, a tarball, the installer script and a signed list of checksums, and for
Windows, a zip package, its installer script and its own signed list of checksums. [Install on Windows](#install-on-windows)
is further down.

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
5. Enrolls the host with the token and starts the `jane-agent` service.

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

## Install on Windows

In the JANE dashboard, go to **Hosts → Enroll host**, choose Windows and copy the command. Paste it in a PowerShell
opened **as administrator**. It looks like this, with a one-time enrollment token already filled in:

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12; $env:FORENSIC_TOKEN = 'fat1.…'; irm https://github.com/jane-cybersec/releases/releases/download/v0.2.0/install.ps1 | iex
```

The installer:

1. Checks the host can run the agent and reports every problem at once.
2. Downloads the package, its checksums and their signature.
3. Verifies the signature with the public key built into the script and the package against the signed checksums. If
   either fails, **nothing is installed**.
4. Copies the agent to `C:\Program Files\Jane\jane-agent` and creates the `jane-agent` service (it runs as LocalSystem,
   starts with Windows, and restarts by itself if it fails).
5. Enrolls the host with the token and starts the service.

The host appears in the dashboard within a minute. Running the installer again upgrades the agent in place. The token
is read from the environment and never printed or stored, and each token works once and expires.

### Requirements for Windows

- x64 Windows 10 22H2 (build 19045) or newer, with Windows PowerShell 5.1 or later, and an administrator.
- Tested end to end on Windows 11. Windows 10 22H2 and Windows Server are accepted by the installer's checks but have not
  been tested end to end yet.
- Outbound access to the JANE ingest endpoint named in your enrollment token.

### Verify a Windows release by hand

The installer does this for you. `SHA256SUMS.windows` is signed with an ECDSA P-256 key (`SHA256SUMS.windows.sig`, 64
bytes). The public half is the value of `ReleaseKeyPoint` near the top of `install.ps1`, and must be:

```
lH9xJQ82yhM0iwSWN04+AmvZ5bA6wiNTs2OhuzgMmCW7mqol+msEzWC3nl/MYmxyLgNh1nlEWpZ/TKz31atRRw==
```

(the SHA-256 of those 64 bytes is `a576835e200611a050febee1f621ae740e494b8e742a8d98b0bb048ecf399730`). Check it against
this page, not only against the script that came with the download. To check the package against the checksums:

```powershell
(Get-FileHash .\jane-agent-<version>-windows-amd64.zip -Algorithm SHA256).Hash   # compare with its line in SHA256SUMS.windows
```

### Install on Windows by hand

Download the zip from the release, verify it, then in an administrator PowerShell:

```powershell
Expand-Archive .\jane-agent-<version>-windows-amd64.zip -DestinationPath 'C:\Program Files\Jane\jane-agent'
'<token>' | & 'C:\Program Files\Jane\jane-agent\jane-agent.exe' enroll -
New-Service -Name jane-agent -BinaryPathName '"C:\Program Files\Jane\jane-agent\jane-agent.exe" service' -DisplayName 'JANE forensic agent' -StartupType Automatic
Start-Service jane-agent
```

### Uninstall on Windows

```powershell
$env:FORENSIC_UNINSTALL = '1'; irm https://github.com/jane-cybersec/releases/releases/download/v0.2.0/install.ps1 | iex
```

This removes the service and the program. The agent's data and keys stay in `C:\ProgramData\Jane\jane-agent`; set
`$env:FORENSIC_PURGE = '1'` as well to remove them too. Then revoke the host in the dashboard.

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
sudo apt install ./jane-agent_<version>-1_amd64.deb      # Debian, Ubuntu
sudo dnf install ./jane-agent-<version>-1.x86_64.rpm     # RHEL, Rocky, Alma, Amazon Linux
sudo jane-agent enroll -                             # paste the token, then press Ctrl-D
sudo systemctl enable --now jane-agent
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
| `dpapi_machine` | Windows: the key is protected with DPAPI, machine scope | Yes, if the whole disk is copied and it is not encrypted (BitLocker) |

To move a host to `tpm2`, give it a TPM2 (for example a vTPM from your cloud provider, which needs UEFI boot), then
re-enroll it with a new token.

## Uninstall

```sh
sudo apt purge jane-agent       # also deletes /etc/jane-agent and /var/lib/jane-agent
sudo dnf remove jane-agent      # keeps them; delete both by hand to remove every trace
```

Then revoke the host in the dashboard.

## Security

Report vulnerabilities privately to cdavidsanchez054@gmail.com rather than in a public issue.
