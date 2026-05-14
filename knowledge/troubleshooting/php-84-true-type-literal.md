---
title: PHP 8.4 — `true|\WP_Error` return type literal parses as class reference
type: troubleshooting
status: resolved
created: 2026-05-14
last_updated: 2026-05-14
related_commit: 2202032 (sp-infra)
related_phase: P14.7
---

# PHP 8.4 — `true|\WP_Error` return type literal parses as class reference

## Tünet

A `spektra-api` plugin aktiválásakor (vagy bármilyen plugin-load eseménynél) **fatal error** PHP 8.4-en:

```
Fatal error: Uncaught Error: Class "Spektra\API\true" not found
  in /wp-content/plugins/spektra-api/includes/class-rest-controller.php:48
```

A WP admin felület 500-ast dob, vagy white screen-t mutat. A `wp-content/debug.log`-ban a fenti hibaüzenet látszik.

## Ok

A `class-rest-controller.php` `validate_preview_param()` metódusa **`true` literal-t** használt return type-ként:

```php
namespace Spektra\API;

class Rest_Controller {
    public static function validate_preview_param( string $value ): true|\WP_Error {
        // ...
    }
}
```

A **`true` és `false` literal types** PHP 8.2-ben lettek bevezetve, és **PHP 8.4 előtt** mindenhol literál típusként parse-olódtak (intersect / union type kontextusban is).

**PHP 8.4 viselkedési változás:** bizonyos union type pozíciókban a `true` token **újra class-referenciaként** parse-olódik (visszamenőlegesen kompatibilis class autoloaderrel rendelkező kódbázisokkal). Mivel a fájl `namespace Spektra\API;` alá van rendelve, a PHP a `true`-t `Spektra\API\true`-ként próbálja autoload-olni — ami nem létezik → fatal.

> Megjegyzés: ez **nem dokumentált breaking change** a PHP 8.4 release notes-ban (legalábbis az enquêteur írásának idején, 2026-05-14). Lehet hogy bug, lehet hogy szándékos AST-szigorítás. A workaround minden esetben egyszerű és szemantikailag ekvivalens.

## Megoldás

Cseréld `true|\WP_Error` → `bool|\WP_Error`:

```php
public static function validate_preview_param( string $value ): bool|\WP_Error {
    if ( $value === 'true' ) {
        return true;
    }
    return new \WP_Error( 'rest_invalid_param', '...', [ 'status' => 400 ] );
}
```

**Szemantikailag ekvivalens** mert:
- A függvény soha nem ad vissza `false`-t (csak `true` vagy `WP_Error`)
- A WordPress REST API a return value-t `is_wp_error()` ellenőrzéssel kezeli — `bool|WP_Error` ugyanúgy működik
- A `bool` típus magában foglalja a `true`-t

## Hol érinthet még

Bárhol a kódbázisban, ahol egy függvény `validate` / `permission_check` típusú szerepet tölt be, és **csak `true` vagy `\WP_Error` lehetséges return value**. Keresés:

```bash
grep -rn ": true|\\\\WP_Error\|: \\\\WP_Error|true" --include="*.php" .
```

WordPress core kontextusban:
- `permission_callback` callbackek (REST route)
- `sanitize_callback` / `validate_callback` paraméter validátorok
- Bármilyen custom middleware ami `true`-t vagy `WP_Error`-t ad vissza

## Prevention

1. **Új `validate_*` / `permission_*` callbackeket írj `bool|\WP_Error` return type-pal** (nem `true|\WP_Error`-rel) — semantikailag ekvivalens, jövőbiztos.
2. **CI run PHP 8.4-en is.** Ha a Spektra CI pipeline (ha lesz) több PHP verziót tesztel, az ilyen literál-vs-class parse hibák azonnal kibukkannak.
3. **PHP verzió fixálás production-ön.** Rackforest cPanel-en a `Select PHP Version` UI-ban explicit fixálni — automatikus „latest" upgrade ne történjen tudat nélkül.

## Agent / fejlesztő ellenőrző lista

Ha új WP REST plugin metódust írsz Spektra kontextusban:

- [ ] Return type `bool|\WP_Error` (NEM `true|\WP_Error`)
- [ ] Namespace-elt fájlban a `true`/`false` literál typeokat **kerülöd** ott ahol class autoloading aktiválódhat
- [ ] Tesztelve PHP 8.1, 8.2, 8.3, 8.4 mindegyiken (vagy legalább 8.1 és 8.4 mint határértékek)
- [ ] `wp-content/debug.log` üres a plugin aktiválás után

## Történet

| Esemény | Mikor | Hol |
|---------|-------|-----|
| Bevezetve | 2025-08 (?) | `sp-infra/plugin/spektra-api/includes/class-rest-controller.php` — P5.2 release (`0f7a129`) |
| Felfedezve | 2026-05-14 | Rackforest cPanel WP telepítés, P14.7 stage-ready (`wp.benettcar.hu`) — PHP 8.4 alapértelmezett verzió volt egy darabig |
| Hot-fix szerveren | 2026-05-14 | FTP overwrite `class-rest-controller.php`-ban |
| Commit a forrás-repo-ba | 2026-05-14 | `2202032` (sp-infra) — `fix(api): PHP 8.4 compatibility — bool\|WP_Error return type` |
| Dokumentálva | 2026-05-14 | ez a fájl |

## Kapcsolódó

- [`spektra-api-guide.md`](../../spektra-api-guide.md) — `class-rest-controller.php` szerepe
- [`plugin-architecture-guide.md`](../../plugin-architecture-guide.md) — plugin file struktúra
- `sp-clients/sp-benettcar/docs/deployment-runbook.md §3.4` — PHP verzió döntés Rackforest cPanel-en (8.1/8.2 ajánlott, 8.4 a fix után működik)
- `sp-clients/sp-benettcar/docs/deployment-runbook.md §17.1` — kliens-projekt szintű hivatkozás ehhez a hibához
