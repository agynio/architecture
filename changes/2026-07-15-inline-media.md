# Inline Media Display

**Date:** 2026-07-15

## Target

- [Product: Inline Media](../product/chat/inline-media.md)
- [Architecture: Media Proxy](../architecture/media-proxy.md)

## Delta

Inline media display does not exist yet. The following are not implemented:

- Media Proxy service (`agynio/media-proxy` repository)
- Media Proxy: Kubernetes deployment, Service, and Istio VirtualService (`agyn.dev/media/*`)
- Media Proxy: independent JWT validation (JWKS-based, same as Gateway)
- Chat app: Service Worker for authenticated media requests (JWT injection on `/media/proxy/` path)
- Chat app: markdown renderer media components (`<img>`, `<video>`, `<audio>`) using proxy URLs
- Chat app: download-original link on each rendered media element
- Image downsampling in the Media Proxy
- Range request support in the Media Proxy (video/audio streaming)
- SSRF protections in the Media Proxy (private network blocking, DNS rebinding, redirect limits)
- Content-type whitelisting in the Media Proxy
- Platform file proxy (`agyn://file/<id>`) with authorization check

## Acceptance Signal

- Agent responses containing `![](https://...)` render images inline in the chat UI.
- Agent responses containing video/audio URLs render inline players with seeking support.
- Platform files (`agyn://file/<id>`) render inline via the media proxy.
- No direct browser requests to third-party servers for media content.
- Large images are downsampled for inline display; download link provides the original.
- Video and audio support range requests for progressive playback in all browsers including Safari.
- SSRF protections prevent the proxy from fetching private network addresses.
