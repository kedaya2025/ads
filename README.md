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

## Quick Start

### Base URL

```
https://raw.githubusercontent.com/kedaya2025/ads/main
```

### Fetch Ads

```
GET {baseUrl}/manifest.json                          # Version check
GET {baseUrl}/slots/{appId}/{slotId}.json            # Ad slot
GET {baseUrl}/campaigns/{campaignId}.json            # Ad creative
```

## Usage Examples

### iOS / Android

```swift
// 1. Check version
let manifest = try await fetch(url: baseUrl + "/manifest.json")
let remoteTag = manifest["tag"] as! String

// 2. Fetch if version differs
if remoteTag != localTag {
    let slot = try await fetch(url: baseUrl + "/slots/com.app/home.json")
}

// 3. Match targeting rules → Display
```

### Web

```javascript
const baseUrl = 'https://raw.githubusercontent.com/kedaya2025/ads/main';

const manifest = await fetch(`${baseUrl}/manifest.json?t=${Date.now()}`).then(r => r.json());
const slot = await fetch(`${baseUrl}/slots/com.example.app1/home_banner.json`).then(r => r.json());
```

### Python

```python
import requests

base_url = "https://raw.githubusercontent.com/kedaya2025/ads/main"
manifest = requests.get(f"{base_url}/manifest.json").json()
slot = requests.get(f"{base_url}/slots/com.example.app1/home_banner.json").json()
```

## CDN Nodes

| Region | Endpoint | Notes |
|--------|----------|-------|
| Global | `raw.githubusercontent.com` | Official CDN |
| China | **Self-hosted required** | GitHub access is unstable in CN |

### China Acceleration

Use Cloudflare Workers to proxy `raw.githubusercontent.com`:

```javascript
addEventListener('fetch', event => {
  const url = new URL(event.request.url);
  const target = `https://raw.githubusercontent.com${url.pathname}`;
  event.respondWith(fetch(target));
});
```

### Multi-Endpoint Fallback

Configure multiple endpoints for automatic fallback:

```json
{
  "endpoints": [
    "https://your-cf-worker.example.com/ads",
    "https://raw.githubusercontent.com/kedaya2025/ads/main"
  ]
}
```

## Documentation

- [Architecture](ARCHITECTURE.md) — JSON Schema, targeting system, fallback mechanisms

## License

[MIT](LICENSE)
