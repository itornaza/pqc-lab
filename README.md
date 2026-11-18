# pqc-lab

This is a hands-on lab for post-quantum cryptography (PQC) with OpenSSL 3.5+ on macOS that is designed to help with an organisations assessment on PQC readiness. 

Each step has the exact command, the real captured output, why it worked or failed, and what that means for a PQC readiness assessment.

No custom code and no benchmarking will be applied, only official tools and their documented options. Tool versions are given with each result, because newer releases may behave differently since this is a fast-evolving topic.

**All keys in this lab are throwaway test keys that protect nothing.** The `keys/` and `logs/` folders
(and any `*-key.pem`) are excluded from git by `.gitignore`, so you create your own keys and logs as you go.

The example run as well as the tools are as of Nov 2026.

## Get the lab

Clone the repository and work from its folder.

```bash
git clone https://github.com/itornaza/pqc-lab.git
cd pqc-lab
```

All commands below are run from this `pqc-lab` folder, and all generated files stay inside it.

```bash
mkdir -p keys logs
```

- `keys/` holds the keys and certificates (Labs 1 and 3).
- `logs/` holds the scanner reports and the CBOM (Labs 3 and 4).

Both folders start empty and are never committed, because their files are specific to your machine.

## Tasks

| Lab | Task                                                                                       |
| --- | ------------------------------------------------------------------------------------------ |
| 1   | Create an ML-DSA-65 key pair and a self-signed certificate                                 |
| 2   | Run TLS 1.3 with the hybrid `X25519MLKEM768` key exchange                                  |
| 3   | Scan the PQC endpoint with nmap, testssl.sh and sslyze, and compare what each tool can see |
| 4   | Produce a CycloneDX cryptographic bill of materials (CBOM) with cbomkit-theia              |

## Requirements

### Python and the virtual environment

The lab requires Python 3.10 or newer (sslyze 6.3.1 needs `>=3.10`; tested with 3.13.2). Check which one your shell finds first:

```sh
which python3
python3 --version
```

On macOS, `/usr/bin/python3` is Apple's system Python (3.9.6 on the test Mac), which is too old. Install a newer one from Homebrew or python.org if needed.

**1. Create the virtual environment** inside the `pqc-lab` folder, calling your newer Python explicitly:

```sh
python3.13 -m venv .venv   # any Python 3.10+, not /usr/bin/python3
```

- `-m venv .venv` creates a private Python environment in the hidden folder `.venv/`. Packages installed there stay inside the lab and do not touch the rest of your Mac. `.venv/` is excluded from git.

**2. Install the pinned packages** from the `requirements.txt` that ships with this repo:

```sh
.venv/bin/pip install -r requirements.txt
```

- `requirements.txt` pins the exact versions used in this lab (sslyze 6.3.1 with nassl 5.4.0 and their dependencies), so your results match the outputs shown below.
- Calling `.venv/bin/pip` directly installs into the venv without needing `source .venv/bin/activate`.

**3. Check** that the venv is the one in use:

```sh
.venv/bin/python -c "import sys; print(sys.prefix)"
```

It should print a path ending in `pqc-lab/.venv`.

**If something breaks** (for example after a Python upgrade, or when moving to another machine), delete the venv and repeat steps 1 and 2:

```sh
rm -rf .venv
python3.13 -m venv .venv
.venv/bin/pip install -r requirements.txt
```

### OpenSSL

We need OpenSSL 3.5 or later. That release (3.5.0, 8 April 2025) added native ML-KEM, ML-DSA and SLH-DSA, and changed the default TLS key shares to offer X25519MLKEM768 and X25519 ([NEWS.md for the 3.5 branch](https://raw.githubusercontent.com/openssl/openssl/openssl-3.5/NEWS.md)).

Do not use macOS `/usr/bin/openssl`. On the test Mac that is LibreSSL 3.3.6, which does not provide ML-DSA or ML-KEM.

You can use Homebrew's `openssl@3` formula, which tracks the 3.5 LTS branch. Point every command at it:

```sh
export OPENSSL=/opt/homebrew/opt/openssl@3/bin/openssl
"$OPENSSL" version
```

- `export OPENSSL=...` stores the Homebrew OpenSSL path in a variable. It does not change `PATH`. A bare `openssl` or `which openssl` still hits Apple’s LibreSSL at `/usr/bin/openssl`. Always call `"$OPENSSL"`, quotes included!
- `"$OPENSSL" version` prints the library we are actually using, so we can confirm it is OpenSSL 3.5+ and not LibreSSL.

We are going to run that command in every new terminal window we use, and it will be noted on the associated command set.

Expected output (captured 16 Nov 2026; terminal only, not saved to a file):

```
OpenSSL 3.5.9 29 Sep 2026 (Library: OpenSSL 3.5.9 29 Sep 2026)
```

This verifies version 3.5 or later, so ML-KEM, ML-DSA and `X25519MLKEM768` all work.

### Scanners

We will use three scanners. testssl.sh and nmap come from Homebrew; sslyze is a Python package and was already installed into `.venv/` from `requirements.txt` above. Run everything from within the `pqc-lab` directory:

```sh
brew install testssl nmap
```

The versions of these tools used in the lab are Nmap 7.993, testssl.sh 3.2.5, sslyze 6.3.1.

To keep Homebrew and every brew-installed tool current, run `brew update && brew upgrade && brew cleanup`. Add `brew outdated` first if you want to see what would change. An upgrade can also move `openssl@3` to a newer version, and then the version line above will differ.

## Algorithms we are going to use

We are going to use two algorithms in our lab, namely `ML-DSA-65` and `X25519MLKEM768`

- **ML-DSA** is a signature algorithm, so it belongs in certificates and we are going to see more on that in Lab 1. The official `genpkey` example uses ML-DSA-65, which is NIST security category 3.

- **X25519MLKEM768** is a TLS 1.3 hybrid key-agreement *group*, not a certificate algorithm. OpenSSL 3.5 generates those keys during the handshake, so they cannot be put in an X.509 certificate ([EVP_PKEY-MLX-KEM](https://docs.openssl.org/master/man7/EVP_PKEY-MLX-KEM/)).

That split matches [NIST IR 8547](https://csrc.nist.gov/pubs/ir/8547/final): migrate key establishment first because of harvest-now-decrypt-later (HNDL), and treat ML-KEM-768 as the default KEM (category 3). The hybrid stays FIPS-validatable because one approved input (ML-KEM) is enough.

### Hybrid key agreement, and why signatures can wait longer

HNDL targets *confidentiality*. An attacker can record a TLS handshake today and decrypt it later if the key-agreement algorithm falls to a cryptanalytically relevant quantum computer. So the urgent part is key establishment: the handshake’s KEM or hybrid group. A recorded session that used only X25519 (or another classical key agreement) could be opened later; a session that used ML-KEM-768, alone or in a hybrid, is meant to stay confidential even then.

A hybrid such as `X25519MLKEM768` runs a classical key agreement (X25519) and a post-quantum KEM (ML-KEM-768) together, then mixes both shared secrets. An attacker has to break *both*. In the unfortunate scenario that a flaw is later found in ML-KEM, X25519 still protects the session against today’s non-quantum attackers. If a quantum computer arrives and breaks X25519, ML-KEM-768 still protects it. The main reason for hybrids is this defence in depth while the new algorithms are still young — not only a graceful goodbye to the old ones.

A secondary benefit is transition: old clients that do not speak ML-KEM can still negotiate something, and the classical half can be dropped later once the PQ algorithm is trusted and widely deployed. There is also a compliance reason: [IR 8547](https://csrc.nist.gov/pubs/ir/8547/final)’s hybrid rule says a combination stays FIPS-validatable when at least one approved algorithm contributes (here, ML-KEM).

Signatures are a different beast. They authenticate (who you are talking to) and protect integrity. A recorded handshake from today does not become forgeable in the past just because ML-DSA was not used. The pressure on signatures is “when an attacker can break the *current*
signature and impersonate someone,” plus anything that must stay verifiable for a long time (firmware, document signing, roots of trust). Those long-lived signatures are more urgent than a short-lived TLS server certificate, but still generally behind confidentiality for HNDL.

The takeaway here is to use PQC or hybrid key agreement now, against recording; schedule PQC signatures on a longer ramp, however, sooner rather than later where the signature itself has a long lifetime.

This lab follows that split: 
- ML-DSA-65 in the certificate (Lab 1),
- `X25519MLKEM768` in the handshake (Lab 2).

## Lab 1 — Generate ML-DSA keys and self-signed certificate

We are going to create an ML-DSA-65 key pair, then use the private key to issue a self-signed X.509 certificate that carries the public key, and eventually print both files to confirm the algorithm names.

This lab is about *signatures*, not confidentiality. The certificate will later prove the TLS server’s identity (Lab 2). ML-DSA-65 is the post-quantum signature in that certificate: the public key goes in Subject Public Key Info, and the same private key signs the certificate (self-signed) and, in Lab 2, the TLS 1.3 CertificateVerify message. We use ML-DSA here so the lab shows a PQ signature, but [IR 8547](https://csrc.nist.gov/pubs/ir/8547/final) does not treat TLS server signatures as the HNDL-urgent migration. A short-lived localhost cert is the less urgent half; firmware- or document-signing keys with a long lifetime would be the signature cases to move sooner. We do *not* put a KEM or the hybrid group into this certificate.

Commands from
[openssl-genpkey(1)](https://docs.openssl.org/3.5/man1/openssl-genpkey/)
and
[openssl-req(1)](https://docs.openssl.org/3.5/man1/openssl-req/).
`genpkey` documents that ML-DSA and ML-KEM support was added in OpenSSL 3.5.

#### Run the following Commands

```sh
export OPENSSL=/opt/homebrew/opt/openssl@3/bin/openssl

# 1. Generate a new ML-DSA-65 key pair
"$OPENSSL" genpkey -algorithm ML-DSA-65 -out keys/ml-dsa-key.pem

# 2. Build a self-signed certificate
"$OPENSSL" req -new -x509 -key keys/ml-dsa-key.pem -out keys/ml-dsa-cert.pem \
  -days 365 -subj "/CN=localhost" \
  -addext "subjectAltName = DNS:localhost" \
  -batch

# 3. Dump the private key
"$OPENSSL" pkey -in keys/ml-dsa-key.pem -text -noout | head

# 4. Print certificate fields
"$OPENSSL" x509 -in keys/ml-dsa-cert.pem -noout -text
```

What each command does in more detail:

- `"$OPENSSL" genpkey -algorithm ML-DSA-65 -out keys/ml-dsa-key.pem` — generate a new ML-DSA-65 key pair and write it as a PKCS#8 private key. The matching public key is stored inside that file; we do not create a separate public-key file. Prints nothing on success.

- `"$OPENSSL" req -new -x509` — build a certificate signing request *and* immediately turn it into a self-signed certificate (the `-x509` switch). Signing uses the private key from `-key`, because a signature needs the private half, not the public half alone. The certificate’s Subject Public Key
  Info field then displays that public key.
  - `-out keys/ml-dsa-cert.pem` — write the certificate here.
  - `-days 365` — validity of one year from the issue time.
  - `-subj "/CN=localhost"` — set the subject common name (and, because this is self-signed, the issuer is the same).
  - `-addext "subjectAltName = DNS:localhost"` — add a SAN. Modern TLS clients match the name against the SAN, not only `CN`.
  - `-batch` — do not prompt for subject fields; we already passed `-subj`.
  
- `"$OPENSSL" pkey -in keys/ml-dsa-key.pem -text -noout | head` — read the private key and print a text dump (`-text`) without writing a new PEM (`-noout`). `head` keeps the start of the dump (algorithm name) and avoids flooding the terminal with hex.

- `"$OPENSSL" x509 -in keys/ml-dsa-cert.pem -noout -text` — read the certificate and print
  its fields: subject, issuer, dates, SAN, public-key algorithm, and signature algorithm.

Check: the certificate text should name ML-DSA-65 for both the public key and the signature. These are disposable lab keys; do not reuse them elsewhere.

#### Expected output (terminal only; the commands create the two `.pem` files in `keys/`)

`"$OPENSSL" genpkey` and `"$OPENSSL" req` print nothing on success. They create these two files in `keys/`:

- `keys/ml-dsa-key.pem` — `BEGIN PRIVATE KEY` / `END PRIVATE KEY`
- `keys/ml-dsa-cert.pem` — `BEGIN CERTIFICATE` / `END CERTIFICATE`

`"$OPENSSL" pkey -text -noout` starts with:

```
ML-DSA-65 Private-Key:
seed:
```

followed by `priv:` and `pub:` hex dumps. Please note that these bytes are unique per run
and are supposed to be secret.

`"$OPENSSL" x509 -noout -text` includes these fields:

```
        Signature Algorithm: ML-DSA-65
        Issuer: CN=localhost
            Not Before: Nov 18 11:12:27 2026 GMT
            Not After : Nov 18 11:12:27 2027 GMT
        Subject: CN=localhost
            Public Key Algorithm: ML-DSA-65
            X509v3 Subject Alternative Name:
                DNS:localhost
    Signature Algorithm: ML-DSA-65
```

Also, the public-key hex after `Public Key Algorithm: ML-DSA-65` is also unique per run; the algorithm names, subject, issuer, and SAN stay the same.

## Lab 2 — TLS 1.3 with hybrid X25519MLKEM768

We start a TLS 1.3 server that presents the Lab 1 certificate, then connect a client to it. The certificate (ML-DSA-65) authenticates the server. The handshake’s key agreement uses the hybrid group X25519MLKEM768 (X25519 plus ML-KEM-768), which is generated during the handshake and is not stored in the certificate.

This lab is the HNDL-urgent half: confidentiality of the session key. `X25519MLKEM768` is a hybrid KEM group, not a signature. Both peers run X25519 and ML-KEM-768 and mix the two shared secrets, so an attacker must break both algorithms to recover the traffic keys. That is why we force `-groups X25519MLKEM768` rather than leaving a classical-only fallback: we want to *see* the hybrid in the negotiated group line, not silently land on X25519 alone. The ML-DSA-65 certificate from Lab 1 still only authenticates the server; it does not protect against HNDL by itself. If this handshake used only X25519, a recording of it could theoretically be decrypted later; with the hybrid group, this is not meant to be.

On OpenSSL 3.5+, both ends offer X25519MLKEM768 by default (NEWS 3.5.0). We still pass `-groups X25519MLKEM768` so the negotiated group is explicit.

[openssl-s_server(1)](https://docs.openssl.org/3.5/man1/openssl-s_server/)
and
[openssl-s_client(1)](https://docs.openssl.org/3.5/man1/openssl-s_client/)
both accept `-groups`. Default listen port is 4433.

#### Terminal 1 (server)

```sh
export OPENSSL=/opt/homebrew/opt/openssl@3/bin/openssl

# Spin up the server
"$OPENSSL" s_server -accept 4433 -www -tls1_3 \
  -cert keys/ml-dsa-cert.pem -key keys/ml-dsa-key.pem \
  -groups X25519MLKEM768
```

What the server command does:

- `"$OPENSSL" s_server` — run a test TLS server (OpenSSL’s built-in demo server).
- `-accept 4433` — listen on port 4433 on this machine.
- `-www` — answer HTTP-like GET requests with a status page; we only need the handshake, but this flag keeps the server in a simple web mode.
- `-tls1_3` — accept only TLS 1.3 (no older protocol versions).
- `-cert keys/ml-dsa-cert.pem` — present the self-signed ML-DSA-65 certificate from Lab 1 to connecting clients.
- `-key keys/ml-dsa-key.pem` — use the matching private key to sign the handshake (CertificateVerify). That is authentication, not key agreement.
- `-groups X25519MLKEM768` — offer only this TLS 1.3 key-agreement group for the handshake’s shared secret.

The server stays running until you stop it (Ctrl-c). It writes no new files; it only uses the two PEM files from Lab 1.

#### Terminal 2 (client)

```sh
export OPENSSL=/opt/homebrew/opt/openssl@3/bin/openssl

# Connect to server through TLS 1.3
printf 'Q\n' | "$OPENSSL" s_client -connect 127.0.0.1:4433 -tls1_3 \
  -groups X25519MLKEM768 -brief \
  -CAfile keys/ml-dsa-cert.pem -verify_return_error
```

- `printf 'Q\n' |` — send the letter `Q` after the handshake so `s_client` quits. `Q` is the s_client command that ends the session.
- `"$OPENSSL" s_client -connect 127.0.0.1:4433` — open a TLS connection to the server we just started on localhost port 4433. This performs the TLS handshake: version negotiation, key agreement, and certificate check.
- `-tls1_3` — speak only TLS 1.3.
- `-groups X25519MLKEM768` — offer only that hybrid group, so the negotiated group cannot silently fall back to something else.
- `-brief` — print a short summary of the established connection instead of a full packet dump (the s_client manual’s SMTP example shows `Protocol version: TLSv1.3`).
- `-CAfile keys/ml-dsa-cert.pem` — trust this file as a certificate authority when verifying the server. Because the certificate is self-signed, the cert is its own trust anchor.
- `-verify_return_error` — abort if verification fails, instead of continuing with an untrusted cert.

Check: the output should show TLS 1.3 and the group X25519MLKEM768. ML-DSA-65 authenticates the server (certificate); X25519MLKEM768 protects the handshake (key agreement).

#### Expected output (terminal 2 only, not saved to a file)

```
Connecting to 127.0.0.1
Can't use SSL_get_servername
CONNECTION ESTABLISHED
Protocol version: TLSv1.3
Ciphersuite: TLS_AES_256_GCM_SHA384
Peer certificate: CN=localhost
Hash used: UNDEF
Signature type: mldsa65
Verification: OK
Negotiated TLS1.3 group: X25519MLKEM768
DONE
```

- `CONNECTION ESTABLISHED` / `Protocol version: TLSv1.3` — the handshake completed as TLS 1.3.
- `Ciphersuite: TLS_AES_256_GCM_SHA384` — the record-protection cipher for the session (symmetric crypto after the handshake).
- `Peer certificate: CN=localhost` — the server presented the Lab 1 certificate.
- `Signature type: mldsa65` — the server proved possession of the ML-DSA-65 private key.
- `Hash used: UNDEF` — expected for ML-DSA: it hashes the message internally with SHAKE256 as part of signing (pure ML-DSA, FIPS 204), so there is no separate hash for OpenSSL to report.
- `Verification: OK` — the client accepted the certificate against `-CAfile`.
- `Negotiated TLS1.3 group: X25519MLKEM768` — key agreement used the hybrid group, not a classical-only group.
- `Can't use SSL_get_servername` — expected here: the client connected to `127.0.0.1` and did not send a server name (SNI).
- `DONE` — `s_client` exited after `Q`.

## Lab 3 — Scanning the endpoint

In this lab we are going to scan the server that we have set up in Lab 2 with nmap, testssl.sh and sslyze, and compare what each external scanner reports about a post-quantum endpoint. As a reminder, our server has an ML-DSA-65 certificate and accepts only the hybrid `X25519MLKEM768` group.

If you have closed the server from the previous lab, run the server again

```sh
export OPENSSL=/opt/homebrew/opt/openssl@3/bin/openssl

"$OPENSSL" s_server -accept 4433 -www -tls1_3 \
  -cert keys/ml-dsa-cert.pem -key keys/ml-dsa-key.pem \
  -groups X25519MLKEM768
```

- **`"$OPENSSL" s_client`** is part of OpenSSL (`"$OPENSSL" s_client ...`). It is a test TLS client that does a real handshake using the OpenSSL 3.5.9 library. In this lab it is our **reference client**, the one we know supports ML-DSA signatures and the `X25519MLKEM768` group. When a scanner reports nothing, `s_client` tells us whether the server is fine and the gap is in the scanner.

### 1. NMAP (`ssl-enum-ciphers`)

**nmap** is a separate network scanner. Its `ssl-enum-ciphers` script builds its own ClientHello in Lua and does not use OpenSSL to connect, so keep in mind that it only offers the algorithms the script itself knows.

#### Run 1 - plain scan

```sh
nmap -p 4433 --script ssl-enum-ciphers 127.0.0.1 -oN logs/nmap-4433-plain.txt
```

#### Output (also in `logs/nmap-4433-plain.txt`)

```
Starting Nmap 7.993 ( https://nmap.org ) at 2026-11-18 13:24 +0200
Nmap scan report for localhost (127.0.0.1)
Host is up (0.00011s latency).

PORT     STATE SERVICE
4433/tcp open  vop

Nmap done: 1 IP address (1 host up) scanned in 1.30 seconds
```

nmap picks scripts by service. Port 4433 isn't a standard TLS port, so nmap guessed `vop` from its port table and never ran `ssl-enum-ciphers`.

#### Run 2 - with service detection on

Note: The following command can be slow because `s_server -www` handles one connection at a time and doesn't answer most of nmap's non-TLS probes, so each one waits for a timeout.

```sh
nmap -sV -p 4433 --script ssl-enum-ciphers 127.0.0.1 -oN logs/nmap-4433.txt
```

- `-sV` probes the port to identify the service, so nmap here eventually discovers TLS.

#### Output (shortened; full version in `logs/nmap-4433.txt`)

```
PORT     STATE SERVICE  VERSION
4433/tcp open  ssl/vop?
| fingerprint-strings:
|   FourOhFourRequest, GetRequest:
|     HTTP/1.0 200 ok
|     s_server -accept 4433 -www -tls1_3 -cert keys/ml-dsa-cert.pem -key keys/ml-dsa-key.pem -groups X25519MLKEM768
|     Ciphers supported in s_server binary
|     TLSv1.3 :TLS_AES_256_GCM_SHA384 TLSv1.3 :TLS_CHACHA20_POLY1305_SHA256
|     ...

Nmap done: 1 IP address (1 host up) scanned in 153.64 seconds
```

- Service detection got through TLS and read the `-www` status page. Its cipher list says "supported in s_server binary": that is what the OpenSSL build can do, not what this server accepts. With `-tls1_3`, none of the TLSv1.2 suites on that page would be accepted.
- There is still no `| ssl-enum-ciphers:` block, so the TLS script reported nothing.

#### Run 3 - forcing the script and turning on debug output

```sh
nmap -Pn -p 4433 --script +ssl-enum-ciphers -d 127.0.0.1
```

- `+ssl-enum-ciphers` runs the script whatever service nmap guesses, with no slow `-sV` step.
- `-Pn` skips host discovery.
- `-d` prints what the script is doing.

#### Output (NSE lines only; terminal only, no `-oN`. The server errors below are in terminal 1)

```
NSE: Starting ssl-enum-ciphers against 127.0.0.1:4433.
NSE: [ssl-enum-ciphers 127.0.0.1:4433] Trying protocol SSLv3.
NSE: [ssl-enum-ciphers 127.0.0.1:4433] Trying protocol TLSv1.3.
NSE: [ssl-enum-ciphers 127.0.0.1:4433] Trying protocol TLSv1.0.
NSE: [ssl-enum-ciphers 127.0.0.1:4433] Trying protocol TLSv1.2.
NSE: [ssl-enum-ciphers 127.0.0.1:4433] Trying protocol TLSv1.1.
NSE: Finished ssl-enum-ciphers against 127.0.0.1:4433.
Completed NSE at 13:32, 1.23s elapsed
```

The order of the `Trying protocol` lines can change from run to run (it differed from our first capture); what matters is that all five are tried.

The script did run and tried every protocol, but no handshake succeeded, and it says nothing about the failure. Luckily, looking at terminal 1, **the server log explains why.** `s_server` printed three kinds of errors:

```
tls_early_post_process_client_hello:unsupported protocol
tls_early_post_process_client_hello:version too low
tls_choose_sigalg:no suitable signature algorithm
```

- `unsupported protocol` and `version too low` are the SSLv3 to TLSv1.2 attempts, which `-tls1_3` refuses as expected.
- `no suitable signature algorithm` is the TLSv1.3 attempt. nmap's own ClientHello, built in its Lua scripting engine rather than with OpenSSL, doesn't list ML-DSA among the signature algorithms it accepts. So the server cannot sign the handshake with its ML-DSA-65 certificate and aborts.
- If you also see `wrong version number` in terminal 1, it comes from Run 2: an `-sV` plain-text probe hitting the TLS port. Clear terminal 1 before Run 3 to keep the two apart.

#### Lessons learnt

1. **False negative.** Against a PQC-only endpoint, nmap 7.993's `ssl-enum-ciphers` returns no result at all. An assessor reading only the scan output could wrongly conclude "no TLS on this port".
2. **Check the other side,** eventually the server log, not the scanner, showed the real cause.
3. **Change one variable at a time** before you conclude anything.
4. **Confirm with a real client,** `s_client` (Lab 2) connected normally to the same server.

#### Run 4 - isolate the variable (ECDSA certificate, hybrid group)

Keep the hybrid-only group and swap the certificate for a classic ECDSA P-256 one. First stop the ML-DSA server in terminal 1 (Ctrl-C), because only one server can listen on port 4433.

> **Caution: unencrypted private keys.** `-nodes` writes `keys/ec-key.pem` without a passphrase, and `genpkey` in Lab 1 wrote `keys/ml-dsa-key.pem` the same way. Anyone who can read these files has the private key. That's acceptable here only because they are throwaway lab keys on your own
> Mac and protect nothing real. Check that only you can read them (`ls -l keys/*-key.pem` should show `-rw-------`, or run `chmod 600 keys/*-key.pem`). Never commit them to git, and never reuse them anywhere. In production, keep keys encrypted with a passphrase from a secrets manager, or in an HSM or cloud key store, so the private key is never on disk in the clear. The lab uses unencrypted keys only so `s_server` can restart without a passphrase prompt.

```sh
export OPENSSL=/opt/homebrew/opt/openssl@3/bin/openssl

# 1. Generate ECDSA keypair
"$OPENSSL" req -new -x509 -newkey ec \
  -pkeyopt ec_paramgen_curve:P-256 -nodes \
  -keyout keys/ec-key.pem -out keys/ec-cert.pem -days 365 -subj "/CN=localhost" \
  -addext "subjectAltName = DNS:localhost"

# 2. Spin up the server with ECDSA
"$OPENSSL" s_server -accept 4433 -www -tls1_3 -cert keys/ec-cert.pem \
  -key keys/ec-key.pem -groups X25519MLKEM768

# 3. Scan with nmap
nmap -Pn -p 4433 --script +ssl-enum-ciphers 127.0.0.1 -oN logs/nmap-4433-ecdsa.txt
```

#### Output (also in `logs/nmap-4433-ecdsa.txt`)

```
Starting Nmap 7.993 ( https://nmap.org ) at 2026-11-18 13:35 +0200
Nmap scan report for localhost (127.0.0.1)
Host is up (0.00022s latency).

PORT     STATE SERVICE
4433/tcp open  vop
| ssl-enum-ciphers:
|   TLSv1.3:
|     ciphers:
|       TLS_AKE_WITH_AES_128_GCM_SHA256 (X25519MLKEM768) - A
|       TLS_AKE_WITH_AES_256_GCM_SHA384 (X25519MLKEM768) - A
|       TLS_AKE_WITH_CHACHA20_POLY1305_SHA256 (X25519MLKEM768) - A
|     cipher preference: client
|_  least strength: A

Nmap done: 1 IP address (1 host up) scanned in 1.55 seconds
```

- `TLS_AKE_WITH_...` is nmap's name for the three TLS 1.3 cipher suites (AKE stands for authenticated key exchange; TLS 1.3 suite names no longer contain the key exchange).
- `(X25519MLKEM768)` is the key-exchange group used with that cipher. nmap can negotiate and report the hybrid group.
- `- A` is nmap's grade (best).
- `cipher preference: client` means the server let the client choose the order.

#### Conclusion

Changing only the certificate fixed the scan, so the ML-DSA signature was the only blocker. nmap 7.993 can confirm a hybrid ML-KEM key exchange (the urgent harvest-now decrypt-later half), but returns nothing when the server has an ML-DSA certificate.

### 2. TESTSSL

We are going to investigate our second scanner, testssl.sh and we are running it against the original ML-DSA server from Lab 2. First stop the ECDSA server from Run 4 in terminal 1 (Ctrl-C), because only one server can listen on port 4433. Then spin up the ML-DSA server again.

```sh
export OPENSSL=/opt/homebrew/opt/openssl@3/bin/openssl

# Stop the ECDSA server first (Ctrl-C), then spin up the ML-DSA server
"$OPENSSL" s_server -accept 4433 -www -tls1_3 \
  -cert keys/ml-dsa-cert.pem -key keys/ml-dsa-key.pem \
  -groups X25519MLKEM768
```

Now run testssl.sh against our server.

```sh
testssl.sh --openssl "$OPENSSL" --logfile logs/testssl-4433-mldsa.log 127.0.0.1:4433
```

- `--openssl "$OPENSSL"` makes testssl.sh use our OpenSSL for the checks it runs through OpenSSL. Many of its checks send raw bytes over its own sockets instead.
- `--logfile` saves the full report as `logs/testssl-4433-mldsa.log`.
- `127.0.0.1:4433` gives the host and port together.

#### Output (key lines only; full report in `logs/testssl-4433-mldsa.log`)

```
 Start 2026-11-18 13:40:11        -->> 127.0.0.1:4433 (127.0.0.1) <<--

 Testing protocols via sockets except NPN+ALPN
 SSLv2      not offered (OK)
 SSLv3      not offered (OK)
 TLS 1      not offered
 TLS 1.1    not offered
 TLS 1.2    not offered
 TLS 1.3    offered (OK): final

TLSv1.3 (listed by strength)
 x1302   TLS_AES_256_GCM_SHA384        ECDH/MLKEM AESGCM    256
 x1303   TLS_CHACHA20_POLY1305_SHA256  ECDH/MLKEM ChaCha20  256
 x1301   TLS_AES_128_GCM_SHA256        ECDH/MLKEM AESGCM    128

 KEMs offered                 X25519MLKEM768
 TLS 1.3 sig_algs offered:    ML-DSA-65
 Signature Algorithm          ML-DSA-65
 Server key size              ML-DSA-65
 Trust (hostname)             certificate does not match supplied URI
 Chain of trust               NOT ok (self signed)

 Running client simulations (HTTP) via sockets
 (35 clients: Android 7-15, Chrome, Chromium, Firefox, IE, Edge, Safari,
  Java 7-21, go, LibreSSL, OpenSSL 1.0.2/1.1.1/3.0, Apple Mail, Thunderbird)
 ...                          No connection   (all except one)
 OpenSSL 3.5.0 (git)          TLSv1.3   TLS_AES_256_GCM_SHA384   X25519MLKEM768

 Overall Grade                T
 Grade cap reasons            Grade capped to T. Issues with chain of trust (self signed)
                              Grade capped to M. Domain name mismatch

 Done 2026-11-18 13:41:06 [0057s] -->> 127.0.0.1:4433 (127.0.0.1) <<--
```

- **testssl.sh sees the whole PQC picture.** It reports TLS 1.3 only, the hybrid KEM `X25519MLKEM768` (shown as `ECDH/MLKEM`), and the ML-DSA-65 signature algorithm and certificate. That's exactly where nmap returned nothing.
- **Client simulations show the compatibility cost.** Only the OpenSSL 3.5 client connects. Every simulated browser, mobile, Java and older OpenSSL client fails, because none of them accepts an ML-DSA certificate and most don't offer the hybrid group either. An ML-DSA-only server is not deployable for public clients today. That's why signatures migrate later
  (or with dual certificates), while hybrid key exchange can go first.
- **Grade T has nothing to do with PQC.** It's capped by the self-signed chain and the name mismatch (we scanned `127.0.0.1`, while the certificate says `localhost`). Always read the grade-cap reasons.
- **No other weaknesses.** Every vulnerability test (Heartbleed, ROBOT, POODLE and the rest) is "not vulnerable", as expected for a TLS 1.3-only server.
- The log header says testssl.sh used OpenSSL 3.5.9 (29 Sep 2026). That's correct: after a `brew upgrade`, `"$OPENSSL" version` reports 3.5.9.

### 3. SSLYZE

#### Run 1 - scan ML-DSA

First we are going to scan the original ML-DSA server (ML-DSA-65 certificate, `-groups X25519MLKEM768`) and see what we get.

```sh
.venv/bin/sslyze 127.0.0.1:4433 --tlsv1_3 --elliptic_curves --certinfo --json_out=logs/sslyze-4433-mldsa.json
```

- `--tlsv1_3` tests the TLS 1.3 cipher suites.
- `--elliptic_curves` tests which key-exchange groups the server accepts.
- `--certinfo` reads and checks the certificate.
- `--json_out` saves machine-readable results.

#### Output (terminal; JSON results in `logs/sslyze-4433-mldsa.json`)

```
 CHECKING CONNECTIVITY TO SERVER(S)
   127.0.0.1:4433  => ERROR: TLS probing failed: could not find a TLS version and cipher suite supported by the server; discarding scan.
```

The JSON shows `ERROR_NO_CONNECTIVITY`, so sslyze ran no scans at all.

Why: sslyze 6.3.1 uses `nassl` 5.4.0, which bundles its own OpenSSL. In our venv that's **OpenSSL 1.1.1w (11 Sep 2023)**, found with 

```sh
strings .venv/lib/python3.13/site-packages/nassl/_nassl.cpython-313-darwin.so | grep -i "openssl 1\."
```

However, OpenSSL 1.1.1 has neither ML-KEM nor ML-DSA, and it's out of support. sslyze can offer neither the hybrid group nor the ML-DSA signature, so the handshake has nothing in common with the server.

#### Run 2 - scan ECDSA

Our next obvious move is to run the scanner against the ECDSA certificate with a classic fallback group. Stop the ML-DSA server with Ctrl-C and spin it up again with the ECDSA certificate from Run 4, as shown below.

```sh
export OPENSSL=/opt/homebrew/opt/openssl@3/bin/openssl

# Stop any running server first (Ctrl-C), then run the server with ECDSA
"$OPENSSL" s_server -accept 4433 -www -tls1_3 -cert keys/ec-cert.pem \
  -key keys/ec-key.pem -groups X25519MLKEM768:X25519

# Scan the ECDSA server with sslyze
.venv/bin/sslyze 127.0.0.1:4433 --tlsv1_3 --elliptic_curves \
  --certinfo --json_out=logs/sslyze-4433-ecdsa-fallback.json
```

#### Output (shortened; full results in `logs/sslyze-4433-ecdsa-fallback.json`)

```
 * TLS 1.3 Cipher Suites:
     The server accepted the following 3 cipher suites:
        TLS_CHACHA20_POLY1305_SHA256   256   ECDH: X25519 (253 bits)
        TLS_AES_256_GCM_SHA384         256   ECDH: X25519 (253 bits)
        TLS_AES_128_GCM_SHA256         128   ECDH: X25519 (253 bits)

 * Elliptic Curve Key Exchange:
       Supported curves:   X25519
       Rejected curves:    X448, brainpoolP256r1, ... secp256r1, secp384r1, secp521r1, sect...

 * Certificates Information:
       Hostname sent for SNI:   127.0.0.1
       Public Key Algorithm:    ECPublicKey   Curve: secp256r1   Key Size: 256
       Trust (all stores):      FAILED - leaf certificate has no matching subjectAltName
```

- **False negative on PQC.** sslyze connects only through the classic X25519 fallback and reports `Supported curves: X25519`. `X25519MLKEM768` appears nowhere, not even among the rejected groups, because sslyze can't test a group it doesn't know. A report built from sslyze would say "no post-quantum key exchange" for a server that actually prefers one.
- `no matching subjectAltName`: sslyze sent `127.0.0.1` as the server name, but the certificate says `localhost`. It's the same name mismatch testssl.sh flagged and has nothing to do with PQC.
- The `CryptographyDeprecationWarning` about a negative serial number comes from a certificate in sslyze's own trust-store bundle, not ours. Our serial is positive. It's harmless.

### Lab 3 summary — three scanners, one PQC server

| Tool                          | TLS engine                           | ML-DSA-65 cert                                            | `X25519MLKEM768` group                                    |
| ----------------------------- | ------------------------------------ | --------------------------------------------------------- | --------------------------------------------------------- |
| nmap 7.993 `ssl-enum-ciphers` | Its own Lua ClientHello              | Not seen. Silent: no output, error only in the server log | Seen and reported, once the certificate was classic ECDSA |
| testssl.sh 3.2.5              | Raw sockets plus local OpenSSL 3.5.9 | Seen (`ML-DSA-65`)                                        | Seen (`KEMs offered X25519MLKEM768`)                      |
| sslyze 6.3.1                  | Bundled nassl, OpenSSL 1.1.1w        | Not seen. Cannot connect at all                           | Not seen. Reports only the classic X25519 fallback        |
| `s_client` (reference)        | OpenSSL 3.5.9                        | Seen (`mldsa65`)                                          | Seen                                                      |

### Lessons learnt

1. **Check the scanner's TLS engine and version before trusting it on PQC.** Of three common scanners, only testssl.sh (with a 3.5+ OpenSSL) saw both PQC algorithms.
2. **An empty or failed scan is not "no TLS".** Cross-check with a reference client (`s_client`) and with the server log.
3. **A classic fallback hides PQC from old scanners.** sslyze reported X25519 only on a server that prefers the hybrid group. Never report "no PQC key exchange" from a scanner that can't negotiate it.
4. **Change one variable at a time.** Swapping only the certificate showed that nmap's problem was the signature, not the group.
5. **PQC-only signatures cut off clients today.** testssl.sh's client simulation found that only OpenSSL 3.5 could connect to the ML-DSA server.
6. **Read grade caps.** testssl.sh's grade T came from the self-signed certificate and the name mismatch, not from PQC.

### Standing rule — always check which crypto library a tool uses

Before trusting any scanner, inventory tool or client on PQC, find out **which cryptographic library it really uses and which version**. What is installed on the machine doesn't matter if the tool carries its own copy. Lab 3 showed three different cases.

| Case | Example | What happens |
|---|---|---|
| Uses the system or configurable OpenSSL | testssl.sh, `s_client` | Points at OpenSSL 3.5.9 and sees PQC |
| Compiles its own library into the tool | sslyze, through nassl (OpenSSL 1.1.1w and 1.0.2e) | Upgrading brew changes nothing. Blind to PQC |
| Writes its own handshake code | nmap `ssl-enum-ciphers` (Lua ClientHello) | Only knows the algorithms its authors listed |

Here is how to check which cryptographic libraries a scanner really uses on macOS.

```sh
nmap --version                      # "Compiled with: ... openssl-x.y.z"

testssl.sh --help | grep -i openssl # its --openssl option picks the binary

"$OPENSSL" version                  # the OpenSSL the lab and testssl.sh use

# A Python package that bundles a C library: find the .so and read its version string
find .venv -name "_nassl*.so" -exec sh -c 'echo "$1"; strings "$1" | grep -E "^OpenSSL [0-9]"' _ {} \;

# A dynamically linked binary: list the libraries it loads
otool -L "$(command -v nmap)" | grep -i -E "ssl|crypto"
```

As a note, write the library and version in every assessment report next to each tool's result, so a reader can tell "server has no PQC" apart from "tool can't see PQC".

## Lab 4 — produce a CBOM

Our goal here is to produce a CycloneDX cryptographic bill of materials (CBOM), a machine-readable inventory of the crypto assets in a system. It is the kind of artefact the [Dutch PQC Migration Handbook](https://english.aivd.nl/documents/2024/12/3/the-pqc-migration-handbook)'s diagnosis step expects.

While Lab 3 checked what the server **presents on the network**, Lab 4 checks what crypto material **sits on disk**. So, in this lab there is no need to run any server. `cbomkit-theia dir` only reads files. An assessment needs both views.

For this task we are going to use **cbomkit-theia v1.1.2** (13 May 2026), part of CBOMkit, donated to the Post-Quantum Cryptography Alliance (PQCA) by IBM Research. It scans a directory or container image for certificates, keys, secrets and `openssl.cnf` files, and outputs CycloneDX 1.6.

### Build 

`cbomkit-theia` must be built from source with Go 1.26.8 or newer (from `go.mod`). The `GOPATH`/`GOCACHE` exports keep Go's downloads inside the lab folder.

```sh
brew install go && go version
mkdir -p tools && cd tools
export GOPATH="$PWD/gopath" GOCACHE="$PWD/gocache"
git clone https://github.com/cbomkit/cbomkit-theia.git
cd cbomkit-theia && git checkout v1.1.2
go mod download && go build
./cbomkit-theia --help
```

- `git checkout v1.1.2`: git warns **"detached HEAD"**. That is normal. A tag points at one fixed commit, not a branch. We only build, so it doesn't matter, and pinning the tag makes the build reproducible.
- `go build` writes a single binary, `tools/cbomkit-theia/cbomkit-theia` (about 39 MB here).
- `./cbomkit-theia` works only from the folder that holds the binary (`./` = "the folder I'm in"). From `pqc-lab`, call `tools/cbomkit-theia/cbomkit-theia`.
- `--help` lists five default plugins: `secrets`, `opensslconf`, `problematicca`, `certificates`, `javasecurity`.
- **Config file:** `~/.cbomkit-theia` is created on the first real scan and is mentioned here because it is the only file that lives outside the lab folder.

### Run

```sh
tools/cbomkit-theia/cbomkit-theia dir . --ignore "tools/,.venv/" > logs/cbom-pqc-lab.json
```

The CBOM goes to stdout (redirected into `logs/cbom-pqc-lab.json`). The log below goes to stderr, i.e. the terminal only, and is not saved to a file.

```text
INFO[0000] Since no BOM is provided as input the java security check is automatically disabled.
WARN[0000] No BOM provided or provided BOM does not have any components, this scan will only add components
INFO[0000] => Running Certificate File Plugin
WARN[0000] failed to parse file that was assumed to be a valid configuration: (Certificate File Plugin) x509: malformed certificate
WARN[0000] failed to parse file that was assumed to be a valid configuration: (Certificate File Plugin) x509: malformed certificate
INFO[0000] Certificate searching done                    numberOfDetectedCertificates=2
ERRO[0000] Error while adding certificate data to bom    error="x.509 certificate has unknown algorithm"
INFO[0000] => Running Secret Detection Plugin
INFO[0000] Secret detected                               file=keys/ec-key.pem type=private-key
INFO[0000] Found 1 private key(s) in keys/ec-key.pem
INFO[0000] Secret detected                               file=keys/ml-dsa-key.pem type=private-key
INFO[0000] Found 1 private key(s) in keys/ml-dsa-key.pem
INFO[0000] => Running OpenSSL Config Plugin
INFO[0000] No OpenSSL configuration files found.
INFO[0000] => Running Problematic CA Detection Plugin
INFO[0000] Problematic CA detection completed            checked=0 flagged=0
```

### What the CBOM contains

`logs/cbom-pqc-lab.json`: CycloneDX 1.6, 5254 bytes, **7 components, all from the ECDSA files**:

| Component            | Asset type              | Detail                                            | From               |
| -------------------- | ----------------------- | ------------------------------------------------- | ------------------ |
| `localhost`          | certificate             | Self-signed, valid 2026-11-18 to 2027-11-18 (UTC) | `keys/ec-cert.pem` |
| ECDSA (listed twice) | algorithm               | Signature, OID 1.2.840.10045.2.1                  | `keys/ec-cert.pem` |
| SHA256               | algorithm               | Hash                                              | `keys/ec-cert.pem` |
| ECDSA-SHA256         | algorithm               | Signature, OID 1.2.840.10045.4.3.2                | `keys/ec-cert.pem` |
| ECDSA public key     | related crypto material | Public key value included (public, so harmless)   | `keys/ec-cert.pem` |
| ECDSA private key    | related crypto material | Type and format only, **no key value**            | `keys/ec-key.pem`  |

`keys/ml-dsa-cert.pem` (7546 B) and `keys/ml-dsa-key.pem` (5604 B) are on disk with normal PEM headers, but **they are not in the CBOM.**

### Why — the log, line by line

- `numberOfDetectedCertificates=2`: it **found** both certificates.
- `ERRO ... x.509 certificate has unknown algorithm`: it couldn't identify ML-DSA, so it **dropped** that certificate, logged an error and still wrote a normal-looking CBOM. 
- `Secret detected file=keys/ml-dsa-key.pem type=private-key`: the secrets plugin **saw** the ML-DSA key, yet it isn't in the CBOM either, and **nothing is logged** about dropping it. That's the worst case: silent.
- `x509: malformed certificate` (twice): two other files looked like certificates but weren't. This log doesn't say which. A `--log-level debug` rerun would show them, but it doesn't change the finding.
- `javasecurity` was off because there was no input BOM, and no `openssl.cnf` was found. Both are expected for this folder.

### Lessons learnt

1. **We have a CBOM, but it is incomplete while looking complete.** It lists the classical assets and drops the PQC ones. A client reading it would conclude "no PQC here", which is wrong.
2. **Always read the tool's log (stderr), not just the report.** The JSON is clean, and only the log shows that ML-DSA was dropped. For the key, even the log only hints at it ("detected", then absent).
3. **Same rule as Lab 3: check the tool's crypto library.** Here the gap is in cbomkit-theia's X.509 handling. In a report, write "tool inventories classical assets only, PQC assets added manually or with a newer tool".
4. **The private-key entry holds no key material** (type and format only), so this CBOM is safe to keep in the repo. Check this every time before committing a CBOM.
5. **The CBOM still does the most important job.** Every quantum-vulnerable asset (ECDSA here) was found with its OID and file, and those are exactly the items that need upgrading. The gap is on the other side: assets that are already PQC are missing, so the tool under-reports migration progress, not risk. State that limit in the report, and add the PQC keys to the key-management inventory by hand.

**Upstream status (checked Nov 2026).** The silent drop of the ML-DSA key is fixed on `main` by [PR #213](https://github.com/cbomkit/cbomkit-theia/pull/213) (merged 26 Aug 2026), but not yet in a release. ML-DSA certificates are tracked in [issue #227](https://github.com/cbomkit/cbomkit-theia/issues/227) and [PR #228](https://github.com/cbomkit/cbomkit-theia/pull/228), and unknown algorithms in general in [issue #235](https://github.com/cbomkit/cbomkit-theia/issues/235). This lab will be re-run when a release includes them.

## Sources

- NIST IR 8547 (final), *Transition to Post-Quantum Cryptography Standards*: https://csrc.nist.gov/pubs/ir/8547/final ; PDF: https://doi.org/10.6028/NIST.IR.8547
- The PQC Migration Handbook, 2nd ed. (AIVD, CWI, TNO, 29 Nov 2024), Ch. 2 Diagnosis: https://english.aivd.nl/documents/2024/12/3/the-pqc-migration-handbook
- OpenSSL 3.5 `genpkey`: https://docs.openssl.org/3.5/man1/openssl-genpkey/
- OpenSSL 3.5 `req`: https://docs.openssl.org/3.5/man1/openssl-req/
- OpenSSL 3.5 `s_server`: https://docs.openssl.org/3.5/man1/openssl-s_server/
- OpenSSL 3.5 `s_client`: https://docs.openssl.org/3.5/man1/openssl-s_client/
- OpenSSL 3.5 NEWS (3.5.0, 8 April 2025): https://raw.githubusercontent.com/openssl/openssl/openssl-3.5/NEWS.md
- Hybrid TLS groups (`X25519MLKEM768`, added in 3.5): https://docs.openssl.org/master/man7/EVP_PKEY-MLX-KEM/
- nmap `ssl-enum-ciphers`: https://nmap.org/nsedoc/scripts/ssl-enum-ciphers.html
- nmap service and version detection (`-sV`): https://nmap.org/book/man-version-detection.html
- nmap NSE `+` prefix (force a script to run): https://nmap.org/book/man-nse.html
- testssl.sh: https://github.com/testssl/testssl.sh
- sslyze: https://github.com/nabla-c0d3/sslyze
- nassl on PyPI (release history, 7.0.0 of 31 Oct 2026): https://pypi.org/project/nassl/ ; source: https://github.com/nabla-c0d3/nassl
- sslyze on PyPI (release history and pinned nassl range): https://pypi.org/project/sslyze/
- cbomkit-theia (v1.1.2, 13 May 2026; README, `go.mod`): https://github.com/cbomkit/cbomkit-theia
- cdxgen (CBOM scope, 12.9.2 on npm): https://github.com/CycloneDX/cdxgen
- CycloneDX 1.6 CBOM guide: https://cyclonedx.org/capabilities/cbom/
