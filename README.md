# User-level CA Bundle Setup for Hugging Face on VUMC/Vanderbilt Linux Servers

## Purpose

On some VUMC/Vanderbilt Linux servers, accessing Hugging Face may fail with an SSL error:

```bash
SSL certificate problem: self-signed certificate in certificate chain
```

This usually happens because HTTPS traffic is inspected and re-signed by a Vanderbilt/VUMC security certificate chain.

Example certificate chain:

```text
Root CA: vu root ca sha2
Intermediate CA: VEC Security Engineering CA
Website cert: huggingface.co
```

If you do not have `sudo`, you can create a user-level CA bundle and configure `curl`, Python, and Hugging Face tools to use it.

---

## 1. Confirm the SSL problem

Run:

```bash
curl -I https://huggingface.co --connect-timeout 10
```

If you see something like:

```bash
SSL certificate problem: self-signed certificate in certificate chain
```

then test whether the network can reach Hugging Face when certificate verification is ignored:

```bash
curl -k -I https://huggingface.co --connect-timeout 10
```

If this returns:

```text
HTTP/2 200
```

then Hugging Face is reachable, but your Linux environment does not trust the current certificate chain.

---

## 2. Export the certificate chain

Run:

```bash
echo | openssl s_client \
  -connect huggingface.co:443 \
  -servername huggingface.co \
  -showcerts > hf_certs.txt
```

Split the certificates into separate files:

```bash
awk '
/BEGIN CERTIFICATE/ {i++; file=sprintf("cert-%02d.pem", i)}
{if (file) print > file}
/END CERTIFICATE/ {file=""}
' hf_certs.txt
```

Inspect each certificate:

```bash
for f in cert-*.pem; do
  echo "=== $f ==="
  openssl x509 -in "$f" -noout -subject -issuer
done
```

Look for the root certificate. It usually has the same `subject` and `issuer`.

Example:

```text
subject=DC = edu, DC = vanderbilt, CN = vu root ca sha2
issuer=DC = edu, DC = vanderbilt, CN = vu root ca sha2
```

In this example, assume the root certificate is `cert-03.pem`.

Your file number may be different.

---

## 3. Create a personal CA bundle

Create a certificate folder in your home directory:

```bash
mkdir -p ~/.certs
```

Copy the system CA bundle:

```bash
cp /etc/ssl/certs/ca-certificates.crt ~/.certs/my-ca-bundle.crt
```

Append the Vanderbilt/VUMC root CA.

Replace `cert-03.pem` with the actual root certificate file from Step 2:

```bash
cat cert-03.pem >> ~/.certs/my-ca-bundle.crt
```

---

## 4. Tell tools to use your personal CA bundle

Run:

```bash
export SSL_CERT_FILE=$HOME/.certs/my-ca-bundle.crt
export REQUESTS_CA_BUNDLE=$HOME/.certs/my-ca-bundle.crt
export CURL_CA_BUNDLE=$HOME/.certs/my-ca-bundle.crt
```

Test again:

```bash
curl -I https://huggingface.co --connect-timeout 10
```

Expected result:

```text
HTTP/2 200
```

---

## 5. Make the settings permanent

Add the environment variables to `~/.bashrc`:

```bash
cat >> ~/.bashrc <<'EOF'
export SSL_CERT_FILE=$HOME/.certs/my-ca-bundle.crt
export REQUESTS_CA_BUNDLE=$HOME/.certs/my-ca-bundle.crt
export CURL_CA_BUNDLE=$HOME/.certs/my-ca-bundle.crt
export HF_HUB_DISABLE_XET=1
EOF
```

Reload your shell configuration:

```bash
source ~/.bashrc
```

`HF_HUB_DISABLE_XET=1` is helpful on some enterprise networks where Hugging Face Xet-backed large file downloads may fail.

---

## 6. Test Hugging Face CLI

For newer Hugging Face CLI versions, use `hf download`.

Test a small file first:

```bash
hf download Qwen/Qwen3-4B config.json \
  --local-dir Models/Qwen3-4B-test
```

Expected result:

```text
Downloaded
path: Models/Qwen3-4B-test/config.json
```

Then test the full model download:

```bash
hf download Qwen/Qwen3-4B \
  --local-dir Models/Qwen3-4B
```

---

## 7. Test Python download

Create a file called `download_model.py`:

```python
import os
from huggingface_hub import snapshot_download

CA = os.path.expanduser("~/.certs/my-ca-bundle.crt")

os.environ["SSL_CERT_FILE"] = CA
os.environ["REQUESTS_CA_BUNDLE"] = CA
os.environ["CURL_CA_BUNDLE"] = CA
os.environ["HF_HUB_DISABLE_XET"] = "1"

snapshot_download(
    repo_id="Qwen/Qwen3-4B",
    local_dir="Models/Qwen3-4B",
    max_workers=1,
    etag_timeout=60,
)
```

Run:

```bash
python download_model.py
```

---

## 8. Optional: test Python requests directly

Run:

```bash
python - <<'PY'
import os
import requests

print("SSL_CERT_FILE =", os.environ.get("SSL_CERT_FILE"))
print("REQUESTS_CA_BUNDLE =", os.environ.get("REQUESTS_CA_BUNDLE"))

url = "https://huggingface.co/Qwen/Qwen3-4B/resolve/main/config.json"

r = requests.head(url, allow_redirects=False, timeout=20)
print("HEAD status =", r.status_code)
print("x-repo-commit =", r.headers.get("x-repo-commit"))
print("etag =", r.headers.get("etag"))
print("location =", r.headers.get("location"))

g = requests.get(url, allow_redirects=True, timeout=20)
print("GET status =", g.status_code)
print(g.text[:200])
PY
```

A good result should include:

```text
HEAD status = 307
x-repo-commit = ...
GET status = 200
```

---

## 9. Optional cleanup

After everything works, remove temporary certificate files:

```bash
rm -f hf_certs.txt cert-*.pem
```

Keep this file:

```bash
~/.certs/my-ca-bundle.crt
```

---

## Notes

- This setup does not require `sudo`.
- This is a user-level workaround.
- Each user can create their own `~/.certs/my-ca-bundle.crt`.
- Other users can reuse a shared copy only if they have permission to read it.
- The proper system-wide fix is for an administrator to install the Vanderbilt/VUMC root CA into the server trust store.
- Do not use `curl -k` or disabled SSL verification as a long-term solution.
