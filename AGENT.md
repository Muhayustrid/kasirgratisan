# AGENT.md

Permanent instruction manual for AI agents working on this repository.

> **App name:** POS Kasir JURI
> The codebase still contains legacy references to "FreeKasir" and "MyPOS" (former names) in config files, i18n locales, and source strings. The current brand is **POS Kasir JURI**. Use this name for any new code, strings, or documentation. A full codebase rename is pending.

---

## 1. Project Overview

**POS Kasir JURI** is a free, offline-first, open-source Point of Sale (POS) Progressive Web App built for Indonesian UMKM (micro, small, and medium enterprises). All data is stored locally on the user's device via IndexedDB — no server, no registration, no cost.

The app ships as both an installable **PWA** and a native **Android APK** (Capacitor 8) from a single codebase. An optional paid cloud sync service exists for multi-store monitoring and automatic backup.

---

## 2. Purpose of the Application

Give Indonesian small businesses (warung, toko kelontong, retail kecil, F&B) a reliable, free POS that works **without internet**:

- Cashier / checkout with cart, discounts, open bills, change calculation
- Product CRUD with categories, SKU, units, barcode, photos
- Stock management (stock-in with automatic COGS via weighted average, stock-out, stock opname)
- Sales reports, expense tracking, debt tracking
- Multi-user mode with owner + staff roles and granular permissions
- Thermal receipt printing (Bluetooth ESC/POS) and digital receipt sharing
- Multi-language: Bahasa Indonesia, English, Bahasa Malaysia
- Backup/restore (local JSON + optional cloud)

The core promise: **full functionality without a network connection.**

---

## 3. Main Technologies

| Layer | Technology |
|-------|-----------|
| Framework | React 18 + TypeScript (non-strict) |
| Build | Vite (SWC plugin), target ES2022 |
| Styling | Tailwind CSS + shadcn/ui (HSL CSS variables in `src/index.css`) |
| Theming | next-themes (dark mode) |
| Database | IndexedDB via Dexie.js |
| Reactive data | dexie-react-hooks (`useLiveQuery`) |
| Server state | @tanstack/react-query |
| Routing | React Router DOM v6 |
| Forms & validation | React Hook Form + Zod |
| Charts | Recharts |
| Icons | Lucide React |
| i18n | i18next + react-i18next (id/en/ms, fallback `id`) |
| Date | date-fns (id, enUS, ms locales) |
| PWA | vite-plugin-pwa (Workbox, autoUpdate) |
| Native wrapper | Capacitor 8 (Android) |
| Printing | Web Bluetooth (PWA) / Classic Bluetooth (APK) — ESC/POS |
| Barcode | html5-qrcode |
| Tests | Vitest + @testing-library/react + fake-indexeddb + jsdom |
| Lint | ESLint 9 flat config + typescript-eslint |
| Package manager | Bun (preferred) or npm |

**Path alias:** `@/*` → `./src/*` (configured in `tsconfig.json`, `tsconfig.app.json`, `vite.config.ts`, `vitest.config.ts`).

**Dev server:** `http://localhost:8080`.

---

## 4. High-Level Architecture

### Entry & Provider Chain

```
src/main.tsx  →  src/App.tsx  →  providers  →  AppLayout  →  Routes
```

Provider nesting order in `App.tsx` (outer → inner):

`I18nextProvider` → `ErrorBoundary` → `QueryClientProvider` → `TooltipProvider` → `BrowserRouter` → `AuthProvider` → `GoogleOAuthProvider` → `CloudAuthProvider` → `<Routes>`

Every route element is individually wrapped in `<ErrorBoundary>` and rendered inside `<AppLayout />`.

### Directory Layout

```
src/
├── main.tsx              # Entry point + global error/unhandledrejection handlers
├── App.tsx               # Root component, provider chain, all route definitions
├── index.css             # Design tokens (HSL CSS variables), Tailwind layers
├── lib/                  # Business logic (db, auth, sync, cloud-api, printer, analytics, ...)
├── hooks/                # React contexts & hooks (use-auth, use-mobile, use-pwa-install, ...)
├── components/           # Shared components
│   ├── layout/           # AppLayout, BottomNav
│   ├── ui/               # shadcn/ui primitives (40+ components)
│   └── ...               # Feature components (Receipt, BarcodeScanner, Onboarding, ...)
├── pages/                # Route-level page components
│   └── settings/         # Settings sub-pages (13 files)
├── i18n/
│   ├── index.ts          # i18next initialization
│   └── locales/{id,en,ms}/  # 6 namespaces: common, onboarding, dashboard, reports, products, settings
└── test/                 # Vitest tests + setup.ts (fake-indexeddb, jest-dom, matchMedia mock)
```

### Database (Dexie / IndexedDB)

Defined in `src/lib/db.ts`:

- Database name: `kasirgratisan-db`
- Schema is at **version 14** — every version from 1 onward is preserved with `.upgrade()` migration steps. **Never remove or modify an existing version block.** To change the schema, add `this.version(N+1).stores({...}).upgrade(async (tx) => {...})`.
- **Soft deletes**: `isDeleted: 0|1` + `deletedAt: Date | null` (IndexedDB cannot index booleans, so numbers are used). Most tables follow this pattern.
- **Sync fields**: `updatedAt` and `syncedAt` are auto-managed by Dexie hooks in `setupSyncHooks()`. On create: auto-sets `updatedAt` and `syncedAt = null`. On update: auto-sets `updatedAt = now` and resets `syncedAt = null` (unless the update explicitly specifies these fields). Both trigger debounced `triggerBackgroundSync()`.
- **Hard-delete tombstones**: deleting from `paymentMethods`, `users`, `transactions`, `debts`, `stockOpnames` writes a row to `deletedRecords` for cloud sync.
- **Seed data**: `seedDefaultData()` runs from `AppLayout` on mount — idempotent, only seeds when tables are empty.
- **Tables** (17): `categories`, `products`, `suppliers`, `customers`, `stockIns`, `stockOuts`, `hppHistory`, `paymentMethods`, `transactions`, `transactionItems`, `storeSettings`, `users`, `units`, `expenseCategories`, `expenses`, `debts`, `debtPayments`, `stockOpnames`, `stockOpnameItems`, `deletedRecords`.

### Multi-User Auth & Permissions

- **Opt-in**: `storeSettings.multiUserEnabled` (default `false` → legacy single-user mode).
- Owner has all permissions; staff has granular `PermissionKey[]` (12 permission keys defined in `db.ts`).
- PIN: SHA-256 salted with `storeSettings.deviceId` (see `src/lib/auth.ts`).
- `useAuth()` (context in `src/hooks/use-auth.tsx`) provides: `can(key)`, `isOwner`, `currentUser`, `login`, `logout`, `refresh`.
- In legacy mode, `can()` always returns `true`.
- **Gate sensitive UI/actions with `can('permission_key')`**; use `<LockedPage />` as fallback.

### Cloud Sync (Optional, Paid)

- `src/lib/sync.ts` → debounced (2s) push of dirty records (where `updatedAt > syncedAt` or `syncedAt === null`) to `src/lib/cloud-api.ts` → REST API (`VITE_AUTH_API_URL`).
- Auth: Google ID token (JWT) via `@react-oauth/google` (web) or `@capgo/capacitor-social-login` (Android). Token injected via a getter registered by `use-cloud-auth`.
- Only active when `storeSettings.cloudStoreId` is set and a cloud token exists.
- `App.tsx` triggers `triggerBackgroundSync()` on startup and on `window.online` event.

### PWA vs Native Branching

- `Capacitor.isNativePlatform()` gates platform-specific behavior: StatusBar, Bluetooth printing method (Web Bluetooth vs Classic Bluetooth via `cordova-plugin-bluetooth-serial`), PWA install prompts, Google sign-in flow.
- `document.documentElement.classList.add('is-native')` is set on native for CSS branching.

### Build & Version

- `npm run build` auto-runs `scripts/bump-version.js` which updates `version.json` (date-based: `YYYY.MM.DD.N` + incremental `versionCode`).
- Capacitor scripts: `cap:sync`, `cap:android`, `cap:run` — all build web first, then sync to native.

### Environment Variables

Defined in `.env.example`:

| Var | Purpose |
|-----|---------|
| `VITE_APP_NAME` | App display name (PWA manifest, meta tags) |
| `VITE_APP_SHORT_NAME` | Short name for home screen |
| `VITE_APP_ID` | Capacitor/app identifier |
| `VITE_APP_DOMAIN` | Canonical URL |
| `VITE_APP_DESCRIPTION` | App description |
| `VITE_GA_MEASUREMENT_ID` | Google Analytics 4 (empty = disabled) |
| `VITE_AUTH_API_URL` | Cloud backup/sync API base URL |
| `VITE_GOOGLE_CLIENT_ID` | Google OAuth Web Client ID |
| `VITE_ONESIGNAL_APP_ID` | OneSignal push notifications (empty = disabled) |

---

## 5. Coding Philosophy Already in Use

- **shadcn/ui first**: use existing components in `src/components/ui/` before adding new ones. Config in `components.json` (aliases: `@/components`, `@/lib/utils`, `@/components/ui`).
- **Reactive data via `useLiveQuery`**: bind Dexie queries to UI; avoid manual refresh/polling.
- **i18n everywhere**: all user-facing text via `useTranslation('namespace')` + `t('key')`. Keys live in `src/i18n/locales/{id,en,ms}/*.json` across 6 namespaces.
- **Permission gating**: `can()` from `useAuth()` for sensitive actions; `<LockedPage />` as fallback.
- **Money as integer Rupiah**: no decimals; format with `toLocaleString('id-ID')` or locale-aware maps (`NUMBER_LOCALES`, `CURRENCY_SYMBOL` defined per page).
- **Locale-aware dates**: `date-fns` `format()` with locale from `LOCALES` map (`{ id, en: enUS, ms }`).
- **Soft deletes**: set `isDeleted = 1` + `deletedAt = new Date()`; query with `.where('isDeleted').equals(0)` or filter.
- **Error boundaries**: every page route wrapped in `<ErrorBoundary>`.
- **Default exports** for page components; named exports for lib/hooks utilities.
- **Toasts**: `sonner` — `toast.success(...)`, `toast.error(...)`, `toast.warning(...)`.
- **Non-strict TypeScript**: `strict: false`, `noImplicitAny: false`, `strictNullChecks: false` — this is intentional; match the existing style, do not tighten.
- **Minimal comments**: code is largely self-documenting; comments only for complex business logic (COGS formula, migration steps, sync behavior).
- **Responsive**: mobile-first (`max-w-lg`) with landscape/tablet adaptation (`md:max-w-6xl`); side-by-side layouts on wider screens.

---

## 6. Rules Every AI Agent Must Follow

1. **Offline-first always**: no feature may require a network connection to function. Cloud sync is an optional enhancement, not a dependency.
2. **i18n in all 3 languages**: any new user-facing string must be added to `id`, `en`, and `ms` locale files in the correct namespace.
3. **Use existing `ui/` components**: do not re-implement buttons, dialogs, inputs, selects, etc.
4. **`useLiveQuery` for Dexie data**: never poll or manually re-fetch IndexedDB data.
5. **Gate with `can()`**: sensitive actions must check `can('permission_key')` from `useAuth()`.
6. **Integer Rupiah**: monetary values are plain numbers in IDR — no float math, no decimals.
7. **Never hardcode locale**: use `i18n.language` with `NUMBER_LOCALES` / `CURRENCY_SYMBOL` / `LOCALES` maps.
8. **Never modify existing Dexie schema versions**: add a new `this.version(N+1)` block with `.upgrade()` if the schema changes.
9. **Run lint and tests before claiming done**: `npm run lint && npm run test && npm run build`.
10. **Use "POS Kasir JURI"** for any new app name references in code, strings, or docs.
11. **Match existing file conventions**: check neighboring files for import style, export style, and patterns before writing new code.

---

## 7. Files/Folders to Modify Carefully

| File/Folder | Why |
|------------|-----|
| `src/lib/db.ts` | Dexie schema versions (1–14), TypeScript interfaces, sync hooks, seed data. Never edit an existing version block. Always add a new version with an `.upgrade()` step. |
| `src/lib/sync.ts` | Cloud sync — debounce, locking, dirty-record detection. Breaking this corrupts sync state. |
| `src/lib/cloud-api.ts` | Cloud API contract (endpoints, types). Changes must match the backend. |
| `src/lib/auth.ts` | PIN hashing (SHA-256 + deviceId salt), permission definitions, session management. |
| `src/lib/printer.ts` | ESC/POS command generation for thermal printers — platform-branched (Web Bluetooth vs Classic). |
| `src/i18n/locales/**` | All 3 locales (id/en/ms) must stay in sync across 6 namespaces. Missing keys cause missing translations. |
| `src/App.tsx` | Provider nesting order matters (auth → Google → cloud). Route definitions live here. |
| `src/components/layout/AppLayout.tsx` | Onboarding/login gates, seed trigger, responsive layout shell. |
| `capacitor.config.ts` | Native app config (appId, appName, plugins). Changing `appId` breaks Android installs and data migration. |
| `vite.config.ts` | PWA manifest, Workbox runtime caching, build target, path alias, dev server. |
| `tsconfig*.json` | Intentionally non-strict — do not tighten without explicit request. |
| `eslint.config.js` | Flat config — `@typescript-eslint/no-unused-vars` is off; do not change without request. |
| `scripts/bump-version.js` | Version numbering logic; runs automatically on `npm run build`. |
| `version.json` | Auto-managed by `bump-version.js` — do not edit manually. |
| `.env.example` | Template for env vars; document any new `VITE_*` var here. |
| `android/` | Native Android project — modify only with Capacitor/Gradle knowledge. |
| `removal_telementry_plan.md` | Historical plan document — reference only, do not execute without explicit request. |

---

## 8. General Workflow Before Implementing a Feature

1. **Read** `README.md` and relevant files in `src/lib/`, `src/pages/`, `src/i18n/locales/`.
2. **Confirm offline-first**: the feature must work with no network. Cloud sync is optional.
3. **Plan DB changes**: if the Dexie schema needs to change, add a new `this.version(N+1).stores({...}).upgrade(...)` block. Never modify existing versions.
4. **Plan i18n keys**: list all new strings and add them to `id`, `en`, and `ms` locale files in the appropriate namespace.
5. **Build UI**: use existing `ui/` components; use `useLiveQuery` for data binding.
6. **Gate permissions**: wrap sensitive actions with `can()` from `useAuth()`; render `<LockedPage />` when denied.
7. **Format money/dates**: integer Rupiah with locale-aware formatting; `date-fns` with locale map.
8. **Write tests**: add tests under `src/test/` for testable logic (use `fake-indexeddb` — already set up in `src/test/setup.ts`).
9. **Verify**: run `npm run lint && npm run test && npm run build` before claiming the task is done.

---

## 9. Rules for Planning Before Coding

- Reproduce and understand existing patterns first (read neighboring code, check `package.json` for available libraries).
- Never assume a library is available — verify in `package.json` or existing `import` statements.
- Propose Dexie schema migrations explicitly: which tables, which indexes change, what `.upgrade()` does, and whether new tables need sync hooks in `setupSyncHooks()`.
- List all i18n keys needed across all 3 languages up front, in the correct namespace.
- Identify any change that affects the cloud sync contract: `updatedAt`/`syncedAt` auto-management, `deletedRecords` tombstones, `setupSyncHooks()` table list.
- Flag anything that would break offline-first behavior.
- Check `tsconfig` — non-strict mode is intentional; do not propose strict-null-check or `noImplicitAny` fixes unless asked.
- Identify platform branches: does the feature need different behavior on PWA vs native Android? Use `Capacitor.isNativePlatform()`.

---

## 10. Things AI Must Never Do (Unless Explicitly Requested)

- Modify source code outside the requested scope.
- Refactor or reformat code unrelated to the task.
- Change TypeScript strictness (`tsconfig*.json`) or ESLint rules (`eslint.config.js`).
- Delete or modify existing Dexie schema version blocks in `src/lib/db.ts`.
- Bypass `can()` permission checks or hardcode user-facing strings (skip i18n).
- Add server-required behavior that breaks offline-first.
- Commit changes — only commit when explicitly asked.
- Create markdown/documentation files other than what is explicitly requested.
- Manually edit `version.json` — it is auto-bumped by `scripts/bump-version.js` on build.
- Touch `android/` native files without Capacitor/Gradle knowledge.
- Add comments to code (match the existing minimal-comment convention).
- Change `capacitor.config.ts` `appId` — it breaks Android installs and data migration.
- Remove legacy "FreeKasir"/"MyPOS" references without an explicit rename request — some are tied to external service URLs (e.g., `FreeKasir.com`) and cloud branding (`FreeKasir Cloud`).
