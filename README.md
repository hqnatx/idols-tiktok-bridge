# idols TikTok Live Bridge

Connecte un **live TikTok** au **client idolsClient** (Fortnite) pour déclencher des commandes de jeu en temps réel à partir d'événements du live (follows, likes, cadeaux, commentaires, partages).

Disponible en **app de bureau installable** (Windows, style idols glass/violet, titlebar personnalisée) construite avec **Tauri**, packagée en installeur NSIS: `Idols-TikTok-Bridge-Setup.exe`.

## Architecture

```
TikTok Live (WebCast WebSocket)
        │
        ▼
┌──────────────────────────────┐
│  Idols TikTok Bridge (Tauri)  │  App de bureau (Rust + WebView)
│  ┌─────────────────────────┐ │
│  │ UI (titlebar, sidebar,   │ │  Interface glass/violet
│  │ mappings, dashboard)     │ │
│  └───────────┬─────────────┘ │
│              │ fetch() localhost:5000
│  ┌───────────▼─────────────┐ │
│  │ Backend Python (sidecar) │ │  Frozen exe (PyInstaller)
│  │ - TikTokLive lib         │ │  ← reçoit les events TikTok
│  │ - Flask API              │ │  ← config des mappings
│  │ - TCP sender             │ │  ← envoie les commandes
│  └───────────┬─────────────┘ │
└──────────────┼───────────────┘
               │ TCP 127.0.0.1:48484
               ▼
┌──────────────────────┐
│  idolsClient (DLL)    │  (injecté dans Fortnite)
│  - TcpCommandListener │  ← reçoit les commandes
│  - ConsoleUI          │  ← exécute sur le game thread
│  - ExecuteConsoleCmd  │  ← UKismetSystemLibrary
└──────────────────────┘
```

## Installation (utilisateur final)

1. Lancer `Idols-TikTok-Bridge-Setup.exe`
2. Suivre l'installeur (NSIS)
3. Lancer "Idols TikTok Bridge" depuis le menu Démarrer

Le backend Python est embarqué (aucune installation Python requise). La configuration (mappings) est stockée dans `%APPDATA%\idols-tiktok-bridge\config.json`.

## Rebuild de l'installeur (développeur)

```bat
build-installer.bat
```

Ce script:
1. Fige `main.py` en exécutable autonome avec PyInstaller
2. Copie l'exécutable dans `app/src-tauri/binaries/` (sidecar Tauri)
3. Build l'app Tauri (`npx tauri build`) → génère `Idols-TikTok-Bridge-Setup.exe`

Structure de l'app Tauri: `app/src` (UI: `main.ts`, `api.ts`, `sidecar.ts`, `style.css`), `app/src-tauri` (Rust: `lib.rs`, `tauri.conf.json`, icônes, sidecar binaire).

## Sécurité

- **100% local** — Le listener TCP bind sur `127.0.0.1` uniquement, aucune connexion externe possible
- **Open source** — Basé sur `TikTokLive` (MIT, par isaackogan) et Flask (BSD)
- **Pas de credentials** — La connexion TikTok ne nécessite que le username (`@unique_id`), pas de mot de passe
- **Pas de backdoor** — Le code est minimal et auditable. Le listener ne fait que transmettre des lignes de texte à la console idols

## Système de licence

L'app utilise un système de licence par clé, identique à celui de la macro-app idols :

- **Format de clé** : `IDOLS-<DURATION>-<NONCE>-<HMAC_SIGNATURE>`
- **Validation HMAC** côtéé Rust (binaire natif compilé) avec secret `idols_macro_verify_secret_v1`
- **Hardware ID** : MachineGuid Windows hashé en SHA256 (ou fallback hostname+username)
- **Supabase** : Vérification qu'une clé n'a pas déjà été utilisée + enregistrement
- **Triple verrou** : frontend (écran d'activation) + backend (middleware Flask 403) + crypto native (Rust)
- **Durées** : `1H` (1 heure), `1D` (1 jour), `1W` (1 semaine), `LT` (à vie)

Au lancement, l'app vérifie la licence. Si aucune licence active n'est trouvée, l'utilisateur doit entrer une clé valide pour accéder à l'application.

## Mise à jour automatique

L'app vérifie automatiquement les mises à jour au démarrage via GitHub Releases. Si une nouvelle version est disponible, elle est téléchargée et installée automatiquement.

## Intégration idolsClient (une seule fois)

1. Copier `idolsClient_mod/TcpCommandListener.h` dans `idolsClient/Public/`
2. Copier `idolsClient_mod/TcpCommandListener.cpp` dans `idolsClient/Private/`
3. Ajouter les fichiers au `.vcxproj` (+ `ws2_32.lib` aux dépendances du linker)
4. Ajouter `#include "../Public/TcpCommandListener.h"` dans `Client.cpp`
5. Appeler `TcpCommandListener::Start();` au début de `ClientThread()` dans `Client.cpp`
6. Recompiler la DLL

## Utilisation

1. Lancer l'app "Idols TikTok Bridge" (installée via l'installeur, ou `run.bat` en mode dev)
2. Onglet **Settings**: coller le lien du live TikTok (ex: `https://www.tiktok.com/@username/live`) et sauvegarder
3. Onglet **Mappings**: configurer les déclencheurs:
   - **Follow** → déclenche une commande quand quelqu'un s'abonne
   - **Like** → déclenche à un seuil de likes (ex: `100`)
   - **Comment** → déclenche sur un regex (ex: `^!launch$`)
   - **Gift** → déclenche sur un nom de cadeau (ex: `rose`)
   - **Share** → déclenche quand quelqu'un partage le live
   - **Join** → déclenche quand quelqu'un rejoint
4. Onglet **Dashboard**: cliquer **Démarrer**
5. Lancer le jeu avec idolsClient injecté
6. Les événements TikTok déclenchent maintenant les commandes dans le jeu

## Exemples de mappings

| Event | Filtre | Commande | Description |
|-------|--------|----------|-------------|
| Follow | (vide) | `cheat launch 0 0 10000` | Lance le joueur en l'air |
| Gift | rose | `cheat spawnloot 1` | Spawn 1 loot |
| Like | 100 | `cheat god` | 100 likes = mode god |
| Comment | `^!launch$` | `cheat launch 0 0 5000` | !launch dans le chat |
| Share | (vide) | `cheat spawnloot 3` | Partage = 3 loots |

## Test sans TikTok

L'onglet **Dashboard** permet d'envoyer une commande de test directement au idolsClient via le bouton "Envoyer". Utile pour vérifier que la DLL est bien connectée.

## Sources open source utilisées

- [TikTokLive](https://github.com/isaackogan/TikTokLive) — Librairie Python pour recevoir les events TikTok Live (MIT)
- [TikTokLiveJava](https://github.com/jwdeveloper/TikTokLiveJava) — Référence pour l'API Java (MIT)
- [StreamLink](https://github.com/decryptable/stream-link) — Inspiration pour l'architecture event→action
- [TikTokLiveTool](https://github.com/petersvp/TikTokLiveTool) — Inspiration pour le mapping configurable

## Structure du projet

```
idols-tiktok-bridge/
├── main.py                     # API Flask (backend, sans UI servie)
├── license_manager.py          # Gestion licence (HMAC, Supabase, hardware ID)
├── tiktok_handler.py           # Logique de connexion TikTok Live
├── client_connector.py         # Client TCP vers idolsClient
├── config_manager.py           # Gestion config et matching events (config dans %APPDATA%)
├── config.json                 # Config par défaut (seed initial)
├── requirements.txt            # Dépendances Python
├── install.bat / run.bat       # Dev: lancer le backend seul (mode web legacy)
├── build-installer.bat         # Fige le backend + build l'app Tauri + installeur
├── Idols-TikTok-Bridge-Setup.exe  # Installeur généré (NSIS)
├── app/                         # App de bureau Tauri
│   ├── src/
│   │   ├── main.ts             # UI: titlebar, sidebar, dashboard, mappings, settings, licence
│   │   ├── api.ts              # Client HTTP vers le backend Python + types licence
│   │   ├── sidecar.ts          # Spawn du backend en sidecar au démarrage
│   │   └── style.css           # Thème glass violet (style idols)
│   └── src-tauri/
│       ├── src/lib.rs          # Setup fenêtre custom (transparent, sans décorations)
│       ├── src/license.rs      # Validation licence HMAC en Rust natif
│       ├── tauri.conf.json     # Config fenêtre + bundle NSIS + updater
│       ├── icons/               # Icônes de l'app
│       └── binaries/            # Backend Python figé (sidecar)
└── idolsClient_mod/
    ├── TcpCommandListener.h    # Header du listener TCP
    └── TcpCommandListener.cpp  # Implementation
```

## Ports utilisés

| Port | Usage | Bind |
|------|-------|------|
| 5000 | API backend Python (interne à l'app) | 127.0.0.1 |
| 48484 | TCP idolsClient listener | 127.0.0.1 |
