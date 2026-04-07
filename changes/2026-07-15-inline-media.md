# Inline Media Display

**Date:** 2026-07-15

## Target

- [Product: Inline Media](../product/chat/inline-media.md)
- [Architecture: Media Proxy](../architecture/media-proxy.md)

## Delta

Inline media display does not exist yet. The following are not implemented:

- Media Proxy service (`agynio/media-proxy` repository)
- Media Proxy: Kubernetes deployment, Service, and Istio VirtualService (`media.agyn.dev`)
- Media Proxy: independent JWT validation (JWKS-based, same as Gateway)
- Media Proxy: CORS headers for cross-origin requests from `agyn.dev`
- Chat app: Service Worker registration and token lifecycle (`postMessage` for JWT updates)
- Chat app: Service Worker fetch interception for `media.agyn.dev` (auth header injection, `no-cors` → `cors` mode change)
- Chat app: markdown renderer media components (`<img>`, `<video>`, `<audio>`) using proxy URLs
- Chat app: download-original link on each rendered media element
- Image downsampling in the Media Proxy (`size` parameter)
- Range request support in the Media Proxy (video/audio streaming)
- SSRF protections in the Media Proxy (private network blocking, DNS rebinding, redirect limits)
- Content-type whitelisting in the Media Proxy (external URLs only — platform files serve any type)
- Platform file proxy (`agyn://file/<id>`) with authorization check

## Acceptance Signal

- Agent responses containing `![](https://...)` render images inline in the chat UI.
- Agent responses containing video/audio URLs render inline players with seeking support.
- Platform files (`agyn://file/<id>`) render inline via the media proxy, including non-media types (PDF, text, code).
- No direct browser requests to third-party servers for media content.
- Large images are downsampled for inline display; download link provides the original.
- Video and audio support range requests for progressive playback in all browsers including Safari.
- SSRF protections prevent the proxy from fetching private network addresses.
- Service Worker injects JWT on all `media.agyn.dev` requests; token refreshes propagate to the worker.
