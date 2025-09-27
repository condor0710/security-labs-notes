---
title: Stored XSS via Image Upload — concise walkthrough
date: 2025-09-27 00:00:00 +0000
categories: [writeups, hackviser]
tags: [xss, web, file-upload]
excerpt: "Exploited a stored XSS by injecting a crafted filename that became part of an img src attribute and delivered an alert."
---

## Goal

Trigger a stored XSS delivered through an uploaded photo on `https://tidy-bloodstorm.europe1.hackviser.space/` without breaking the site.

## Summary

Tried embedding payloads inside images (SVG, EXIF, PNG tEXt, polyglot) using xss2png and similar methods; when those failed because the server served files as plain images, the vulnerability was exploited by injecting a crafted filename that became part of an `img src` attribute and delivered `alert(1)`.

## What I tried

- Generate payload‑containing images (SVG, PNG with IDAT/metadata) and upload them. Nothing executed because the app re-encoded or served files as `image/jpeg`.
- Append SVG to JPEG and upload. The file was still served as an image, so embedded JS did not run.
- Use `xss2png.py` to embed `<img onload>` or `<script>` into a PNG and upload — payload remained inert because the file was treated strictly as an image.

## Final, working approach (filename injection)

The app reflected the uploaded filename into the page `src` attribute unsafely. By closing that attribute and adding an event handler, we can make the browser run JS when the image fails to load.

### Working filename
```html
"a" onerror=alert(1) x="b.png
```

## References

- [vavkamil/xss2png on GitHub](https://github.com/vavkamil/xss2png)
- [HackerOne Report #964550](https://hackerone.com/reports/964550)
- [From PNG to XSS on Medium](https://medium.com/@rashith6384/from-png-to-xss-how-i-tricked-a-web-app-into-executing-my-payload-c66f5a990195)
