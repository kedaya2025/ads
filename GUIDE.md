# 广告接入手册

本手册面向 App / 桌面应用开发者，说明如何接入本广告服务。

---

## 1. 请求地址

所有请求均为标准 HTTP GET，返回 JSON。无需认证、无需请求头、无需 query 参数。

```
GET {baseUrl}/manifest.json
GET {baseUrl}/slots/{appId}/{slotId}.json
GET {baseUrl}/campaigns/{campaignId}.json
GET {baseUrl}/targeting/{appId}/{slotId}.json
GET {baseUrl}/defaults/{appId}/{slotId}.json
```

`{baseUrl}` 可使用官方地址或镜像地址，路径部分完全一致。详见 [README.md](README.md) 的 Mirror Endpoints & Fallback 章节。

---

## 2. 标准尺寸规范

本仓库预定义了一套标准广告尺寸。所有广告位必须从标准尺寸中选取，广告素材必须匹配所投放广告位的尺寸。

### 2.1 标准尺寸列表

| 尺寸 ID | 宽 × 高 (px) | 宽高比 | 适用场景 |
|---------|-------------|--------|----------|
| `banner_600x100` | 600 × 100 | 6:1 | 首页横幅、列表页横幅 |
| `banner_320x50` | 320 × 50 | 6.4:1 | 移动端顶部/底部条 |
| `banner_728x90` | 728 × 90 | 8.1:1 | Web 端横幅（Leaderboard） |
| `banner_300x250` | 300 × 250 | 1.2:1 | 内嵌矩形广告（Medium Rectangle） |
| `sidebar_130x300` | 130 × 300 | 1:2.3 | 侧边栏竖幅 |
| `splash_1080x400` | 1080 × 400 | 2.7:1 | 开屏广告 |

### 2.2 尺寸声明位置

- **`manifest.json`** 中的 `standardSizes` 字段定义所有合法尺寸（作为权威来源）
- **`slots/{appId}/{slotId}.json`** 中的 `dimensions` 必须引用上述标准尺寸之一
- **`campaigns/{campaignId}.json`** 中的 `media` 必须包含与广告位尺寸匹配的图片资源

### 2.3 manifest.json 中的 standardSizes 字段

```json
{
  "standardSizes": [
    {
      "id": "banner_600x100",
      "width": 600,
      "height": 100,
      "aspectRatio": "6:1",
      "description": "首页横幅"
    },
    {
      "id": "sidebar_130x300",
      "width": 130,
      "height": 300,
      "aspectRatio": "1:2.3",
      "description": "侧边栏竖幅"
    }
  ]
}
```

---

## 3. 多尺寸广告机制

同一个广告可能需要在不同尺寸的广告位中投放。本系统通过 **media.variants** 数组实现同一广告的多尺寸版本。

### 3.1 结构说明

`campaign.json` 的 `content.media` 中：

- **`image`** / **`width`** / **`height`**：默认尺寸的主素材（向后兼容）
- **`variants`**：可选数组，列出同一广告的其他尺寸版本

```json
{
  "content": {
    "media": {
      "image": "https://cdn.example.com/ads/b123-600x100.png",
      "width": 600,
      "height": 100,
      "format": "png",
      "variants": [
        {
          "sizeId": "banner_600x100",
          "image": "https://cdn.example.com/ads/b123-600x100.png",
          "width": 600,
          "height": 100
        },
        {
          "sizeId": "sidebar_130x300",
          "image": "https://cdn.example.com/ads/b123-130x300.png",
          "width": 130,
          "height": 300
        }
      ]
    }
  }
}
```

### 3.2 匹配规则

1. 客户端请求 `slots/{appId}/{slotId}.json`，获取广告位的 `dimensions`（包含 `width` 和 `height`）
2. 客户端请求 `campaigns/{campaignId}.json`，获取广告素材
3. 在 campaign 的 `media.variants` 中查找 `width` 和 `height` 与广告位尺寸匹配的版本
4. 若找到匹配的 variant，使用该 variant 的 `image`
5. 若无 `variants` 或无匹配项，回退到 `media` 顶层的 `image`（默认尺寸）

### 3.3 校验规则

- `media.width` 和 `media.height`（顶层默认尺寸）必须与 `standardSizes` 中的某个尺寸匹配
- `variants` 中每个 variant 的 `width` 和 `height` 必须与 `standardSizes` 中的某个尺寸匹配
- 若广告投放的广告位尺寸不在 `variants` 中，且不等于顶层默认尺寸，则该广告在该广告位中不展示

---

## 4. 客户端请求逻辑

### 4.1 完整流程

```
1. GET /manifest.json
   → 获取 tag（版本号）、apps 列表、standardSizes

2. 比较本地缓存的 tag 与远程 tag
   ├─ 相同 → 使用本地缓存，结束
   └─ 不同 → 继续

3. GET /slots/{appId}/{slotId}.json
   → 获取广告位尺寸、广告列表（含 campaignId、priority、weight、active、schedule）

4. 过滤广告列表
   ├─ active 为 false → 跳过
   └─ 当前时间不在 scheduleStart ~ scheduleEnd 范围内 → 跳过

5. GET /targeting/{appId}/{slotId}.json（可选）
   → 获取定向规则，按 priority 从高到低匹配，命中规则的 campaignIds 作为候选列表

6. GET /campaigns/{campaignId}.json
   → 对每个候选广告，拉取素材
   → 按 slot 的 dimensions 从 campaign 的 media.variants 中匹配尺寸
   ├─ 有匹配 variant → 使用 variant.image
   └─ 无匹配 → 跳过该广告（或回退到默认尺寸）

7. 按 priority 排序，同 priority 按 weight 加权轮播，选取最终展示广告

8. 展示广告，展示失败时走 fallback
   → GET /defaults/{appId}/{slotId}.json
```

### 4.2 请求地址不变时如何获取正确广告

客户端的请求地址（Base URL）始终不变。不同广告、不同尺寸的区分完全通过**路径参数**和**JSON 内容匹配**实现：

| 区分维度 | 实现方式 |
|----------|----------|
| 不同 App | 路径中的 `{appId}` |
| 不同广告位 | 路径中的 `{slotId}` |
| 不同广告素材 | 路径中的 `{campaignId}` |
| 不同尺寸 | slot 的 `dimensions` 与 campaign 的 `media.variants` 按 width×height 匹配 |
| 不同定向 | `targeting/{appId}/{slotId}.json` 中的 rules |

客户端无需在 URL 中传递尺寸参数。尺寸匹配在客户端本地完成：先拉取 slot 获取尺寸，再拉取 campaign 从 variants 中匹配。

---

## 5. 接入流程

### 5.1 新 App 接入

1. 在 `manifest.json` 的 `apps` 数组中新增 App 配置：

```json
{
  "appId": "com.example.app2",
  "name": "My App",
  "platforms": ["ios", "android"],
  "regions": ["CN", "TW"],
  "slots": ["home_banner", "sidebar_ad"],
  "activeCampaigns": 0
}
```

2. 在 `slots/{appId}/` 下为每个广告位创建 JSON 文件：

```
slots/com.example.app2/home_banner.json
slots/com.example.app2/sidebar_ad.json
```

3. 在 `targeting/{appId}/` 下创建定向规则（可选）

4. 在 `defaults/{appId}/` 下创建兜底配置（可选）

### 5.2 新广告位接入

1. 从 `manifest.json` 的 `standardSizes` 中选取尺寸

2. 创建 `slots/{appId}/{slotId}.json`：

```json
{
  "$schema": "slot-v1",
  "appId": "com.example.app2",
  "slotId": "home_banner",
  "label": "首页横幅",
  "dimensions": {
    "sizeId": "banner_600x100",
    "width": 600,
    "height": 100,
    "aspectRatio": "6:1",
    "format": "image"
  },
  "refreshIntervalSeconds": 30,
  "maxVisibleAds": 1,
  "campaigns": [],
  "updated": "2026-09-09T00:00:00Z"
}
```

3. 在 `manifest.json` 的对应 App 的 `slots` 数组中添加 slotId

### 5.3 新广告素材接入

1. 创建 `campaigns/{campaignId}.json`：

```json
{
  "$schema": "campaign-v1",
  "campaignId": "camp_double11_2026",
  "name": "双11大促",
  "type": "banner",
  "status": "active",
  "createdAt": "2026-09-09T00:00:00Z",
  "updatedAt": "2026-09-09T00:00:00Z",
  "createdBy": "marketing-team",

  "content": {
    "title": "双11大促 全场5折",
    "subtitle": "限时3天",
    "description": "全品类精选商品限时特惠",
    "media": {
      "image": "https://cdn.example.com/ads/double11-600x100.png",
      "width": 600,
      "height": 100,
      "format": "png",
      "variants": [
        {
          "sizeId": "banner_600x100",
          "image": "https://cdn.example.com/ads/double11-600x100.png",
          "width": 600,
          "height": 100
        },
        {
          "sizeId": "banner_320x50",
          "image": "https://cdn.example.com/ads/double11-320x50.png",
          "width": 320,
          "height": 50
        }
      ]
    },
    "cta": {
      "text": "立即抢购",
      "action": "web",
      "webUrl": "https://example.com/double11"
    }
  },

  "targeting": {
    "platforms": ["ios", "android"],
    "regions": ["CN", "TW", "HK"],
    "languages": ["zh-Hans", "zh-Hant"],
    "tags": ["ecommerce", "promotion"]
  },

  "tracking": {
    "impressionUrl": "https://track.example.com/impression?cid=camp_double11_2026",
    "clickUrl": "https://track.example.com/click?cid=camp_double11_2026",
    "conversionUrl": null
  },

  "metadata": {
    "budget": 100000,
    "spent": 0,
    "impressions": 0,
    "clicks": 0,
    "ctr": 0
  }
}
```

2. 在对应的 `slots/{appId}/{slotId}.json` 的 `campaigns` 数组中引用：

```json
{
  "campaignId": "camp_double11_2026",
  "priority": 100,
  "weight": 0,
  "active": true,
  "scheduleStart": "2026-11-08T00:00:00Z",
  "scheduleEnd": "2026-11-12T00:00:00Z"
}
```

3. 在 `manifest.json` 中更新对应 App 的 `activeCampaigns` 计数

### 5.4 校验清单

提交前检查：

- [ ] 所有 JSON 文件语法正确
- [ ] `manifest.json` 的 `apps[].slots` 与 `slots/` 目录下的实际文件一致
- [ ] `slots` 中引用的 `campaignId` 在 `campaigns/` 目录下存在对应文件
- [ ] `slots` 的 `dimensions` 引用了 `manifest.json` 中 `standardSizes` 定义的尺寸
- [ ] `campaigns` 的 `media` 顶层尺寸或 `variants` 中至少有一个匹配所投放广告位的尺寸
- [ ] `defaults` 中引用的 `campaignId` 在 `campaigns/` 目录下存在
- [ ] `targeting` 中引用的 `campaignIds` 在 `campaigns/` 目录下存在
- [ ] `manifest.json` 的 `version` 和 `tag` 已更新

---

## 6. 版本管理

| 变更类型 | 版本变化 | 示例 |
|----------|----------|------|
| 新增广告素材（仅内容变更） | PATCH | v1.0.0 → v1.0.1 |
| 新增广告位 | MINOR | v1.0.1 → v1.1.0 |
| 新增 App 注册 | MINOR | v1.0.1 → v1.1.0 |
| 新增标准尺寸 | MINOR | v1.1.0 → v1.2.0 |
| Schema 结构变更 | MAJOR | v1.2.0 → v2.0.0 |

CI 会根据 `manifest.json` 是否变更自动判断 minor 或 patch bump。

---

*文档版本: 1.0 | 最后更新: 2026-09-09*
