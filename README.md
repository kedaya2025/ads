# Ad Service

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub stars](https://img.shields.io/github/stars/kedaya2025/ads?style=social)](https://github.com/kedaya2025/ads)
[![GitHub forks](https://img.shields.io/github/forks/kedaya2025/ads?style=social)](https://github.com/kedaya2025/ads)

Lightweight ad content delivery service powered by GitHub repositories. No backend required.

## How It Works

```
App Start → Fetch manifest.json → Compare tag version
  ├─ Match → Use local cache
  └─ Mismatch → Fetch latest ad data → Cache → Display
```

Ad content is stored as JSON files and distributed via `raw.githubusercontent.com` CDN. Updating ads only requires pushing new JSON and tagging a release.

## API

### Base URL

```
https://raw.githubusercontent.com/kedaya2025/ads/main
```

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/manifest.json` | Global manifest (version, app registry, defaults) |
| GET | `/slots/{appId}/{slotId}.json` | Ad slot definition (active campaigns, priority, weight) |
| GET | `/campaigns/{campaignId}.json` | Campaign creative (content, targeting, tracking) |
| GET | `/targeting/{appId}/{slotId}.json` | Targeting rules (region, platform, language, etc.) |
| GET | `/defaults/{appId}/{slotId}.json` | Fallback ad configuration |

### Path Parameters

| Parameter | Example | Description |
|-----------|---------|-------------|
| `appId` | `com.example.app1` | App identifier registered in `manifest.json` |
| `slotId` | `home_banner` | Ad slot identifier |
| `campaignId` | `camp_b123_official` | Campaign identifier |

### Response

All endpoints return JSON with `Content-Type: application/json; charset=utf-8`. No query parameters, headers, or authentication required.

## Mirror Endpoints & Fallback

The official Base URL may be slow or unreachable on some networks. Use any mirror below as a drop-in replacement — the path suffix stays the same.

### Available Mirrors

| Mirror | Base URL | Cache Delay | Notes |
|--------|----------|-------------|-------|
| JsDelivr | `https://cdn.jsdelivr.net/gh/kedaya2025/ads@main` | ~10 min | Global CDN |
| FastGit | `https://raw.fastgit.org/kedaya2025/ads/main` | Real-time | GitHub raw mirror |
| GitHub Proxy | `https://ghproxy.com/https://raw.githubusercontent.com/kedaya2025/ads/main` | Real-time | Proxy, availability depends on provider |
| Official | `https://raw.githubusercontent.com/kedaya2025/ads/main` | Real-time | GitHub raw CDN |

> Third-party mirrors can go offline at any time. Configure multiple endpoints and fall back automatically.

### Multi-Endpoint Fallback

Configure a list of Base URLs. Try each in order until one returns HTTP 200. Recommended timeout: 5 seconds per attempt.

Recommended endpoint order:

```
https://cdn.jsdelivr.net/gh/kedaya2025/ads@main
https://raw.fastgit.org/kedaya2025/ads/main
https://raw.githubusercontent.com/kedaya2025/ads/main
```

## Documentation

- [Integration Guide](GUIDE.md) — Standard sizes, multi-size mechanism, client request flow, onboarding steps
- [Architecture](ARCHITECTURE.md) — JSON Schema, targeting system, fallback mechanisms

## License

[MIT](LICENSE)
