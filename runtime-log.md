# Spektra -- Runtime Setup Log

Kronologikus naplo a lokalis runtime konfiguracios lepesekrol.
Ezek a lepesek **nem verziokezett** fajlokat erintenek (WAMP config, hosts, WP install, symlink-ek, ACF plugin),
ezert kulon logoljuk oket ide.

**Remote:** https://github.com/Csharlie/sp-docs
**Scope:** Phase 4 (WordPress Runtime Assembly) es kesobb barmi, ami host-specifikus runtime setup

---

## Naplo

| # | Phase | Lepes | Datum | Statusz |
|---|-------|-------|-------|---------|
| 1 | P4.1 | WAMP vhost | 2026-04-05 | done |

---

## #1 -- WAMP vhost: benettcar.local (2026-04-05) -- P4.1

**Cel:** `http://benettcar.local` valaszoljon Apache-bol.

### Mi tortent

1. **Runtime konyvtar letrehozva:**
   - `D:\Projects\spektra\.local\wp-runtimes\benettcar\`
   - Teszt `index.html` elhelyezve

2. **Apache vhost hozzaadva:**
   - Fajl: `D:\Local\wamp\bin\apache\apache2.4.65\conf\extra\httpd-vhosts.conf`
   - Meglevo localhost blokk erintetlen, uj blokk utana:
   ```apache
   <VirtualHost *:80>
     ServerName benettcar.local
     DocumentRoot "D:/Projects/spektra/.local/wp-runtimes/benettcar"
     <Directory "D:/Projects/spektra/.local/wp-runtimes/benettcar/">
       Options FollowSymLinks
       AllowOverride All
       Require all granted
     </Directory>
   </VirtualHost>
   ```

3. **Hosts fajl bovitve:**
   - `C:\Windows\System32\drivers\etc\hosts`
   - `127.0.0.1    benettcar.local`

4. **WAMP ujrainditva** (tray ikon)

### Architekturalis dontesek

- **DocumentRoot NEM `wamp/www/` alatt** -- hanem `D:\Projects\spektra\.local\wp-runtimes/benettcar`
  - Runtime a Spektra projekt fan belul marad
  - `.local/` gitignored
  - Multi-client ready (kesobbi kliensek: `.local/wp-runtimes/<slug>/`)
- **`Require all granted`** a `Require local` helyett -- Vite dev server fetch, REST API, CORS mind mukodjon
- **`Options FollowSymLinks`** -- Indexes kihagyva (WP-nel felesleges directory listing)

### Smoke test

```
Invoke-WebRequest http://benettcar.local --> 200 OK
"benettcar.local works"
```

### Statusz

Kesz.

---

## #2 -- WordPress install (pending) -- P4.2
