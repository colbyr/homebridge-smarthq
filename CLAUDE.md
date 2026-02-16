# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Homebridge plugin that integrates GE SmartHQ appliances with Apple HomeKit. Uses the GE Brillion API with OAuth2 authentication, REST API for device discovery/control, and WebSocket for real-time ERD (Embedded Result Descriptor) state updates.

## Build & Development Commands

```bash
npm run build          # Clean + tsc + tsc-alias + copy UI files
npm run watch          # Build, link, and run with nodemon (live reload)
npm run lint           # ESLint check
npm run lint:fix       # ESLint autofix
npm run test           # Vitest single run
npm run test:watch     # Vitest watch mode
npm run test-coverage  # Vitest with coverage
npm run docs           # Generate TypeDoc documentation
```

- ESM project (`"type": "module"`) — use `.js` extensions in imports even for TypeScript files
- TypeScript path aliases: `@opal/*` → `src/devices/OpalIceMaker/`, `@root` → `src/`
- Node 20/22/24+, Homebridge ^1.9.0 or ^2.0.0

## Architecture

### Plugin Registration & Platform

- `src/index.ts` — entry point, registers `SmartHQPlatform` as a Homebridge DynamicPlatformPlugin
- `src/platform.ts` — core platform class (~1100 lines). Handles OAuth token lifecycle, device discovery, WebSocket setup, and device instantiation via `createSmartHQ[DeviceType]()` factory methods
- `src/settings.ts` — constants (API URLs, plugin/platform names), config interfaces, and all ERD type codes (hex constants like `0x0001`–`0x7A0F`)
- `src/getAccessToken.ts` — OAuth2 flow against GE Brillion accounts (handles MFA enrollment skip, terms acceptance)

### Device Model

All devices extend `deviceBase` (`src/devices/device.ts`), which provides:
- Homebridge API/HAP/logging access
- Per-device config (refreshRate, updateRate, pushRate, logging level)
- Accessory information setup (manufacturer, model, serial, firmware)

Each device class (e.g., `src/devices/oven.ts`) adds HomeKit Services and Characteristics, with `.onGet()`/`.onSet()` handlers that read/write ERD codes via the SmartHQ API.

### Device Discovery Flow

1. OAuth2 authentication → access token
2. WebSocket connection to `/appliance/*/erd/*` for real-time state changes
3. REST `GET /appliance` → list devices → fetch details and features per device
4. Switch on device type string → factory method → device class instantiation → register as PlatformAccessory

### Opal Ice Maker (Complex Device)

`src/devices/OpalIceMaker/` uses a **Manager pattern** — separate manager classes under `Managers/` handle distinct HomeKit services (power, nightlight, filter, scheduling, descale, status indicators). This is the most complex device implementation.

### SmartHqContext

The `PlatformAccessory<SmartHqContext>` context stores device metadata (jid, brand, model, serial, nickname, applianceId, firmware, features) and optional cached state per device type.

## Branch & PR Workflow

- **Stable branch**: `latest`
- **All PRs must target a beta branch** (e.g., `beta-0.5.0`), never directly to `latest`
- Beta branches: `beta-X.Y.Z`, created from `latest`
- Label PRs with `patch`, `minor`, or `major` for semantic versioning
- Beta merges to `latest` trigger stable releases

## Key Dependencies

- **axios** + **axios-cookiejar-support** + **tough-cookie** — HTTP with cookie persistence for OAuth
- **openid-client** — OIDC discovery and token exchange
- **cheerio** — HTML parsing for login form scraping
- **ws** — WebSocket client for real-time ERD updates
- **rxjs** — Interval-based polling for device state refresh
