# User-level CA Bundle Setup for Downloading from Hugging Face

## Purpose

Set up a user-level CA bundle so Hugging Face files can be downloaded without `sudo`.

---

## Step 1: Export the Hugging Face certificate chain

Copy and paste:

```bash
echo | openssl s_client \
  -connect huggingface.co:443 \
  -servername huggingface.co \
  -showcerts > hf_certs.txt
```

---

## Step 2: Split the certificate chain into separate files

Copy and paste:

```bash
awk '
/BEGIN CERTIFICATE/ {i++; file=sprintf("cert-%02d.pem", i)}
{if (file) print > file}
/END CERTIFICATE/ {file=""}
' hf_certs.txt
```

This creates files such as:

```text
cert-01.pem
cert-02.pem
cert-03.pem
```

---

## Step 3: Print each certificate subject and issuer

Copy and paste:

```bash
for f in cert-*.pem; do
  echo "=== $f ==="
  openssl x509 -in "$f" -noout -subject -issuer
done
```

Look for the certificate where `subject` and `issuer` are the same.

Example:

```text
subject=DC = edu, DC = vanderbilt, CN = vu root ca sha2
issuer=DC = edu, DC = vanderbilt, CN = vu root ca sha2
```

That file is the root CA certificate.

In the example below, we assume the root CA is:

```text
cert-03.pem
```

If your root CA is a different file, replace `cert-03.pem` in the next commands with the correct file name.

---

## Step 4: Create a user-level CA bundle

Copy and paste:

```bash
mkdir -p ~/.certs
```

Copy and paste:

```bash
cp /etc/ssl/certs/ca-certificates.crt ~/.certs/my-ca-bundle.crt
```

Copy and paste, replacing `cert-03.pem` if your root CA file is different:

```bash
cat cert-03.pem >> ~/.certs/my-ca-bundle.crt
```

---

## Step 5: Set environment variables for the current shell

Copy and paste:

```bash
export SSL_CERT_FILE=$HOME/.certs/my-ca-bundle.crt
export REQUESTS_CA_BUNDLE=$HOME/.certs/my-ca-bundle.crt
export CURL_CA_BUNDLE=$HOME/.certs/my-ca-bundle.crt
export HF_HUB_DISABLE_XET=1
```

---

## Step 6: Test with a small Hugging Face file in the current path

Copy and paste:

```bash
curl -L \
  --cacert "$HOME/.certs/my-ca-bundle.crt" \
  -o ./hf_test_config.json \
  https://huggingface.co/Qwen/Qwen3-4B/resolve/main/config.json
```

Check the downloaded file:

```bash
ls -lh ./hf_test_config.json
```

Preview the first few lines:

```bash
head ./hf_test_config.json
```

Expected: the file should exist in the current directory and should contain JSON text.

---

## Step 7: Test Hugging Face CLI with a small file in the current path

Copy and paste:

```bash
hf download Qwen/Qwen3-4B config.json \
  --local-dir ./hf_cli_test
```

Check the downloaded file:

```bash
ls -lh ./hf_cli_test/config.json
```

Preview it:

```bash
head ./hf_cli_test/config.json
```

Expected: `./hf_cli_test/config.json` should exist and contain JSON text.

---

## Step 8: Test Python with a small file in the current path

Copy and paste:

```bash
cat > ./hf_python_test.py <<'PY'
import os
from huggingface_hub import hf_hub_download

CA = os.path.expanduser("~/.certs/my-ca-bundle.crt")

os.environ["SSL_CERT_FILE"] = CA
os.environ["REQUESTS_CA_BUNDLE"] = CA
os.environ["CURL_CA_BUNDLE"] = CA
os.environ["HF_HUB_DISABLE_XET"] = "1"

path = hf_hub_download(
    repo_id="Qwen/Qwen3-4B",
    filename="config.json",
    local_dir="./hf_python_test",
)

print("Downloaded to:", path)
PY
```

Run it:

```bash
python ./hf_python_test.py
```

Check the downloaded file:

```bash
ls -lh ./hf_python_test/config.json
```

Preview it:

```bash
head ./hf_python_test/config.json
```

Expected: `./hf_python_test/config.json` should exist and contain JSON text.

---

## Step 9: Make the settings permanent

Copy and paste:

```bash
cat >> ~/.bashrc <<'EOF'
export SSL_CERT_FILE=$HOME/.certs/my-ca-bundle.crt
export REQUESTS_CA_BUNDLE=$HOME/.certs/my-ca-bundle.crt
export CURL_CA_BUNDLE=$HOME/.certs/my-ca-bundle.crt
export HF_HUB_DISABLE_XET=1
EOF
```

Reload the shell configuration:

```bash
source ~/.bashrc
```

Confirm the variables are set:

```bash
echo "$SSL_CERT_FILE"
echo "$REQUESTS_CA_BUNDLE"
echo "$CURL_CA_BUNDLE"
echo "$HF_HUB_DISABLE_XET"
```

Expected:

```text
/home/YOUR_USERNAME/.certs/my-ca-bundle.crt
/home/YOUR_USERNAME/.certs/my-ca-bundle.crt
/home/YOUR_USERNAME/.certs/my-ca-bundle.crt
1
```

---

## Step 10: Optional cleanup

After the tests work, remove temporary certificate extraction files:

```bash
rm -f hf_certs.txt cert-*.pem
```

Optional: remove test downloads and test script:

```bash
rm -rf ./hf_test_config.json ./hf_cli_test ./hf_python_test ./hf_python_test.py
```

Keep this file:

```text
~/.certs/my-ca-bundle.crt
```

---

## Notes

- This setup does not require `sudo`.
- Each user can create their own `~/.certs/my-ca-bundle.crt`.
- The file `~/.certs/my-ca-bundle.crt` is the important file to keep.
- Do not use `curl -k` or disabled SSL verification as a long-term solution.
