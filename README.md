# 👻 Spectre - Secure P2P Terminal

**[ 🇬🇧 English ]** Spectre is a zero-trust, end-to-end encrypted (AES-256-GCM) P2P web terminal with a built-in QR scanner. It runs 100% in the browser, with no backend and no database, and leaves no trace.

**[ 🇹🇷 Türkçe ]** Spectre; dahili QR okuyuculu, sıfır güven mimarili, uçtan uca şifreli (AES-256-GCM) bir P2P web terminalidir. Sunucu ve veritabanı olmadan %100 tarayıcıda çalışır, iz bırakmaz.

**[ 🇪🇸 Español ]** Spectre es un terminal web P2P de confianza cero con cifrado de extremo a extremo (AES-256-GCM) y escáner QR integrado. Funciona 100 % en el navegador, sin servidor ni base de datos, y no deja rastro.

**[ 🇫🇷 Français ]** Spectre est un terminal web P2P zéro confiance, chiffré de bout en bout (AES-256-GCM), avec scanner QR intégré. Il fonctionne à 100 % dans le navigateur, sans serveur ni base de données, et ne laisse aucune trace.

**[ 🇩🇪 Deutsch ]** Spectre ist ein Zero-Trust-P2P-Webterminal mit Ende-zu-Ende-Verschlüsselung (AES-256-GCM) und integriertem QR-Scanner. Es läuft zu 100 % im Browser, ohne Server und Datenbank, und hinterlässt keine Spuren.

**[ 🇷🇺 Русский ]** Spectre — P2P веб-терминал с нулевым доверием и сквозным шифрованием (AES-256-GCM) со встроенным QR-сканером. Работает на 100% в браузере, без сервера и базы данных, и не оставляет следов.

**[ 🇸🇦 العربية ]** Spectre محطة ويب P2P بلا ثقة ومشفرة من طرف إلى طرف (AES-256-GCM) مع ماسح QR مدمج. تعمل بالكامل في المتصفح دون خادم أو قاعدة بيانات ولا تترك أثراً.

---

## Overview

Spectre opens a short-lived, encrypted channel between two devices without registration, backend or database. Messages travel through a public MQTT broker that only ever sees ciphertext.

> **Status:** not independently audited. Read [Threat model](#threat-model) before relying on it.

## Security architecture

| Layer | Implementation |
|---|---|
| Session code | 7 characters: 1 broker marker (`H` HiveMQ, `E` EMQX, `M` Mosquitto, `C` custom) + 6 random characters from a 31-symbol alphabet (CSPRNG, rejection sampling, no `I L O 0 1`). |
| Rendezvous | PBKDF2-SHA-256, **600,000 iterations**, derives the handshake key and the MQTT topic id. The raw code never appears on the broker. |
| Key agreement | Ephemeral **ECDH P-256** per session, then **HKDF-SHA-256**. Forward secrecy: keys live in RAM only. |
| Approval secret | The host releases a random 256-bit secret only when it approves. Mixed into HKDF, so an unapproved party that knows the code cannot derive the chat key. |
| Message encryption | **AES-256-GCM**, random 96-bit IV. Sender id, random message id, sequence number and display name are **inside** the ciphertext. |
| Replay / reflection / spoofing | Message-id dedupe, pinned peer id, strictly increasing sequence numbers. A gap shows a "message missing" warning. |
| Man-in-the-middle | Both screens show the same **10-digit security code** (hash of both public keys) **before** the host approves. Mismatch button destroys the session. Optional gate blocks messaging until the user confirms the match. |
| Control traffic | Join/leave/panic/typing/heartbeat are encrypted and authenticated too. Pre-chat only the pending guest may end a room. |
| Images | Downscaled and re-encoded as JPEG on a `<canvas>` before encryption. Receiver re-validates: JPEG header, max 4096 px / 12 MP, max 2 MB. |
| UI | Names and messages are rendered as text nodes only (`textContent`). Strict CSP, `no-referrer`, frame-busting. |
| Storage | Nothing is persisted except theme, language and two UI toggles. No chat, key or session data touches `localStorage`/`sessionStorage`. |
| Link | `https://host/#CODE`. The fragment is never sent to a server. It is removed from the address bar after reading. |

## Self-destruct features

- **Panic Mode:** tap the screen **5 times rapidly** (≤ 0.8 s between taps). The local chat, keys and DOM are wiped instantly and the app returns to the home screen with a welcome message. A signed **panic order** (QoS 1, retained) tells the peer to wipe too. A peer that is offline or backgrounded receives it when it reconnects.
- **Dead-man switch:** both sides send an encrypted heartbeat every 15 s. If the peer is silent for 60 s the session is destroyed locally, so the room still disappears if a panic order is lost.
- **Screen lock / background wipe** (toggle, default on): when the page becomes hidden the chat is destroyed immediately. Picking an image is exempt for 2 minutes. With the toggle off the screen is covered and the session dies after 60 s hidden.
- **Room hygiene:** host room expires after 10 min, approval prompt auto-rejects after 60 s, guest gives up after 90 s, rejections back off (15 s, 30 s) and the 3rd rejection closes the room. One guest per room.

## How to use

### Host
1. Open Spectre, enter a name, press **Create Secure Node**.
2. Show the QR code (or share the 7-character code).
3. When a guest asks to join, **compare the security code** shown in the prompt with the guest's screen. Accept only if identical.

### Guest
1. Open Spectre, enter a name, scan the host's QR (or type the code) and press **Connect**.
2. Your own security code appears **while waiting**. Tell/show it to the host.
3. After approval, confirm **Match** in the amber bar (or **Mismatch** to destroy the session).

### Settings (home screen)
- *Block messaging until security code is verified* (default on).
- *Destroy chat instantly when the screen locks or the app leaves the foreground* (default on).
- *Advanced → own MQTT broker* (`wss://…`). The guest must use the same broker; the QR link carries it. Using this requires `connect-src wss:` in the CSP, which is already set.

## Threat model

**Protects against:** a passive observer of the public broker, a broker operator, a third party who learns or brute-forces the session code (they can request to join but need host approval and cannot read the chat), replayed/reflected/forged messages, oversized or malformed images, XSS via names/messages, clickjacking, and casual shoulder surfing (Panic Mode).

**Does not protect against:**
- A malicious or compromised device, browser extension, keylogger, screenshot or screen recording.
- Users who skip the security-code comparison (active MITM goes unnoticed).
- Metadata: a public broker sees who connects, when and message sizes.
- Availability attacks: flooding the broker or occupying the approval slot.
- Supply-chain attacks on CDN scripts until you vendor them (see below).
- The session code has about 30 bits of entropy. A well-funded attacker can recover it offline from a captured topic. That only lets them attempt to join. The chat itself stays protected by ECDH, approval and the security code.
- **Web limits:** browsers expose no "screen locked" event. "Page hidden" is the signal, so app and tab switches trigger it too. Mobile OSes may freeze the socket right away, so the *remote* wipe on lock is best-effort. Memory cannot be zeroed from JavaScript.

## Hardening checklist for deployment

1. Vendor `mqtt.js`, `qrcodejs`, `html5-qrcode`; compile Tailwind instead of the Play CDN; add SRI hashes (`curl -s URL | openssl dgst -sha384 -binary | openssl base64 -A`). Then remove external origins and `'unsafe-inline'` from the CSP.
2. Prefer a host that sends real headers (Cloudflare Pages / Netlify `_headers`). A `<meta>` CSP cannot set `frame-ancestors`:
   ```
   /*
     Content-Security-Policy: default-src 'none'; script-src 'self'; style-src 'self'; img-src 'self' data: blob:; connect-src 'self' wss:; object-src 'none'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'
     Referrer-Policy: no-referrer
     Permissions-Policy: camera=(self), microphone=(), geolocation=()
     X-Content-Type-Options: nosniff
   ```
3. Run your own MQTT broker (Advanced setting) to remove the public-broker metadata exposure.
4. Check advisories for the pinned third-party libraries.

## Tests

```bash
npm test      # Node 18+, no dependencies
```

`tests/security.test.mjs` extracts the real crypto/protocol code from `index.html` and runs it in isolated contexts (host, guest, attacker). It covers the handshake, wrong-code and tamper rejection, MITM detection via the security code, approval-secret binding, replay, reflection, spoofed sender, sequence gaps, panic order (QoS 1 + retained, encrypted, sender-pinned), image-bomb guard, name sanitising, code generation, custom-broker validation, send gate, reject back-off, plus static checks (CSP, no `eval`, 5-tap panic, no persistence) and i18n key parity across all 7 languages. CI runs it on every push (`.github/workflows/test.yml`).

Not covered: DOM wiring, real-browser CSP behaviour, visibility/lock events on real devices, QR scanning, and delivery on a real broker. Test those on two real devices before release.

## Tech stack
- **Frontend:** HTML5, vanilla JavaScript (ES2020), single `index.html`
- **Styling:** Tailwind CSS (CDN)
- **Crypto:** Web Crypto API (PBKDF2, ECDH, HKDF, AES-GCM)
- **Transport:** MQTT over WebSockets (`mqtt.js`) on HiveMQ `:8443`, EMQX `:8084`, Mosquitto `:8081`, or your own `wss://` broker
- **QR:** `qrcodejs`, `html5-qrcode`
- **Languages:** English, Türkçe, Español, Français, Deutsch, Русский, العربية (full RTL)

## Deployment (GitHub Pages)
1. Fork this repository.
2. **Settings → Pages**, source: `main` branch.
3. Live at `https://<user>.github.io/<repo>/`.

## ⚠️ Disclaimer & Acceptable Use

**[🇬🇧]** Spectre is provided "as is" for educational purposes, security research and legitimate privacy needs, without warranty and without independent security audit. The developer assumes no responsibility or liability for misuse, illegal activities or damage caused by this software. Users are fully responsible for ensuring their communications comply with all applicable local, state and international laws.

**[🇹🇷]** Spectre; eğitim, güvenlik araştırmaları ve meşru gizlilik ihtiyaçları için, garanti ve bağımsız güvenlik denetimi olmaksızın "olduğu gibi" sunulmaktadır. Geliştirici, bu yazılımın kötüye kullanımından, yasadışı faaliyetlerde kullanılmasından veya verebileceği zararlardan sorumlu tutulamaz. Kullanıcılar, iletişimlerinin yerel ve uluslararası yasalara uygun olmasından tamamen kendileri sorumludur.
