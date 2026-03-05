# PATCHES.md

This repository (`nemopopemo/dockerfiles`) contains **small self-hosting patches** on top of the upstream
EventSchedule application (`eventschedule/eventschedule`) to make it run reliably behind Coolify.

The upstream app is cloned during the Docker build. Therefore, patches are applied **during image build**
(e.g. by copying patched files from `patches/` into `/var/www/html/...`).

---

## Why patches are needed

When self-hosting (Coolify + reverse proxy + fresh database), some Blade layouts assume certain variables
exist (e.g. values normally injected by the hosted/SaaS environment). In a self-hosted setup those variables
may be missing, causing **HTTP 500** errors like:

- `Undefined variable $schedules (View: ... )`
- `Undefined variable $upgradeSubdomain (View: ... )`

Additionally, early bootstrapping can run before a full settings table exists during build time.

---

## Patch: AppServiceProvider.php

**File patched:**
- `app/Providers/AppServiceProvider.php`

**Patch source:**
- `patches/AppServiceProvider.php`

**What it changes:**
1. Ensures there is only **one** `boot()` method (prevents PHP issues).
2. Keeps the original behavior:
   - `Schema::defaultStringLength(191)`
   - `FORCE_HTTPS` support
   - loads settings from DB (if present) and shares them as `globalSettings`
3. Adds safe defaults for view variables that are required by layouts but may not be provided in self-hosted mode:
   - `schedules` → `collect()`
   - `venues` → `collect()`
   - `curators` → `collect()`
   - `upgradeSubdomain` → `null`

This prevents common 500 errors after login when layouts reference these variables.

---

## Notes for deployments (Coolify)

After deploying a new image, clear caches to ensure Blade templates and config are fresh:

```bash
cd /var/www/html
php artisan optimize:clear
php artisan view:clear
rm -f storage/framework/views/*.php 2>/dev/null || true
