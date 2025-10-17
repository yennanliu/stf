# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

STF (Smartphone Test Farm) is a web application for debugging Android devices remotely from a browser. The system uses a microservices architecture where multiple independent processes communicate via ZeroMQ and Protocol Buffers.

## Architecture

STF follows a distributed microservices architecture with the following main components:

### Core Units (lib/units/)

- **provider**: Connects to ADB and spawns worker processes for each device. Sends/receives commands from processors
- **processor**: Main workhorse that bridges devices and app. Nearly all communication flows through it
- **app**: HTTP server serving static resources (images, scripts, stylesheets) and main UI
- **auth**: Authentication providers (mock, OAuth2, LDAP, SAML2, OpenID)
- **websocket**: Communication layer between client-side JavaScript and server-side ZeroMQ+Protobuf
- **api**: RESTful API endpoints for external integrations
- **storage**: Storage units for temporary files, APKs, and images (temp, s3, plugin-apk, plugin-image)
- **triproxy**: "Dumb" proxies that distribute requests across processor units (appside and devside)
- **reaper**: Monitors heartbeat events and marks lost devices as absent
- **groups-engine**: Core of device booking/partitioning system (scheduler, watchers for groups/devices/users)
- **device**: Device-specific functionality
- **log**: Logging units (e.g., log-rethinkdb)
- **notify**: Notification integrations (HipChat, Slack)

### Communication Flow

1. Client (browser) ↔ websocket unit ↔ triproxy-app ↔ processor units ↔ triproxy-dev ↔ provider units ↔ devices
2. All units use ZeroMQ sockets (PUB/SUB, DEALER/ROUTER, PUSH/PULL patterns) for inter-process communication
3. Protocol Buffers for serialization (lib/wire/)

### Frontend (res/)

- **res/app/**: Main AngularJS application with components organized by feature:
  - control-panes/: Device control UI
  - device-list/: Device inventory and filtering
  - group-list/: Group management
  - settings/: User settings
  - user/: User management
- **res/auth/**: Authentication UI (ldap, mock)
- **res/common/**: Shared components, language files, status pages
- **res/web_modules/**: Reusable web modules
- **Build system**: Webpack for bundling, Gulp for task automation

### Backend (lib/)

- **lib/cli/**: Command-line interface definitions for each unit
- **lib/db/**: Database models and migrations (RethinkDB)
- **lib/util/**: Shared utilities (logger, procutil, apiutil, devutil, etc.)
- **lib/wire/**: Protocol Buffer definitions

### Database

- Uses RethinkDB (>= 2.2) for storing device state, users, groups, and logs
- Database migrations handled via `stf migrate` command
- Built-in objects: administrator user and Common root group (configurable via env vars)

## Development Commands

### Setup
```bash
npm install          # Install dependencies (also runs bower install and gulp build)
npm link             # Link stf command globally (optional)
```

### Running Locally
```bash
rethinkdb            # Start RethinkDB first (creates rethinkdb_data folder)
stf local            # Start all STF processes with mock auth
stf local 0123456789ABCDEF  # Filter to specific device serials
```

Access at http://localhost:7100 (default port)

For external access:
```bash
stf local --public-ip <your_ip>
```

### Building
```bash
npm run prepare      # Full build (bower + webpack)
gulp build           # Build frontend only
gulp webpack:build   # Production webpack build
gulp clean           # Clean build artifacts
```

### Testing
```bash
npm test             # Run linting and version check
gulp lint            # JSON and ESLint
gulp eslint-cli      # ESLint with cache
gulp karma           # Unit tests (frontend)
gulp karma_ci        # Unit tests (CI mode, single run)

# E2E tests with Protractor
gulp webdriver-update        # Update webdriver (first time)
gulp protractor              # Run E2E tests (requires RethinkDB + stf local + device)
gulp protractor --multi      # Multiple browsers
gulp protractor --suite devices  # Specific suite
```

For remote STF testing:
```bash
export STF_URL='http://stf-url/#!/'
export STF_USERNAME='user'
export STF_PASSWORD='pass'
gulp protractor
```

### Translations
```bash
gulp translate       # Extract strings, push to Transifex, pull translations, compile
```

### Database
```bash
stf migrate          # Run database migrations
```

Environment variables for built-in objects:
- `STF_ROOT_GROUP_NAME`: Root group name (default: "Common")
- `STF_ADMIN_NAME`: Admin user name (default: "administrator")
- `STF_ADMIN_EMAIL`: Admin email (default: "administrator@fakedomain.com")

### Other Commands
```bash
stf -h               # Show all available commands
stf <command> -h     # Help for specific command
stf -V               # Show version
```

## Code Style

- Use EditorConfig for whitespace rules
- ESLint configured for JavaScript linting
- Do NOT touch version field in package.json
- Do NOT commit generated files unless already in repo
- Do NOT create top-level files/directories without updating .npmignore

## Key Implementation Details

### Adding a New Unit

1. Create unit implementation in `lib/units/<unit-name>/`
2. Create CLI command in `lib/cli/<unit-name>/` (exports command definition)
3. Register command in `lib/cli/index.js`
4. Unit should export a function that returns a Promise

### Device Communication

- Devices communicate through provider units using ADB
- Provider spawns workers for each device
- Workers use utility binaries: minicap (screen), minitouch (input), minirev (reverse port forwarding)
- STFService APK (jp.co.cyberagent.stf) installed on devices for enhanced functionality

### Authentication Flow

- Multiple auth providers available (mock, OAuth2, LDAP, SAML2, OpenID)
- Auth units generate session tokens
- Tokens validated by app/websocket/api units
- Users can generate personal access tokens for API usage

### Deployment

- Production deployment uses systemd + Docker
- Each unit runs in separate Docker container
- See doc/DEPLOYMENT.md for detailed systemd unit configurations
- Units communicate over network (ZeroMQ TCP sockets)
- Nginx typically used as reverse proxy to tie HTTP units together

## Dependencies

### Required External Tools
- Node.js >= 18.20.5 (up to 20.x supported)
- RethinkDB >= 2.2
- ADB (Android Debug Bridge)
- GraphicsMagick (for screenshot resizing)
- ZeroMQ libraries
- Protocol Buffers libraries
- CMake >= 3.9
- yasm
- pkg-config

### macOS Installation
```bash
brew install rethinkdb graphicsmagick zeromq protobuf yasm pkg-config cmake
```

## Important Notes

- Node.js versions beyond 20.x not supported due to dependency constraints
- macOS can be used for development but is NOT recommended for production (ADB reliability issues)
- Provider units cannot run multiple instances on same host (device contention)
- Some units like reaper, log-rethinkdb, and groups-engine should only have one instance
- Template units (ending with @) use instance identifiers (often port numbers)
- Device USB connections can be unreliable; reaper unit ensures database consistency

## User Instructions from Global Config

- Do NOT save compiled JS or .d.ts files in @lib paths (CDK handles during deployment)
