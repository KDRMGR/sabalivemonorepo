# SABALIVE Monorepo

| Folder | Stack | What it is |
| --- | --- | --- |
| [`sabalive/`](sabalive/) | Flutter (Dart) | Consumer mobile app |
| [`sabaliveadmin/`](sabaliveadmin/) | React + Vite (Node) | Admin panel (plus `landing/`, the public site) |

The two projects are fully independent — each has its own dependencies, lockfile
and toolchain, and both talk to the same Supabase project. There is deliberately
no workspace tool (Melos, Nx, Turborepo, npm workspaces): nothing is shared
between the apps, so it would only add overhead. Always `cd` into the project
before running its commands.

## Getting the code

Each app keeps its own git repo and is linked here as a git submodule:

```bash
git clone --recurse-submodules https://github.com/KDR9MGR/sabalivemonorepo.git
# or, in an existing clone:
git submodule update --init --recursive
```

Commit and push inside `sabalive/` or `sabaliveadmin/` first, then commit the
updated submodule pointer here (`git add sabalive sabaliveadmin`).

## Requirements

- Flutter SDK (Dart `^3.11.3`) — <https://docs.flutter.dev/get-started/install>
- Node.js 20+ and npm

## Flutter app — `sabalive/`

```bash
cd sabalive
flutter pub get
flutter run                     # debug on a connected device/simulator
flutter analyze
flutter test
flutter build apk --release     # Android APK
flutter build appbundle         # Play Store bundle
flutter build ios --release     # iOS (macOS only)
```

## Admin panel — `sabaliveadmin/`

```bash
cd sabaliveadmin
cp .env.example .env.local      # then fill in your Supabase URL + anon key
npm install
npm run dev                     # http://localhost:5173
npm run build                   # production build -> dist/
npm run preview                 # serve the production build locally
```

The public landing site in `sabaliveadmin/landing/` is a separate Vite project;
see its [README](sabaliveadmin/landing/README.md). Deployment details for the
combined site are in [`sabaliveadmin/README.md`](sabaliveadmin/README.md).

## Repository notes

- `sabalive/` and `sabaliveadmin/` are submodules; this repo only stores which
  commit of each is current.
- Root `.gitignore` covers Flutter/Dart, Node/Vite, IDE files, build output and
  env files. `.env.example` files are kept; real `.env*` / `*.local` files and
  signing keys (`*.jks`, `key.properties`) are never committed.
- Never commit secrets. The admin panel only ever uses the Supabase anon key.
