# gpgsm-wsl

Sign Git commits from WSL using a certificate stored on a YubiKey (PIV), by delegating to `gpgsm.exe` running on the Windows host via Gpg4win.

---

## Overview

WSL does not have native access to smart-card readers, so `gpgsm` inside WSL cannot talk to a YubiKey directly. This setup works around that by:

1. Configuring Gpg4win on Windows to talk to the YubiKey over PCSC.
2. Providing a thin wrapper script (`gpgsm-wsl`) inside WSL that translates Linux paths to Windows paths and forwards all calls to `gpgsm.exe` on the host.
3. Pointing Git's X.509 signing pipeline at that wrapper.

---

## Prerequisites

- Windows with a YubiKey (PIV applet) plugged in
- [Gpg4win](https://www.gpg4win.org/) installed (install GnuPG and Kleopatra)
- WSL 2 (any distro)

---

## Certificate Generation

This section covers creating a self-signed CA, generating a CSR from the YubiKey, signing it, and importing the resulting certificate back onto the key.

### 1. Generate a CSR on the YubiKey (PIV slot 9a)

Use [YubiKey Manager](https://www.yubico.com/support/download/yubikey-manager/) to generate a CSR directly on the key so that the private key never leaves the hardware.

In the YubiKey Manager GUI:

1. Go to **Applications → PIV → Certificates → Authentication (9a)**
2. Click **Generate** → choose **Certificate Signing Request (CSR)**
3. Fill in the subject (e.g. `CN=Your Name,E=you@example.com`)
4. Save the CSR as `csr-9a.csr`

> Slot **9a** (Authentication) is the one confirmed to work with this setup. I was not able to make 9c/9d work.

### 2. Create a self-signed CA

```bash
# Generate the CA private key
openssl genrsa -out ca.key 4096

# Self-sign the CA certificate (10-year validity)
openssl req -x509 -new -nodes -key ca.key -days 3650 -out ca.crt -subj "/CN=Cat Factory CA/"
```

Keep `ca.key` and `ca.crt` somewhere safe — you will need `ca.crt` later when importing into Gpg4win so it trusts signatures made by this CA.

### 3. Create the OpenSSL extension config

Edit [`openssl.config`](openssl.config) and update the email address in the `[ san ]` section to match your Git identity.

The `otherName` OID (`1.3.6.1.4.1.311.20.2.3`) is the Microsoft UPN SAN, which Gpg4win/gpgsm uses to match a certificate to a Git `user.signingkey` value.

### 4. Sign the CSR

```bash
openssl x509 -req \
  -days 3650 \
  -in csr-9a.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out cert.pem \
  -extfile openssl.config \
  -extensions v3_req
```

Verify the extensions were applied correctly:

```bash
openssl x509 -in cert.pem -noout -text | grep -A5 "X509v3 extensions"
```

### 5. Import the certificate back onto the YubiKey

In YubiKey Manager:

1. Go to **Applications → PIV → Certificates → Authentication (9a)**
2. Click **Import** and select `cert.pem`

---

## Windows Setup

### 1. Install Gpg4win

Download and install from <https://www.gpg4win.org/>. Make sure to select at least **GnuPG** and **Kleopatra** during installation.

### 2. Configure `gpg-agent`

Edit `%APPDATA%\gnupg\gpg-agent.conf` (create it if it does not exist):

```
pinentry-program "C:\Program Files\Gpg4win\bin\pinentry.exe"
# log-file C:\temp\gpg-agent.log
# debug-level basic
```

### 3. Configure `scdaemon`

Edit `%APPDATA%\gnupg\scdaemon.conf` (create it if it does not exist):

```
disable-ccid
pcsc-shared
application-priority piv
# log-file C:\temp\scd.log
# debug-level guru
```

`disable-ccid` tells scdaemon to use the system PCSC stack instead of its own CCID driver, which is required for YubiKey PIV to work reliably on Windows.

### 4. Restart the GnuPG agent

Open PowerShell or Command Prompt and run:

```powershell
& 'C:\Program Files\GnuPG\bin\gpgconf.exe' --kill all
```

### 5. Verify the YubiKey is visible

```powershell
certutil.exe -scinfo
```

Your certificate should appear in the output. If it does not, check that the YubiKey PIV applet is initialised and that the PCSC service is running.

---

## Importing Certificates into GnuPG (Kleopatra)

`gpgsm --learn-card` can detect the card but fails at the PIN prompt in some environments, so use Kleopatra to import the certificate directly instead.

### 1. Import the signing certificate from the YubiKey

1. Open **Kleopatra**
2. Go to **Tools → Manage Smartcards**
3. Select your YubiKey from the card list
4. Click **Import Certificate** — this reads `cert.pem` off the card and registers it with gpgsm

### 2. Import the CA certificate

If the CA certificate was not imported automatically in the previous step, import it manually:

1. In Kleopatra, go to **File → Import...**
2. Select your `ca.crt` file

### 3. Trust the CA certificate

gpgsm will refuse to use a certificate whose CA it does not explicitly trust:

1. In Kleopatra's certificate list, find the imported CA certificate
2. Right-click it → **Change Validity** (or **Certify**)
3. Set trust to **Full** and confirm

### 4. Verify gpgsm can see the key

```powershell
& 'C:\Program Files\GnuPG\bin\gpgsm.exe' --list-keys
```

The signing certificate should appear in the output. Copy its fingerprint — you will need it for the Git configuration in the next section.

---

## WSL Setup

### 1. Install the wrapper script

Copy [`gpgsm-wsl`](gpgsm-wsl) from this repository to `/usr/bin/` and make it executable:

```bash
sudo install -m 0755 gpgsm-wsl /usr/bin/gpgsm-wsl
```

The script handles two things that would otherwise break the Windows binary:
- **Path translation** — converts absolute Linux paths (e.g. `/home/user/file`) to their Windows equivalents using `wslpath`.
- **Stdin forwarding** — when Git passes `-` to signal that signed data should be read from stdin, the script buffers it to a temporary file and passes the Windows path to `gpgsm.exe` instead, since cross-OS stdin piping is unreliable.

### 2. Configure Git

Find your certificate fingerprint:

```bash
/mnt/c/Program\ Files/GnuPG/bin/gpgsm.exe --list-keys
```

Then configure Git globally:

```bash
git config --global gpg.format x509
git config --global gpg.x509.program gpgsm-wsl
git config --global commit.gpgsign true
git config --global user.signingkey "<fingerprint>"
```

Replace `<fingerprint>` with the fingerprint of your signing certificate as shown by `--list-keys`.

---

## Verification

Make a test commit and confirm the signature is present:

```bash
git commit --allow-empty -m "test: verify gpgsm signing"
git log --show-signature -1
```

You should see a line like `Good signature from [...]` in the output.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `gpgsm.exe` cannot find the key | gpg-agent not running | Run `gpgconf.exe --kill all` and retry |
| PIN prompt never appears | Wrong `pinentry-program` path | Verify the path in `gpg-agent.conf` matches your Gpg4win install |
| `certutil -scinfo` shows no card | PCSC service stopped or YubiKey not inserted | Restart the Smart Card service and replug the YubiKey |
| `wslpath: invalid argument` | Path contains characters special to WSL | Ensure filenames do not contain backslashes or colons |

To enable verbose logging, uncomment the `log-file` and `debug-level` lines in the config files and restart the agent.
