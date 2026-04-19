# MEMORY.md — Long-Term Memory

## About Rob
- Semi-retired software engineer, runs Mobilinkd (ham radio TNC business)
- Focus: APRS and packet radio, embedded development, electronic design
- Values: privacy — keep private stuff private; prefers I ask before doing anything borderline
- Preferred communication: Telegram (username: Mobilinkd)

## Active Project
**Mobilinkd TNC Web Configurator** — browser-based config tool for Mobilinkd BLE TNCs
- Repo: `/home/node/.openclaw/workspace/mobilinkd-webconfig/`
- Deployed: http://dev.mobilinkd.org (S3 bucket `dev.mobilinkd.org`)
- Stack: Vanilla JS, single HTML file, Web Bluetooth API (BLE/GATT only)
- Reference apps: mobilinkd/BLEConfig (Android), mobilinkd/iosTncConfig (iOS)
- Both BLE apps are co-authoritative for UI features

## Key Decisions
- Amber (#F7BC60) = logo/branding only; steel blue (#3A7CCC) = interactive elements
- Auto-adjust, TX Tail, Squelch, Connection Tracking = REMOVED (not in BLE hardware)
- Protocol layer implements ALL hardware commands; UI shows only what's in iOS/Android apps
- Planned: protocol log view (yellow = valid but unhandled, red = invalid)

## Infrastructure
- AWS IAM user: nic-ai / Account: 795478879997 / Region: us-east-1
- AWS CLI v2 installed at `/tmp/aws-cli-bin/aws`
- **Creds workaround**: OpenClaw's `sanitizeHostExecEnv` blocks AWS_*/GITHUB_TOKEN vars
  - AWS: `AWS_SHARED_CREDENTIALS_FILE` and `AWS_CONFIG_FILE` are in `allowedInheritedOverrideOnlyKeys` — should pass through if container runtime sets them; `.aws/credentials` created at `/home/node/.openclaw/workspace/.aws/credentials`
  - GitHub: `GH_CONFIG_DIR=/home/node/.openclaw/workspace/.gh gh ...` workaround for read-only; write ops need inline token
  - Both keys stored in workspace `.env` (not in openclaw.json env block)
