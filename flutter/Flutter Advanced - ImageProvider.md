# How Flutter's ImageProvider Loads Images, and Why `cacheWidth` Matters

You call `Image.network(url)` in Flutter and the image shows up just fine. But have you ever asked yourself how many megabytes of RAM that image takes? 🤔

The answer might surprise you, and it can crash your app without you ever understanding why.

---
## The ImageProvider Pipeline (2 Steps)

![[ImageProvider pipeline.png]]

### 1) The Key (`obtainKey` → ImageCache lookup)

Before anything else, Flutter builds a key for the image from its **url** and **scale**, then looks it up in the `ImageCache`.
- If it finds the image, it returns it immediately, with no download and no decode.
- A different `scale` (1, 0.5, 2) means a different key, which means another copy in the cache.

### 2) Load → Resize → Decode

If the image isn't cached:
1. It is **loaded** (downloaded).
2. If you set `cacheWidth` or `cacheHeight`, it is **resized**.
3. Only then is it **decoded**.

> **Golden rule: never decode full size.** ❌

---
## ⚠️ Why This Matters

An image in RAM doesn't take its file size. It takes its size **after decoding**:

```
Decoded size = width × height × 4 bytes (R, G, B, A)
```

|Case|Calculation|RAM|
|---|---|---|
|🔴 10000 × 12000 image|120M pixels × 4 bytes|≈ **480 MB** 😱|
|🟢 Same image resized to 1280 × 720|1280 × 720 × 4|≈ **3.7 MB** ✅|

That's a difference of more than 100×. If a `ListView` holds several images like this, the app will crash.

---
## ✅ The One-Line Fix

```dart
Image.network(url, cacheWidth: 1280)
// Under the hood, this becomes:
ResizeImage(NetworkImage(url), width: 1280)
```

Flutter shrinks the image **before** the decode, not after.

---
## 💡 Tip

`cacheWidth` and `cacheHeight` are in **physical pixels**, so account for `devicePixelRatio`.
There's no reason to decode a 4000px image to show it in a card that is 300 logical pixels wide.

```dart
final dpr = MediaQuery.devicePixelRatioOf(context);

Image.network(
  url,
  cacheWidth: (300 * dpr).round(),
)
```

---

## 🧠 Image Memory Cache (`ImageCache`)

Flutter keeps decoded images in a single in-memory cache, `PaintingBinding.instance.imageCache`.

- **Stores decoded images**, so what sits in the cache is the `w × h × 4` size, not the file size. This is why `cacheWidth` also shrinks the cache footprint.
- **LRU eviction:** when the cache is full, the least recently used images are dropped first.
- **Default limits:** 1000 images and 100 MB. Whichever limit is hit first triggers eviction.
- **Memory only:** nothing is persisted, so every app restart starts with an empty cache. For a disk cache, use a package like `cached_network_image`.
- **Oversized images:** an image bigger than the byte limit can't be kept in the cache at all, so it gets decoded again each time it's needed.

### Tuning and controlling it

```dart
final cache = PaintingBinding.instance.imageCache;

// Adjust limits
cache.maximumSize = 200;                    // max number of images
cache.maximumSizeBytes = 50 << 20;          // 50 MB

// Manual control
cache.clear();                              // drop all cached images
cache.clearLiveImages();                    // drop live image references
cache.evict(key);                           // remove one image by its key
```

### Cache flow recap

```dart
Image.network(url)
   │
  ▼
obtainKey (url + scale)
   │
   ├── in ImageCache? ──► return immediately (no load, no decode)
   │
   └── not cached ──► Load → Resize (cacheWidth/cacheHeight) → Decode → store in ImageCache
```

---
## Key Takeaways

- `ImageCache` keys depend on the url and scale.
- The memory cache holds **decoded** images (default: 1000 images / 100 MB, LRU eviction) and is cleared on app restart.
- RAM usage = decoded size (`w × h × 4`), not file size.
- Use `cacheWidth` / `cacheHeight` so Flutter resizes **before** decoding.
- Size the value to the display size × `devicePixelRatio`.
