# Removal Plan: Original-Project Telemetry, Webhooks & Branding

> **Goal:** Remove only components that communicate with infrastructure owned by the original project or create vendor lock-in to the original maintainers.
>
> **Keep:** Generic, configurable third-party integrations (GA4, Google OAuth, OneSignal, Google Fonts, Leaflet/OSM, Google Play Billing). These already use env vars and are NOT hardcoded to the original owner's accounts.

---

## Phase 1: Remove Original-Project Webhooks & Telemetry

### 1.1 — Delete `src/lib/version-check.ts`
- Hardcoded to `https://api.kasirgratisan.my.id/webhook/kasir-gratisan/latest-version`
- Fires on every app load, sends `installation_id` + `version`
- No opt-out mechanism

### 1.2 — Remove version check from `src/App.tsx`
- Remove `import { checkVersion } from "@/lib/version-check"` (line 7)
- Remove `checkVersion()` call (line 58)

### 1.3 — Delete `src/lib/user-type.ts`
- Hardcoded to `https://external-api.freekasir.com/webhook/user-type`
- Sends `identifier` (deviceId) + `business-type`
- Designed for original project's data collection
- The survey feature serves no purpose without the webhook destination

### 1.4 — Delete `src/components/UserTypeModal.tsx`
- Depends on `user-type.ts`
- Rendered on Reports page to collect business type data

### 1.5 — Clean up `src/pages/Reports.tsx`
- Remove `import UserTypeModal from '@/components/UserTypeModal'` (line 16)
- Remove `import { shouldShowUserTypeSurvey } from '@/lib/user-type'` (line 17)
- Remove `surveyOpen` state (line 37)
- Remove `useEffect` block that triggers survey (lines 44-49)

### 1.6 — Remove submitReport from `src/components/ErrorBoundary.tsx`
- Hardcoded to `https://external-api.freekasir.com/webhook/issue-report` (line 107)
- Remove `submitReport` method (lines 70-131)
- Remove `showConsentOpen` and `reporting` state variables
- Remove the `<AlertDialog>` consent dialog (lines 171-194)
- Remove the "Report Issue" `<Button>` (lines 165-169)
- Keep: error display, stack trace display, copy-to-clipboard functionality

### 1.7 — Delete `src/pages/settings/IssueReport.tsx`
- Hardcoded to `https://external-api.freekasir.com/webhook/issue-report` (line 48)
- Entire page exists only to submit reports to original project's infrastructure

### 1.8 — Remove IssueReport route from `src/App.tsx`
- Remove `import IssueReport` (line 40)
- Remove `<Route path="/settings/report-issue" ...>` block (lines 260-267)

### 1.9 — Remove IssueReport nav item from `src/pages/Settings.tsx`
- Remove the navigation link/section pointing to `/settings/report-issue`

### Phase 1 Verification
```bash
npm run build
```

---

## Phase 2: Remove Original-Project External Links

### 2.1 — Clean `src/pages/Settings.tsx`
Remove these hardcoded external links and their surrounding UI elements:
- Google Play link: `https://play.google.com/store/apps/details?id=com.freekasir.app` (line 362)
- Feature requests: `https://kasirgratisan.fider.io` (line 869)
- Donation: `https://traktir.jipraks.com` (line 877)
- Telegram community: `https://t.me/kasirgratisan` (line 885)

### 2.2 — Clean `src/pages/settings/CloudBackupSettings.tsx`
Remove all links and references to:
- `https://dashboard.freekasir.com` (lines 430, 602, 636)
- Translation strings referencing `dashboard.freekasir.com` / `market.freekasir.com`
- "Preview Dashboard" buttons and related UI

### 2.3 — Clean `src/pages/settings/CloudOnlineStoreSettings.tsx`
Remove all links and references to:
- `https://market.freekasir.com/stores/{identifier}` (lines 267, 433, 995, 1000, 1044)
- Keep: OpenStreetMap tile server (`tile.openstreetmap.org`) — generic
- Keep: Leaflet icon CDN (`unpkg.com/leaflet`) — generic

### 2.4 — Remove T&C/Privacy links from `src/components/Onboarding.tsx`
- Remove `<a href="https://freekasir.com/terms">` link (line 694)
- Remove `<a href="https://freekasir.com/privacy">` link (line 704)
- Keep the checkbox and agreement text, just remove the hyperlinks

### 2.5 — Remove WhatsApp entry from `src/lib/whats-new.ts`
- Remove the `2026-06-join-whatsapp` feature entry from `FEATURES` array
- This entry links to `https://s.id/wafreekasir` (original project's WhatsApp group)

### Phase 2 Verification
```bash
npm run build
```

---

## Phase 3: Translation Cleanup

### 3.1–3.3 — Clean cloud/dashboard references in locale files
In `src/i18n/locales/en/settings.json`, `id/settings.json`, `ms/settings.json`:
- Remove/simplify strings referencing `dashboard.freekasir.com`, `market.freekasir.com`
- Remove analytics consent description string
- Remove report-issue translation keys
- Remove survey translation keys

### 3.4 — Clean whats-new translations in `src/i18n/locales/*/common.json`
- Remove `2026-06-rebrand-freekasir` entry
- Remove `2026-06-join-whatsapp` entry

### 3.5 — Remove survey translations in `src/i18n/locales/*/reports.json`
- Remove all `survey.*` translation keys

### Phase 3 Verification
```bash
npm run build
```

---

## Phase 4: Branding Replacement — Environment Variables

### 4.1 — Add branding env vars to `.env.example`
```env
# === Branding (configure for your project) ===
VITE_APP_NAME=MyPOS
VITE_APP_SHORT_NAME=MyPOS
VITE_APP_ID=com.example.mypos
VITE_APP_DOMAIN=https://example.com
VITE_APP_DESCRIPTION=Point of Sale application
```

### 4.2 — Update `capacitor.config.ts`
```ts
// Before:
appId: 'com.freekasir.app',
appName: 'FreeKasir',

// After:
appId: import.meta.env.VITE_APP_ID ?? 'com.example.app',
appName: import.meta.env.VITE_APP_NAME ?? 'MyPOS',
```

### 4.3 — Update `vite.config.ts` PWA manifest
```ts
// Before:
name: "FreeKasir - POS UMKM Gratis",
short_name: "FreeKasir",
description: "Aplikasi kasir gratis untuk UMKM Indonesia. Offline & tanpa biaya.",

// After:
name: import.meta.env.VITE_APP_NAME ?? 'MyPOS',
short_name: import.meta.env.VITE_APP_SHORT_NAME ?? 'MyPOS',
description: import.meta.env.VITE_APP_DESCRIPTION ?? 'Point of Sale',
```

### 4.4 — Update `index.html`
Replace all brand references with Vite `%VAR%` syntax where possible, or generic placeholders:
- `<title>` — use env var or placeholder
- `<meta name="author">`, `<meta name="application-name">`, `<meta name="apple-mobile-web-app-title">`
- JSON-LD structured data (name, author, description, image URLs)
- OG/Twitter meta tags (title, description, url, image, site_name)

### 4.5 — Rename localStorage key in `src/lib/cloud-auth.ts`
```
// Before: 'freekasir_cloud_token_v1'
// After:  `${VITE_APP_NAME}_cloud_token_v1` or generic prefix
```

### 4.6 — Rename backup filename in `src/lib/backup.ts`
```
// Before: 'freekasir-backup-${date}.json'
// After:  '${VITE_APP_NAME}-backup-${date}.json'
```

### 4.7 — Rename localStorage key in `src/components/PushPermissionModal.tsx`
```
// Before: 'freekasir_push_asked_v1'
// After:  '${VITE_APP_NAME}_push_asked_v1'
```

### 4.8 — Replace "FreeKasir" in all translation files
Bulk search-and-replace across `src/i18n/locales/{id,en,ms}/*.json`:
- `FreeKasir` → env var reference or generic placeholder
- `kagirgratisan.com` → env var reference
- `freekasir.com` → env var reference

### Phase 4 Verification
```bash
npm run build
```

---

## Phase 5: Static Asset Replacement

Replace all brand-specific assets in `public/` with new ones:

| Old File | Notes |
|----------|-------|
| `favicon.ico` | Replace with new favicon |
| `favicon-16x16.png` | Replace |
| `favicon-32x32.png` | Replace |
| `kasirgratisan-icon.png` | Replace + update references in `index.html`, `vite.config.ts` |
| `og-image.png` | Replace |
| `apple-touch-icon.png` | Replace |
| `android-chrome-192x192.png` | Replace |
| `android-chrome-512x512.png` | Replace |
| `header-icon.png` | Replace or remove |
| `site.webmanifest` | Update or remove (PWA manifest is generated by vite-plugin-pwa) |

### Phase 5 Verification
```bash
npm run build
```

---

## Phase 6: Final Verification

```bash
npm run lint     # No ESLint errors
npm run build    # Clean build, no broken imports
npm run test     # All existing tests pass
```

---

## Summary

### Files Deleted (4)
- `src/lib/version-check.ts`
- `src/lib/user-type.ts`
- `src/components/UserTypeModal.tsx`
- `src/pages/settings/IssueReport.tsx`

### Files Modified (21+)
- `src/App.tsx` — remove imports, calls, route
- `src/pages/Reports.tsx` — remove survey imports/state/effect
- `src/components/ErrorBoundary.tsx` — remove submit + button
- `src/pages/Settings.tsx` — remove external links, issue report nav
- `src/pages/settings/CloudBackupSettings.tsx` — remove dashboard links
- `src/pages/settings/CloudOnlineStoreSettings.tsx` — remove market links
- `src/components/Onboarding.tsx` — remove T&C/privacy links
- `src/lib/whats-new.ts` — remove WhatsApp entry
- `src/lib/cloud-auth.ts` — rename storage key
- `src/lib/backup.ts` — rename backup filename
- `src/components/PushPermissionModal.tsx` — rename storage key
- `.env.example` — add branding vars
- `capacitor.config.ts` — use env vars for appId, appName
- `vite.config.ts` — use env vars for PWA manifest
- `index.html` — replace brand meta/JSON-LD
- `src/i18n/locales/en/settings.json` — clean brand refs
- `src/i18n/locales/id/settings.json` — clean brand refs
- `src/i18n/locales/ms/settings.json` — clean brand refs
- `src/i18n/locales/en/common.json` — clean whats-new entries
- `src/i18n/locales/id/common.json` — clean whats-new entries
- `src/i18n/locales/ms/common.json` — clean whats-new entries
- `src/i18n/locales/en/reports.json` — clean survey keys
- `src/i18n/locales/id/reports.json` — clean survey keys
- `src/i18n/locales/ms/reports.json` — clean survey keys

### Assets Replaced (9)
- `public/favicon.ico`, `favicon-16x16.png`, `favicon-32x32.png`
- `public/kasirgratisan-icon.png` (rename + update refs)
- `public/og-image.png`
- `public/apple-touch-icon.png`
- `public/android-chrome-192x192.png`, `android-chrome-512x512.png`
- `public/header-icon.png`
- `public/site.webmanifest`

### Configurable 3rd-Party SDKs KEPT (no changes needed)
- Google Analytics 4 — `VITE_GA_MEASUREMENT_ID`
- Google OAuth — `VITE_GOOGLE_CLIENT_ID`
- OneSignal Push — `VITE_ONESIGNAL_APP_ID`
- Cloud API — `VITE_AUTH_API_URL`
- Google Fonts — generic CDN
- Leaflet/OpenStreetMap — generic tile server
- Google Play Billing — standard Android
