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
| 2 | P4.2 | WordPress install | 2026-04-05 | done |
| 3 | P4.3 | Plugin symlink + activate | 2026-04-05 | done |

---

## #1 -- WAMP vhost: benettcar.local (2026-04-05) -- P4.1

**Cel:** `http://benettcar.local` valaszoljon Apache-bol.

### Mi tortent

1. **Runtime konyvtar letrehozva** (PowerShell):
   ```powershell
   New-Item -ItemType Directory -Path "D:\Projects\spektra\.local\wp-runtimes\benettcar" -Force
   ```
   - Teszt `index.html` elhelyezve (VS Code create file)

2. **Apache vhost hozzaadva** (VS Code szerkesztes):
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

3. **Hosts fajl bovitve** (elevated PowerShell / kezzel):
   ```powershell
   # Admin jogosultsag szukseges!
   Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "`n127.0.0.1    benettcar.local"
   ```

4. **WAMP ujrainditva** (WAMP tray ikon → Restart All Services)

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

## #2 -- WordPress install (2026-04-05) -- P4.2

**Cel:** WordPress runtime a benettcar vhost-ban.

### Mi tortent

Minden parancs **PowerShell**-bol futott (VS Code terminal).

1. **WordPress 6.9.4 letoltve es kicsomagolva:**
   ```powershell
   # Letoltes
   $zipPath = "$env:TEMP\wordpress-latest.zip"
   Invoke-WebRequest -Uri "https://wordpress.org/latest.zip" -OutFile $zipPath -UseBasicParsing

   # Kicsomagolas temp-be
   $extractPath = "$env:TEMP\wp-extract"
   Expand-Archive -Path $zipPath -DestinationPath $extractPath -Force

   # Masolasa runtime konyvtarba (root-ba, nem alkonyvtarba)
   $src = "$env:TEMP\wp-extract\wordpress"
   $dst = "D:\Projects\spektra\.local\wp-runtimes\benettcar"
   Remove-Item "$dst\index.html" -Force
   Copy-Item -Path "$src\*" -Destination $dst -Recurse -Force

   # Temp takaritas
   Remove-Item $zipPath -Force
   Remove-Item $extractPath -Recurse -Force
   ```

2. **Adatbazis letrehozva** (PowerShell → WAMP MySQL):
   ```powershell
   & "D:\Local\wamp\bin\mysql\mysql8.4.7\bin\mysql.exe" -u root -e "
     CREATE DATABASE IF NOT EXISTS sp_benettcar
     CHARACTER SET utf8mb4
     COLLATE utf8mb4_unicode_ci;
   "
   ```
   - Nev: `sp_benettcar` (Spektra naming: `sp_{client_slug}`)

3. **wp-config.php letrehozva** (VS Code create file):
   - DB: `sp_benettcar`, root, ures jelszo, localhost
   - Debug: `WP_DEBUG=true`, `WP_DEBUG_LOG=true`, `WP_DEBUG_DISPLAY=false`
   - Salt generalas:
     ```powershell
     (Invoke-WebRequest -Uri "https://api.wordpress.org/secret-key/1.1/salt/" -UseBasicParsing).Content
     ```
   - Spektra config: **NEM** hozzaadva (Phase 5)

4. **WordPress telepites** (PowerShell → HTTP POST az installerhez):
   ```powershell
   $body = @{
     weblog_title    = 'Benett Car'
     user_name       = 'admin'
     admin_password  = 'admin'
     admin_password2 = 'admin'
     pw_weak         = '1'
     admin_email     = 'dev@benettcar.local'
     blog_public     = '0'
     language         = 'hu_HU'
   }
   Invoke-WebRequest -Uri "http://benettcar.local/wp-admin/install.php?step=2" `
     -Method POST -Body $body -UseBasicParsing
   ```
   - Site title: `Benett Car`
   - Admin: `admin` / `admin` (dev only)
   - Email: `dev@benettcar.local`
   - Search engine visibility: off

### Smoke test

```
http://benettcar.local        --> 200 OK
http://benettcar.local/wp-admin/ --> 200 OK
http://benettcar.local/wp-json/  --> 200 OK
```

### Guardrails betartva

- Nem telepitettem plugint
- Nem telepitettem themet
- Nem adtam hozza Spektra configot
- Nem hoztam letre symlinket
- Nem modositottam wp-content-et

### Statusz

Kesz.

---

## #3 -- Plugin symlink + activate (2026-04-05) -- P4.3

**Cel:** Spektra API plugin elerheto WP-ben Junction-on keresztul, aktivalva, nincs fatal error.

### Mi tortent

Minden parancs **PowerShell**-bol futott (VS Code terminal).

1. **link-plugin.ps1 frissitve** (sp-infra commit):
   - Scaffold lecserelve valos implementaciora
   - `New-Item -ItemType Junction` (nem SymbolicLink -- admin jog nem kell)
   - Idempotens: `Test-Path` guard

2. **Junction letrehozva** (PowerShell):
   ```powershell
   cd D:\Projects\spektra\sp-infra
   & .\scripts\link-plugin.ps1 -Client benettcar
   ```
   Eredmeny:
   ```
   .local\wp-runtimes\benettcar\wp-content\plugins\spektra-api
     --> D:\Projects\spektra\sp-infra\plugin\spektra-api
   ```

3. **Plugin aktivalas** (PowerShell --> WAMP MySQL):
   ```powershell
   echo 'UPDATE wp_options SET option_value=''a:1:{i:0;s:27:"spektra-api/spektra-api.php";}'' WHERE option_name=''active_plugins'';' | & "D:\Local\wamp\bin\mysql\mysql8.4.7\bin\mysql.exe" -u root sp_benettcar
   ```
   Fontos: `s:27` -- a string hossz pontosan 27 karakter!

4. **Permalink struktura** (PowerShell --> WAMP MySQL):
   ```powershell
   & "D:\Local\wamp\bin\mysql\mysql8.4.7\bin\mysql.exe" -u root sp_benettcar -e "UPDATE wp_options SET option_value='/%postname%/' WHERE option_name='permalink_structure'"
   & "D:\Local\wamp\bin\mysql\mysql8.4.7\bin\mysql.exe" -u root sp_benettcar -e "DELETE FROM wp_options WHERE option_name='rewrite_rules'"
   ```

### Bug fix: `const NAMESPACE` (sp-infra commit)

- `class-rest-controller.php`: `const NAMESPACE` --> `const API_NAMESPACE`
- `namespace` PHP reserved keyword -- PHP 8-ban fatal error
- WP error handler elnyelte, plugin csendben nem regisztralodott
- Fix utan: route regisztralodott, endpoint mukodik

### Smoke test

```
http://benettcar.local/wp-json/spektra/v1/site --> 200 OK
Response: {"site":[],"navigation":[],"pages":[]}
```

### sp-infra commit

`49d5a96` -- `feat: link-plugin.ps1 real impl + fix NAMESPACE reserved keyword in rest-controller (P4.3)`

### Statusz

Kesz.
