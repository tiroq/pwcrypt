# pwcrypt

Minimal console tool to encrypt/decrypt secrets with your *master passphrase*, powered by OpenSSL (PBKDF2 + salt + AES-256).  
Single file, no dependencies beyond OpenSSL.

> Use it to safely stash small secrets (passwords, tokens) in dotfiles, scripts, or notes without introducing heavy key management.

---

## Features

- 🔒 **Password-based encryption** (no key files to track)
- 🧂 **Salt + PBKDF2 (SHA-256, 200k iters)** for key derivation
- 🧾 **Base64 output** for easy copy/paste and VCS storage
- 🖥️ Works with **stdin/stdout** or files
- 🧑‍💻 Single-file **Bash** script, POSIX-friendly
- ⚙️ Defaults to **AES-256-CBC**; optionally switch to **AES-256-GCM** (AEAD)

---

## Requirements

- `bash` (4+ recommended)
- `openssl` (1.1.1+ recommended)

---

## Install

```bash
# Option A: clone
git clone https://github.com/tiroq/pwcrypt.git
cd pwcrypt
chmod +x pwcrypt

# Option B: download the script directly
curl -fsSL -o pwcrypt https://raw.githubusercontent.com/tiroq/pwcrypt/main/pwcrypt && chmod +x pwcrypt
```

Optionally put it on your PATH:

```bash
install -m 0755 pwcrypt /usr/local/bin/
```

---

## Quick start

```bash
# Encrypt (interactive secret and master passphrase)
pwcrypt enc > secret.enc

# Encrypt a file
pwcrypt enc -i mypass.txt -o secret.enc

# Decrypt to stdout
pwcrypt dec -i secret.enc

# Decrypt to file
pwcrypt dec -i secret.enc -o mypass.txt
```

`pwcrypt` will prompt for the **master passphrase** via `/dev/tty` so it won’t leak through pipelines or history.

---

## Usage

```text
pwcrypt enc [-i FILE] [-o FILE]
pwcrypt dec [-i FILE] [-o FILE]

Without -i reads from stdin (for enc and a TTY, it asks secret interactively).
Without -o writes to stdout.
```

### One-liners (no script)

```bash
# Encrypt string
read -s -p "Master: " M; echo; read -s -p "Secret: " S; echo; \
printf '%s' "$S" | openssl enc -aes-256-cbc -salt -pbkdf2 -iter 200000 -md sha256 -a -pass env:M

# Decrypt file
read -s -p "Master: " M; echo; \
openssl enc -aes-256-cbc -salt -pbkdf2 -iter 200000 -md sha256 -a -d -pass env:M -in secret.enc
```

---

## Security notes

- **KDF**: PBKDF2-HMAC-SHA256 with **200,000** iterations (tune up if your CPU is fast).
- **Salted**: each encryption uses a random salt.
- **Encoding**: Base64 output makes it safe to paste or commit (the *ciphertext*, not the plaintext).
- **Algorithm**: default `AES-256-CBC`. For authenticated encryption, consider `AES-256-GCM` (OpenSSL ≥ 1.1.1).

> ⚠️ CBC does not authenticate. If an attacker can modify ciphertext, prefer **GCM**. See below.

---

## Switching to AES-256-GCM (optional)

If your OpenSSL supports GCM, you can edit the script:

```bash
openssl_enc_args=(
  enc -aes-256-gcm
  -salt
  -pbkdf2
  -iter 200000
  -md sha256
  -a
)
```

GCM provides integrity/authentication; decryption fails if the passphrase or data are wrong.

---

## Format & compatibility

OpenSSL `enc` with `-a -salt -pbkdf2 -iter N -md sha256 -aes-256-<mode>`.  
The Base64 output typically starts with `U2FsdGVkX1...` when using `-salt`.

You can decrypt with plain OpenSSL:

```bash
openssl enc -aes-256-cbc -d -a -pbkdf2 -iter 200000 -md sha256 -in secret.enc -out plaintext.txt
```

---

## Deterministic behavior

- New salt each time → different ciphertext for the same plaintext + master passphrase (good!).
- To reproduce exactly, you’d need to pin a salt (not recommended).

---

## Testing

```bash
# Round-trip self-test
echo "hello" | ( read -s -p "Master: " M; echo; \
openssl enc -aes-256-cbc -salt -pbkdf2 -iter 200000 -md sha256 -a -pass env:M | \
openssl enc -aes-256-cbc -d -a -pbkdf2 -iter 200000 -md sha256 -pass env:M )
```

You should see `hello`.

---

## Troubleshooting

- **`bad decrypt`**: wrong master passphrase or corrupted ciphertext.
- **`unknown option -pbkdf2`**: OpenSSL too old. Upgrade to ≥ 1.1.1.
- **No TTY** (CI): export the passphrase via env var and pipe data:
  ```bash
  export MASTER="your-long-passphrase"
  printf '%s' "$SECRET" | openssl enc -aes-256-cbc -salt -pbkdf2 -iter 200000 -md sha256 -a -pass env:MASTER
  ```

---

## FAQ

**Q: Is this a password manager?**  
A: No. It’s a thin wrapper around OpenSSL for small secrets.

**Q: Can I store binary data?**  
A: Yes. Use `-i`/`-o` with files; Base64 keeps ciphertext text-friendly.

**Q: What passphrase length is safe?**  
A: Use a long, unique passphrase (e.g., 5–7 random words).

---

## Roadmap

- `--algo` flag (`cbc`/`gcm`)
- `--iters` flag and param stamping
- Simple header with versioning of parameters

---

## License

MIT © <your-name>

---

## Acknowledgements

Built for simplicity and auditability: one script, sensible defaults.
