- ~/gits/xiaozhi-esp32-server is the server implementation. I followed docs/deployment-all.md to burn the firmware in the xiaozhi-esp32 folder into the robot

## Caddy Reverse Proxy Configuration Lessons Learned

**Critical: Route Order Matters for SPA Applications**
- The xiaozhi web application is a Single Page Application (SPA) that redirects to `/#/login`
- Caddy route handlers are processed in order (first match wins)
- Specific API routes must be defined BEFORE the catch-all route
- Catch-all route `/*` must be LAST to handle all SPA routes (/, /#/login, /other-paths)

**Working Caddy Configuration Pattern:**
```caddy
ramen.9chives.com:443 {
    # Specific API routes first
    handle /xiaozhi/ota/* {
        reverse_proxy http://192.168.1.204:8002
    }

    handle /xiaozhi/v1/* {
        reverse_proxy http://192.168.1.204:8000 {
            header_up Host {host}
            header_up X-Real-IP {remote}
            header_up X-Forwarded-For {remote}
            header_up X-Forwarded-Proto {scheme}
        }
    }

    handle /mcp/vision/explain/* {
        reverse_proxy http://192.168.1.204:8003
    }

    # Catch-all route LAST for SPA
    handle /* {
        reverse_proxy http://192.168.1.204:8002
    }
}
```

**WebSocket Connection Issues:**
- ESP32 must use URL parameters for device identification, not headers
- Working web client pattern: `ws://ramen.9chives.com/xiaozhi/v1/?device-id=<MAC>&client-id=<UUID>`
- ESP32 headers approach fails: `Device-Id: <MAC>`, `Client-Id: <UUID>`
- Let Caddy handle WebSocket upgrades automatically (don't force headers)

**Infrastructure:**
- Caddy proxy host: 192.168.1.11
- ESP32 server host: 192.168.1.204
- Ports: 8000 (WebSocket), 8002 (Web), 8003 (MCP)