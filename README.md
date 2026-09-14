# Plex on PlayStation 5 Pro — local playback without a subscription

Your media server sits in the next room, your console is on the same Wi-Fi, and Plex still asks you
to pay for "remote" playback. This is the full path to getting it working **locally** — what to do,
how to verify each step, and, just as importantly, **what not to waste your time on**.

Tested on a **PlayStation 5 Pro**, Plex app `1.007.001`, Plex Media Server `1.43.4.10903`
(`lscr.io/linuxserver/plex`, `network_mode: host`) on Debian 13, September 2026.

The same path applies to any client with a minimal TLS stack — Xbox, Roku, older Android TV,
LG webOS — they behave identically here.

> **Before anything else, update Plex Media Server.** This guide documents server behaviour as of
> September 2026. If Plex has since changed how it builds the certificate chain, a plain update is
> the whole fix.

---

## Start here: match your symptom

You probably arrived with one specific thing on screen. Find it below.

| What you're seeing | What it means | Go to |
|---|---|---|
| **"Remote Playback requires a Remote Watch Pass"** — on your own home network | The console never established a local connection and fell back to the remote one | [§1 — do this first](#1-do-this-first) |
| Server is listed but marked **Remote** | Same cause as above | [§1](#1-do-this-first) |
| **Works on browser and phone, fails only on the console** | Classic signature of this whole situation | [§1](#1-do-this-first), then [§3 for why](#3-why-this-happens) |
| `tlsv1 alert unknown ca` in the server log | The console rejected the server's certificate — the chain is incomplete | [§2 — confirm it](#2-confirming-you-are-in-the-right-place) |
| `CERT: incomplete TLS handshake from <console-ip>` | Same line, same cause | [§2](#2-confirming-you-are-in-the-right-place) |
| Console **doesn't see the server at all** | May be something else entirely — check the chain first | [§2](#2-confirming-you-are-in-the-right-place) |
| You already changed DNS on the console and nothing happened | Expected — DNS is not the cause here | [§4 — things that don't work](#4-what-not-to-do) |
| You already set up your own certificate and the paywall stayed | Also expected, and it's by design | [§4](#4-what-not-to-do) |
| It worked before and broke on its own | Certificate rotation — same path | [§8](#8-after-certificate-rotation) |
| You deleted the certificate and now the server has none | You hit the rate limit — restore the backup | [§5](#5-the-one-thing-that-can-actually-hurt-you) |

Short version: **a certificate chain problem shows up as a billing problem.** That disconnect is why
this is so hard to search for.

---

## 1. Do this first

Plex caches its `plex.direct` certificate on disk. Deleting it makes the server request a fresh one —
this time with a complete chain.

**Where the cache lives:**

| Setup | Path |
|---|---|
| Docker (`linuxserver/plex`) — on the host | `<your-volume>/Library/Application Support/Plex Media Server/Cache/` |
| Docker — inside the container | `/config/Library/Application Support/Plex Media Server/Cache/` |
| Linux (native package) | `/var/lib/plexmediaserver/Library/Application Support/Plex Media Server/Cache/` |
| macOS | `~/Library/Caches/PlexMediaServer/` |
| Windows | `%LOCALAPPDATA%\Plex Media Server\Cache\` |

The file is `cert-v2.p12` on current versions, `certificate.p12` on older ones. Remove whichever
exists — and **back it up first**, you will want it if anything goes wrong.

```bash
CACHE="./config/plex/Library/Application Support/Plex Media Server/Cache"

docker compose stop plex

# back up whatever is there
for f in cert-v2.p12 certificate.p12; do
  [ -f "$CACHE/$f" ] && cp -a "$CACHE/$f" "$CACHE/$f.bak"
done

rm -f "$CACHE/cert-v2.p12" "$CACHE/certificate.p12"
docker compose up -d plex
```

> **Run this once.** Each deletion triggers a re-issue, and repeating it hits a Let's Encrypt rate
> limit that leaves you with no certificate at all. Read [§5](#5-the-one-thing-that-can-actually-hurt-you)
> before you consider a second attempt.

Then reopen Plex on the console. If it remembers the old state, sign out and back in via
`plex.tv/link`.

Verify the result with [§6](#6-verifying-the-result) before assuming it worked.

---

## 2. Confirming you are in the right place

Symptoms that point here:

- The console shows the server as unavailable, or connects but labels it **Remote**
- Playback is blocked by **"Remote Playback requires a Remote Watch Pass"** — on your own network
- **Every other client works fine**: browser, phone, desktop
- The server log repeats:

```
CERT: incomplete TLS handshake from 192.168.1.50:62749: tlsv1 alert unknown ca (SSL routines)
```

### Find your plex.direct hostname

Every server has its own 32-character hash. Ask the server itself:

```bash
openssl s_client -connect <server-ip>:32400 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject
# subject=CN=*.01c00d0663104e03b8de1fa2a2b4baf5.plex.direct
#             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ this is your hash
```

Your local hostname is then your LAN IP with dots replaced by dashes, plus that hash:
`192-168-1-100.<hash>.plex.direct`

### Inspect the chain

```bash
HASH=01c00d0663104e03b8de1fa2a2b4baf5      # yours, from above
IP_DASHED=192-168-1-100                     # your server's LAN IP, dashed

openssl s_client -connect <server-ip>:32400 \
  -servername $IP_DASHED.$HASH.plex.direct -showcerts </dev/null 2>/dev/null \
  | grep -E '^ [0-9] s:|^   i:'
```

**Two certificates — this guide applies to you:**

```
 0 s:CN=*.<hash>.plex.direct
   i:C=US, O=Let's Encrypt, CN=YR2
 1 s:C=US, O=Let's Encrypt, CN=YR2
   i:C=US, O=ISRG, CN=Root YR
```

The chain stops at `Root YR`, which no client trusts on its own.

**Three certificates, the last one issued by `ISRG Root X1` — your chain is already fine**, and your
problem is something else.

The intermediate may be `YR1`, `YR2` or `YR3` (or the `YE*` family for ECDSA keys) — what matters is
where the chain *ends*, not which intermediate signed the leaf.

---

## 3. Why this happens

On **2026-05-13** Let's Encrypt switched its default issuance profile to the
[Generation Y hierarchy](https://letsencrypt.org/2025/11/24/gen-y-hierarchy.html). Plex obtains
certificates for `*.plex.direct` on your server's behalf — Plex Media Server generates a key and
uploads a CSR to plex.tv, and plex.tv gets the certificate from the CA and hands it back — so servers
started receiving certificates from the new hierarchy automatically.

The new roots — **ISRG Root YR** and **ISRG Root YE** — are not yet present in any trust store;
Let's Encrypt says so directly on its [Chains of Trust](https://letsencrypt.org/certificates/) page.
Compatibility comes from **cross-signing**: `ISRG Root YR` is also signed by the long-trusted
`ISRG Root X1`.

A complete chain therefore looks like this:

```
leaf (*.<hash>.plex.direct)
  └── Let's Encrypt YR2
        └── ISRG Root YR   (cross-signed by ISRG Root X1)   ← the server must send this
              └── ISRG Root X1   (already in every trust store)
```

When only the first two are sent, a client has nothing to anchor the path to.

### Why only consoles and TVs are affected

Windows, macOS and iOS perform **AIA chasing** — if a link is missing they fetch it themselves via
the Authority Information Access extension. Minimal TLS stacks in consoles and smart TVs do not: they
expect the complete chain from the server and fail closed with `unknown ca`.

Android's system TrustManager does *not* do AIA chasing either — Plex's mobile apps appear to ship
their own trust anchors, which is the likely reason they keep working. Either way, the console has no
such escape hatch.

That is why the browser on your PC works while the PS5 Pro three metres away does not.

### Why it surfaces as a paywall

This chain of consequences is what makes the problem so confusing to diagnose:

1. The console cannot complete TLS to the **local** address → it has no usable local connection
2. It falls back to the **remote** connection published on plex.tv
3. Remote playback on consoles requires a paid pass → paywall appears

**The paywall is the last link in the chain, not the cause.** Attacking it directly leads nowhere —
see the next section.

---

## 4. What not to do

Everything below was actually attempted and measured during this investigation. None of it helps.
Listed so you can skip an entire evening.

| Attempt | What actually happens |
|---|---|
| Change DNS on the console to `8.8.8.8` / `1.1.1.1` | No effect. The router already resolves `plex.direct` to the private IP — verify with `nslookup` before believing otherwise. |
| Disable DNS Rebinding Protection on the router | Not the cause. `unknown ca` occurs **after** DNS and TCP have already succeeded. |
| Set `secureConnections` to Disabled | The PS5 Pro never attempts HTTP. Confirmed with `tcpdump`: 24 TLS attempts, zero HTTP. |
| Look for "Allow insecure connections" in the console app | No such setting in app `1.007.001` — there is no network section at all. |
| `customConnections` (Custom server access URLs) | Silently ignored in PMS 1.43.4 — the value is stored but never published to plex.tv. |
| Your own certificate via `customCertificateDomain` | TLS succeeds and the console connects, but the connection is always flagged `local=false`, so the paywall stays. |
| Point that domain's A record at the private IP | The flag does not change. |
| Use a subdomain shaped like a dashed IP (`192-168-1-100.example.com`) | plex.tv does not parse it. Still `local=false`. |
| Disable Relay, Remote Access or manual port mapping | Removes the remote connection together with the only one the console can actually establish. |
| Remove the console from your account, reinstall the app | No effect — the flag is recalculated on every server start, not cached. |
| Stream to the PS5 Pro over DLNA | It is **not** a DLNA renderer. Its SSDP replies on port 40001 belong to Remote Play / Second Screen. |
| Look for the "Media Player" app in PS Store | It does not exist for PS5 — that was a PS4 app. Media Gallery plays from USB only. |

### Why the custom-certificate route can never work

Worth understanding, because it looks like the obvious solution.

PMS does **not** send a `local` flag. It publishes a flat list of URIs:

```
PUT https://servers.plex.tv/devices/{machineIdentifier}
    ?Connection[][uri]=http://192.168.1.100:32400
    &Connection[][uri]=http://172.17.0.1:32400
    &httpsEnabled=1&httpsRequired=0&dnsRebindingProtection=0
```

The **backend** classifies each entry by parsing the host:

| What PMS sends | plex.tv verdict |
|---|---|
| RFC1918 IP literal (`192.168.x`, `10.x`, `172.16–31.x`) | `local=1`, generates `https://<dashed-ip>.<hash>.plex.direct:port` |
| Public IP literal | `local=0`, also generates a plex.direct name |
| CGNAT `100.64.0.0/10` (Tailscale) | dropped entirely — never appears in `/api/v2/resources` |
| **Any hostname or FQDN** | `local=0`, no plex.direct name generated |

No DNS resolution takes place. A domain is not an IP literal, so it can never be classified local —
no matter where it points or how it is spelled. No Preferences.xml setting changes this;
`LanNetworksBandwidth` affects only server-side bandwidth accounting.

Getting your own certificate for `plex.direct` is not an option either. Its CAA record reads:

```
0 issue ";"
0 issuewild "letsencrypt.org"
0 issuewild "digicert.com"
```

Non-wildcard issuance is forbidden to every CA, and wildcard issuance is restricted to two — neither
of which will issue to you, since proving control over the domain requires access to Plex's DNS zone.

---

## 5. The one thing that can actually hurt you

Each deletion of the certificate triggers a re-issue. Repeat it several times and you hit
**HTTP 429**, leaving the server with **no certificate at all**. The limit belongs to the CA — Plex
can reset its own counters, but not the CA's. Reports on the Plex forum suggest waiting roughly a
week. People have broken their TLS this way while debugging.

**Delete once. If it doesn't help, go to [§7](#7-if-the-quick-fix-doesnt-help) — don't retry.**

### If you already have no certificate

Put the backup from §1 back:

```bash
CACHE="./config/plex/Library/Application Support/Plex Media Server/Cache"

docker compose stop plex
cp -a "$CACHE/cert-v2.p12.bak" "$CACHE/cert-v2.p12"    # or certificate.p12.bak
docker compose up -d plex
```

This restores the *old, incomplete* chain — the console still won't work, but the server is
serviceable for every other client while you wait out the limit.

Docker users: make sure `Cache/` lives on a **persistent volume**. If it is ephemeral, the
certificate is destroyed on every container restart, which guarantees hitting the rate limit sooner
or later.

---

## 6. Verifying the result

**The chain** — three certificates now, the last one issued by ISRG Root X1:

```
 0 s:CN=*.<hash>.plex.direct
   i:C=US, O=Let's Encrypt, CN=YR2
 1 s:C=US, O=Let's Encrypt, CN=YR2
   i:C=US, O=ISRG, CN=Root YR
 2 s:C=US, O=ISRG, CN=Root YR
   i:C=US, O=Internet Security Research Group, CN=ISRG Root X1     ← the cross-signature
```

**The decisive test** — validate against *only* ISRG Root X1, which is what a console has:

```bash
openssl s_client -connect <server-ip>:32400 \
  -servername $IP_DASHED.$HASH.plex.direct \
  -CAfile /etc/ssl/certs/ISRG_Root_X1.pem </dev/null 2>/dev/null \
  | grep "Verify return code"
# want: Verify return code: 0 (ok)
```

**The console stopped failing** — this counter should stay at zero:

```bash
grep -c 'incomplete TLS handshake from <console-ip>' \
  "<config>/Library/Application Support/Plex Media Server/Logs/Plex Media Server.log"
```

**Playback is genuinely local** — start a film and check the session:

```bash
curl -s -H "X-Plex-Token: <token>" "http://<server-ip>:32400/status/sessions" \
  | grep -oE 'local="[01]"'
# want: local="1"
```

---

## 7. If the quick fix doesn't help

Some servers re-issue and still get a truncated chain. You can repair it by hand: extract the
certificate and key, append the cross-signature yourself, repackage. The p12 password is derived
deterministically from the machine identifier.

```bash
umask 077        # the private key is about to be written in the clear

P="<config>/Library/Application Support/Plex Media Server"
MID=$(sed -n 's/.*ProcessedMachineIdentifier="\([^"]*\)".*/\1/p' "$P/Preferences.xml")
PASS=$(printf '%s' "plex${MID}" | openssl dgst -sha512 | cut -d' ' -f2)   # printf, not echo

openssl pkcs12 -in "$P/Cache/cert-v2.p12" -passin "pass:$PASS" -nodes -nocerts -out key.pem
openssl pkcs12 -in "$P/Cache/cert-v2.p12" -passin "pass:$PASS" -nodes -nokeys  -out leafchain.pem

curl -fsSLO https://letsencrypt.org/certs/gen-y/root-yr-by-x1.pem

# leafchain.pem already holds leaf + intermediate; -certfile adds ONLY the missing cross-signature
openssl pkcs12 -export -out fixed.p12 -in leafchain.pem -inkey key.pem \
  -certfile root-yr-by-x1.pem -passout "pass:$PASS" \
  -certpbe AES-256-CBC -keypbe AES-256-CBC -macalg SHA256
```

Check you got exactly three certificates, not five:

```bash
openssl pkcs12 -in fixed.p12 -passin "pass:$PASS" -nokeys -info 2>/dev/null \
  | grep -c "BEGIN CERTIFICATE"
# want: 3
```

Then install it:

```bash
docker compose stop plex
cp -a "$P/Cache/cert-v2.p12" "$P/Cache/cert-v2.p12.bak"
cp fixed.p12 "$P/Cache/cert-v2.p12"
chown 1000:1000 "$P/Cache/cert-v2.p12"      # match PUID/PGID of your container
chmod 600 "$P/Cache/cert-v2.p12"
docker compose up -d plex

shred -u key.pem 2>/dev/null || rm -f key.pem
```

Verify with [§6](#6-verifying-the-result).

Notes:

- On **OpenSSL 1.1.1 and older** the `-certpbe / -keypbe / -macalg` flags are mandatory — without
  them PMS cannot read the file. On OpenSSL 3.x they are already the defaults, but harmless to keep.
- PMS **overwrites this file** when the certificate rotates (~90 days), so this needs a timer to
  survive. See [§8](#8-after-certificate-rotation).

It is also worth reporting to Plex: the fix on their side is simply to include `root-yr-by-x1.pem`
in the served chain, which would resolve it for every affected device at once.

---

## 8. After certificate rotation

The `plex.direct` leaf renews automatically roughly every 90 days. If Plex has not changed its
behaviour by then, a truncated chain may come back. The warning sign is the same log line
reappearing:

```
CERT: incomplete TLS handshake from <console-ip>: tlsv1 alert unknown ca
```

Check the chain with the command from [§2](#2-confirming-you-are-in-the-right-place) — and remember
the rate limit before deleting anything.

---

## References

- [Let's Encrypt — Chains of Trust](https://letsencrypt.org/certificates/)
- [Let's Encrypt — New "Generation Y" Hierarchy](https://letsencrypt.org/2025/11/24/gen-y-hierarchy.html)
- [Cross-signed ISRG Root YR](https://letsencrypt.org/certs/gen-y/root-yr-by-x1.pem)
- [Plex forum — PS5 Local Connection Failing (unknown ca)](https://forums.plex.tv/t/ps5-local-connection-failing-cert-incomplete-tls-handshake-unknown-ca-all-other-devices-work/940263)
- [Plex forum — PS5 cannot connect to local Plex server](https://forums.plex.tv/t/ps5-cannot-connect-to-local-plex-server-tls-handshake-fails-unknown-ca/940811)
- [Plex forum — Certificate stuck on 429](https://forums.plex.tv/t/certificate-stuck-on-429-request-reset-certs-issued-but-not-stored-cause-fixed/942464)
- [Plex forum — Decrypting the plex.direct certificate](https://forums.plex.tv/t/decrypting-the-plex-direct-certificate/566027)
- [Plex Support — Network settings](https://support.plex.tv/articles/200430283-network/)
- [How Plex is doing HTTPS for all its users](https://words.filippo.io/how-plex-is-doing-https-for-all-its-users/)

---

🇺🇦 [Українською](README.uk.md)
