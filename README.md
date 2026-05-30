# User-level CA Bundle Setup for Downloading Models and Datasets from Hugging Face

## Purpose

Set up a user-level CA bundle so `curl`, Python, and Hugging Face tools can download models and datasets from Hugging Face.

---

## 1. Export the certificate chain

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

Find the root certificate. It usually has the same `subject` and `issuer`.

Example:

```text
subject=DC = edu, DC = vanderbilt, CN = vu root ca sha2
issuer=DC = edu, DC = vanderbilt, CN = vu root ca sha2
```

Assume the root certificate is `cert-03.pem`. Your file number may be different.

---

## 2. Create a personal CA bundle

```bash
mkdir -p ~/.certs
```

```bash
cp /etc/ssl/certs/ca-certificates.crt ~/.certs/my-ca-bundle.crt
```

Replace `cert-03.pem` with the actual root certificate file from Step 1:

```bash
cat cert-03.pem >> ~/.certs/my-ca-bundle.crt
```

---

## 3. Tell tools to use your personal CA bundle

```bash
export SSL_CERT_FILE=$HOME/.certs/my-ca-bundle.crt
export REQUESTS_CA_BUNDLE=$HOME/.certs/my-ca-bundle.crt
export CURL_CA_BUNDLE=$HOME/.certs/my-ca-bundle.crt
export HF_HUB_DISABLE_XET=1
```

Test Hugging Face access:

```bash
curl -I https://huggingface.co --connect-timeout 10
```

Expected result:

```text
HTTP/2 200
```

---

## 4. Make the settings permanent

```bash
cat >> ~/.bashrc <<'EOF'
export SSL_CERT_FILE=$HOME/.certs/my-ca-bundle.crt
export REQUESTS_CA_BUNDLE=$HOME/.certs/my-ca-bundle.crt
export CURL_CA_BUNDLE=$HOME/.certs/my-ca-bundle.crt
export HF_HUB_DISABLE_XET=1
EOF
```

```bash
source ~/.bashrc
```

---

## 5. Test Hugging Face CLI

Test a small model file:

```bash
hf download Qwen/Qwen3-4B config.json \
  --local-dir Models/Qwen3-4B-test
```

Expected result:

```text
Downloaded
path: Models/Qwen3-4B-test/config.json
```

Download a full model:

```bash
hf download Qwen/Qwen3-4B \
  --local-dir Models/Qwen3-4B
```

Download a dataset:

```bash
hf download HuggingFaceH4/ultrachat_200k \
  --repo-type dataset \
  --local-dir Datasets/ultrachat_200k
```

---

## 6. Test Python model download

Create `download_model.py`:

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

## 7. Test Python dataset download

Create `download_dataset.py`:

```python
import os
from huggingface_hub import snapshot_download

CA = os.path.expanduser("~/.certs/my-ca-bundle.crt")

os.environ["SSL_CERT_FILE"] = CA
os.environ["REQUESTS_CA_BUNDLE"] = CA
os.environ["CURL_CA_BUNDLE"] = CA
os.environ["HF_HUB_DISABLE_XET"] = "1"

snapshot_download(
    repo_id="HuggingFaceH4/ultrachat_200k",
    repo_type="dataset",
    local_dir="Datasets/ultrachat_200k",
    max_workers=1,
    etag_timeout=60,
)
```

Run:

```bash
python download_dataset.py
```

---

## 8. Optional cleanup

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
- Each user can create their own `~/.certs/my-ca-bundle.crt`.
- Other users can reuse a shared copy only if they have permission to read it.
- The proper system-wide fix is for an administrator to install the Vanderbilt/VUMC root CA into the server trust store.
- Do not use `curl -k` or disabled SSL verification as a long-term solution.
