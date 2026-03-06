# catan-log

A mobile web app for logging Catan board game sessions with your regular group.

Built as a Progressive Web App (PWA) — install it to your home screen and use it like a native app, no app store required.

## Features

- 📷 Photograph your board at the start of each game
- 🤖 AI-powered board analysis — identifies all 19 tiles and number tokens from your photo
- 👥 Persistent player profiles for your regular group
- 🎲 Score tracking with settlements, cities, longest road and largest army
- 📊 Stats across all sessions — win rates, average victory points, special card history
- 📜 Full game history with notes and expansion tracking

## How to use

Visit the live app at `https://jvelas11.github.io/catan-log/` on your mobile browser and tap **Add to Home Screen** to install it as a PWA.

To enable AI board analysis, you'll need an [Anthropic API key](https://console.anthropic.com). Add it once via the Settings screen — it's stored only on your device.

## Tech

Single-file PWA — HTML, CSS and JavaScript with no build step or dependencies. Data stored in browser localStorage. AI analysis via the Anthropic Claude API.

## Development

See `CONTEXT.md` for full technical documentation, data model, and project history.
