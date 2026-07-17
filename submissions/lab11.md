# Lab 11 — BONUS — Submission

## Task 1: TLS + Security Headers

### nginx.conf (paste the SSL + header sections only — not the whole file)
```nginx
http{
    ...
    ssl_certificate     /etc/nginx/certs/localhost.crt;
    ssl_certificate_key /etc/nginx/certs/localhost.key;
    ssl_session_timeout 10m;
    ssl_session_cache   shared:SSL:10m;
    ssl_protocols TLSv1.3;
    ssl_ciphers "TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256:EECDH+AESGCM:EDH+AESGCM";
    ssl_prefer_server_ciphers off;
    ssl_stapling off;

    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "camera=(), geolocation=(), microphone=()" always;
    add_header Cross-Origin-Opener-Policy "same-origin" always;
    add_header Cross-Origin-Resource-Policy "same-origin" always;
    add_header Content-Security-Policy-Report-Only "default-src 'self'; img-src 'self' data:; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'" always;
}
```

### A. HTTPS redirect proof
```
HTTP/1.1 308 Permanent Redirect
Server: nginx
Date: Fri, 17 Jul 2026 18:30:05 GMT
Content-Type: text/html
Content-Length: 164
Connection: keep-alive
Location: https://localhost:443/
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), geolocation=(), microphone=()
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Resource-Policy: same-origin
Content-Security-Policy-Report-Only: default-src 'self'; img-src 'self' data:; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'
```

### B. TLS 1.3 proof
```
Connecting to ::1
Can't use SSL_get_servername
depth=0 CN=juice.local
verify error:num=18:self-signed certificate
CONNECTION ESTABLISHED
Protocol version: TLSv1.3
Ciphersuite: TLS_AES_256_GCM_SHA384
Peer certificate: CN=juice.local
```

### C. Security headers proof (all 6 present)
```
<HTTP/2 200 
server: nginx
date: Fri, 17 Jul 2026 18:30:44 GMT
content-type: text/html; charset=UTF-8
content-length: 9903
feature-policy: payment 'self'
x-recruiting: /#/jobs
accept-ranges: bytes
cache-control: public, max-age=0
last-modified: Fri, 17 Jul 2026 18:20:43 GMT
etag: W/"26af-19f714f3428"
vary: Accept-Encoding
strict-transport-security: max-age=63072000; includeSubDomains; preload
x-frame-options: DENY
x-content-type-options: nosniff
referrer-policy: strict-origin-when-cross-origin
permissions-policy: camera=(), geolocation=(), microphone=()
cross-origin-opener-policy: same-origin
cross-origin-resource-policy: same-origin
content-security-policy-report-only: default-src 'self'; img-src 'self' data:; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'
```

### What each header defends against

* HSTS: Protects against SSL-stripping attacks and forced HTTP downgrades by requiring the browser to use HTTPS.
* X-Content-Type-Options: nosniff: Prevents MIME-sniffing by stopping the browser from interpreting a file as a content type different from the one declared by the server.
* X-Frame-Options: DENY: Protects against clickjacking by preventing the page from being displayed inside a frame or iframe.
* Referrer-Policy: Prevents sensitive information leakage by controlling how much referrer data is sent with requests.
* Permissions-Policy: Reduces abuse of browser features by restricting access to APIs such as the camera, microphone, and geolocation.
* Content-Security-Policy: Protects mainly against XSS and malicious content injection by allowing resources to load or execute only from approved sources.

## Task 2: Production Posture

### Rate limit proof
| HTTP code | Count out of 60 |
|-----------|----------------:|
| 200 | 0 |
| 429 | 54 |
| 5xx | 6 |

### Timeout enforced
```
<Empty output>
```

Note: test with modified command because HTTPS does not accepts plaintext requests:
```
echo "GET / HTTP/1.0" | timeout 15 nc localhost 80 2>&1 | head -5 \
  | tee labs/lab11/results/timeout.txt
```

### Cipher hardening
```
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
```

### Cert rotation runbook (7 steps)
1. **Detect expiry**: Monitor certificate lifetime with an automated check such as `openssl x509 -checkend` or external TLS monitoring, and alert before the renewal window.
2. **Order new cert**: Request a replacement certificate from the CA using the current hostname/SAN list and key policy.
3. **Validate**: Verify the certificate chain, SANs, expiry date, private-key match, and file permissions before putting it in service.
4. **Atomic swap**: Stage the new cert and key beside the current files, run `nginx -t`, then swap the symlink or mounted files and reload Nginx.
5. **Verify**: Confirm the live endpoint serves the new certificate and still negotiates TLS 1.3 with the expected cipher.
6. **Rollback plan**: Keep the previous cert and key available, restore the old symlink or files, run `nginx -t`, and reload Nginx if validation fails.
7. **Audit**: Record the rotation time, certificate fingerprint, operator, validation output, and any rollback or incident notes.

### What OCSP stapling buys you (2-3 sentences, reference Reading 11)
OCSP stapling lets Nginx attach a fresh CA-signed revocation response during the TLS handshake, so clients can check certificate status without contacting the CA directly. As Reading 11 explains, this improves production privacy, latency, and revocation reliability, but it does not help this lab certificate because the self-signed cert has no real CA OCSP responder or trusted revocation status to staple.
