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

In this example, the root CA file is:

```text
cert-03.pem
```

Your file number may be different.

Do not use the website certificate, such as `cert-01.pem`. Use the root CA certificate.

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
export HF_ENDPOINT=https://huggingface.co
```

`HF_ENDPOINT=https://huggingface.co` forces Hugging Face tools to use the official Hugging Face endpoint instead of a mirror such as `hf-mirror.com`.

---

## Step 6: Test Hugging Face download with a small file in the current path

Copy and paste:

```bash
HF_ENDPOINT=https://huggingface.co \
HF_HUB_DISABLE_XET=1 \
SSL_CERT_FILE=$HOME/.certs/my-ca-bundle.crt \
REQUESTS_CA_BUNDLE=$HOME/.certs/my-ca-bundle.crt \
CURL_CA_BUNDLE=$HOME/.certs/my-ca-bundle.crt \
hf download Qwen/Qwen3-4B config.json \
  --local-dir ./hf_official_test
```

Check the downloaded file:

```bash
ls -lh ./hf_official_test/config.json
head ./hf_official_test/config.json
```

Expected result:

```text
./hf_official_test/config.json should exist and contain JSON text.
```

---

## Step 7: Make the settings permanent for future use

Copy and paste:

```bash
perl -0pi -e 's/\n# Hugging Face CA bundle setup.*?# End Hugging Face CA bundle setup\n//s' ~/.bashrc

cat >> ~/.bashrc <<'EOF'

# Hugging Face CA bundle setup
export SSL_CERT_FILE=$HOME/.certs/my-ca-bundle.crt
export REQUESTS_CA_BUNDLE=$HOME/.certs/my-ca-bundle.crt
export CURL_CA_BUNDLE=$HOME/.certs/my-ca-bundle.crt
export HF_HUB_DISABLE_XET=1
export HF_ENDPOINT=https://huggingface.co
# End Hugging Face CA bundle setup
EOF

source ~/.bashrc
```

Confirm the variables are set:

```bash
echo "$SSL_CERT_FILE"
echo "$REQUESTS_CA_BUNDLE"
echo "$CURL_CA_BUNDLE"
echo "$HF_HUB_DISABLE_XET"
echo "$HF_ENDPOINT"
```

Expected:

```text
/home/YOUR_USERNAME/.certs/my-ca-bundle.crt
/home/YOUR_USERNAME/.certs/my-ca-bundle.crt
/home/YOUR_USERNAME/.certs/my-ca-bundle.crt
1
https://huggingface.co
```

---

## Step 8: Optional cleanup

After the test works, remove temporary certificate extraction files:

```bash
rm -f hf_certs.txt cert-*.pem
```

Optional: remove the small Hugging Face test download:

```bash
rm -rf ./hf_official_test
```

Keep this file:

```text
~/.certs/my-ca-bundle.crt
```

Do not delete `~/.certs/my-ca-bundle.crt`; it is the user-level CA bundle used for future Hugging Face downloads.

---

## Notes

* This setup does not require `sudo`.
* Each user can create their own `~/.certs/my-ca-bundle.crt`.
* The file `~/.certs/my-ca-bundle.crt` is the important file to keep.
* `HF_ENDPOINT=https://huggingface.co` avoids accidentally using `hf-mirror.com`.
* `HF_HUB_DISABLE_XET=1` can help avoid Xet-related download issues on enterprise networks.
* Do not use `curl -k` or disabled SSL verification as a long-term solution.
