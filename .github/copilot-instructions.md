# DidierLauncher v2 - AI Assistant Instructions

## Project Overview
A Minecraft launcher built with **Electron** (v39.2.4) and vanilla JS modules. Supports modded gameplay with multi-account authentication (Microsoft, Mojang, AZauth) and real-time game server status monitoring. Uses `minecraft-java-core` for game operations and `node-bdd` for local database management.

## Architecture Patterns

### Window & IPC Communication
- **Two-window design**: `updateWindow.js` (updates) → `mainWindow.js` (launcher)
- **IPC Pattern**: Renderer process sends/invokes via `ipcRenderer` → main process handles via `ipcMain`
- Key handlers in [src/app.js](src/app.js): window lifecycle, progress bars, dev tools toggle, authentication
- Critical: Use `ipcRenderer.send()` for fire-and-forget, `ipcRenderer.invoke()` for async responses

### Module System: ES6 Imports + CommonJS
- **Frontend**: ES6 `import/export` (panels, utils) 
- **Backend/IPC**: CommonJS `require()` (app.js, windows, config)
- File structure: [src/assets/js/panels/](src/assets/js/panels/) (UI controllers) + [src/assets/js/utils/](src/assets/js/utils/) (shared logic)
- Never mix import styles in same file

### Panel Architecture
Three main UI panels, loaded dynamically:
1. **Login** [src/assets/js/panels/login.js](src/assets/js/panels/login.js): Handles auth (Microsoft, Mojang, AZauth) → saves to database
2. **Home** [src/assets/js/panels/home.js](src/assets/js/panels/home.js): Account selection, server status, instance launching
3. **Settings** [src/assets/js/panels/settings.js](src/assets/js/panels/settings.js): Configuration & preferences

Panel pattern: Static `init(config)` called with launcher config, registers event listeners, manipulates DOM via selectors.

### Data Flow: Config → Database → UI
- **Config source**: Remote fetch from [src/assets/js/utils/config.js](src/assets/js/utils/config.js)
  - Endpoint: `${pkg.url}/launcher/config-launcher/config.json` (defined in package.json)
  - Supports RSS for news feed via `config.rss` field
- **Local storage**: [src/assets/js/utils/database.js](src/assets/js/utils/database.js)
  - Uses `node-bdd` wrapper → SQLite in dev, `.db` in prod
  - Path: `userData/databases` (configurable per environment)
  - CRUD methods: `createData()`, `readData(tableName, key=1)`, `readAllData()`, `updateData()`, `deleteData()`
- **State sharing**: [src/assets/js/utils.js](src/assets/js/utils.js) exports singleton utilities (logger, config, database, popup, etc.)

## Build & Development

### Scripts
- `npm start`: Dev mode with Electron (uses `userData: ./data/Launcher`)
- `npm run dev`: Watch mode with nodemon (rebuilds on `.js/.html/.css` changes)
- `npm run build`: Obfuscate + bundle for production (see [build.js](build.js))
- `npm run icon`: Custom icon generation

### Dev Environment Setup
- Dev mode triggered by `NODE_ENV=dev` → uses local `./data/Launcher` for userData instead of system paths
- DevTools auto-open available: Press `Ctrl+Shift+I` or `F12`
- Nodemon ignores `**/test` directories

### Build Process ([build.js](build.js))
- Obfuscates JS via `javascript-obfuscator` (medium preset) when `--obf=true`
- Outputs to `./app/` directory
- Replaces `src/` paths to `app/` in obfuscated code
- Supports platform-specific builds via electron-builder

## Key Integration Points

### Authentication Flow
Three auth methods configured via `config.online` field:
- **Boolean `true`**: Microsoft auth using `client_id` → `ipcMain.handle('Microsoft-window', client_id)`
- **Boolean `false`**: Crack mode (offline)
- **String URL**: AZauth endpoint

Post-auth: Account saved via `database.createData('accounts', accountObject)` with skin processing.

### Game Server Status
- [src/assets/js/utils.js](src/assets/js/utils.js) `setStatus()` uses `minecraft-java-core`'s `Status` class
- Queries IP/port from config, displays online count + response time
- Graceful fallback: Red indicator for unreachable servers

### Instance Management
- Retrieved via `config.getInstanceList()` → fetches from `${url}/files`
- Each instance is a modpack variant
- Launching delegated to `minecraft-java-core` (specific handler in home panel)

## Testing & Debugging

- **Dev Tools**: Toggle via `ipcRenderer.send('main-window-dev-tools')` 
- **Logging**: Import `logger` from utils, automatically prefixes `[DidierLauncher]` with Discord blue color (#7289da)
- **Database inspection**: Check SQLite files in `data/Launcher/databases/` during dev
- **Hot reload**: Press `Ctrl+W` to exit launcher (Ctrl+Shift+R reloads window via ipcMain)

## Critical Patterns & Conventions

1. **CSS variables**: All colors via `--color` prefix (theme support in [src/assets/css/theme.css](src/assets/css/theme.css))
2. **String selectors**: Panels use class names (e.g., `.login`, `.home`) for visibility toggling
3. **Error handling**: Always use `popup` utility for user-facing errors (see [src/assets/js/utils/popup.js](src/assets/js/utils/popup.js))
4. **Path handling**: Use `ipcRenderer.invoke('path-user-data')` to get correct userData path (critical for cross-platform)
5. **Async database**: All DB operations are async, always await calls

## Common Additions
- **New panel**: Create in [src/panels/](src/panels/), import in [launcher.js](src/assets/js/launcher.js), follow Login/Home/Settings pattern
- **New utility**: Export from [src/assets/js/utils.js](src/assets/js/utils.js) and import in panels
- **New IPC handler**: Add to [src/app.js](src/app.js) main process, invoke from renderer via `ipcRenderer.handle()` or `send()`
