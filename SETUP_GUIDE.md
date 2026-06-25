# Camofox Docker Setup Guide

A comprehensive guide to deploying Camofox reliably in Docker with all features working.

## Critical Configuration Issues Not in Docs

### Issue 1: Display Initialization Bug
**Problem**: `Error: cannot open display: [object Promise]`  
**Root Cause**: Missing `await` on async VirtualDisplay.get() method  
**Fix**: Line 955 in `server.js`
```javascript
// WRONG:
vdDisplay = localVirtualDisplay.get();

// CORRECT:
vdDisplay = await localVirtualDisplay.get();
```

### Issue 2: CDP Viewport Schema Mismatch (2 locations)
**Problem**: `Found property "<root>.viewport.isMobile" not described in this scheme`  
**Root Cause**: Playwright sends `isMobile: false` property that Camoufox CDP doesn't recognize  
**Locations**:
1. Session context creation (line ~1180)
2. Proxy validation probe (line ~710)

**Fix**: Set `viewport: null` instead of `{ width, height }`
```javascript
// WRONG:
context = await browser.newContext({
  viewport: { width: 1280, height: 720 },
});

// CORRECT:
context = await browser.newContext({
  viewport: null,
});
```

### Issue 3: VNC Plugin Disabled by Default
**Problem**: noVNC/x11vnc services don't start  
**Solution**: Enable in `camofox.config.json`
```json
{
  "plugins": {
    "vnc": { "enabled": true, "resolution": "1920x1080" }
  }
}
```

### Issue 4: VNC WebSocket Binding
**Problem**: `Connection refused` on port 6080  
**Root Cause**: websockify binds to `127.0.0.1` (localhost only) by default  
**Fix**: Set environment variable
```bash
docker run ... --env VNC_BIND=0.0.0.0 ...
```

### Issue 5: x11vnc Auto-Detection Fails
**Problem**: x11vnc doesn't start automatically from vnc-watcher.sh  
**Root Cause**: Script can't detect Xvfb display when using `-displayfd` (file descriptor)  
**Workaround**: Manually start x11vnc in container init or use explicit DISPLAY

## Docker Compose Configuration

```yaml
version: '3.8'
services:
  camofox-browser:
    build: .
    container_name: camofox-browser
    ports:
      - "9377:9377"    # API server
      - "6080:6080"    # noVNC web UI
      - "5901:5900"    # Direct VNC
    environment:
      - NODE_ENV=production
      - CAMOFOX_PORT=9377
      - VNC_BIND=0.0.0.0           # Allow external connections
      - VNC_PASSWORD=${VNC_PASSWORD}  # Optional
      - VNC_RESOLUTION=1920x1080x24
    volumes:
      - ./profiles:/root/.camofox/profiles  # Persist browser state
      - ./cache:/root/.cache/camoufox       # Cache browser binary
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9377/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
```

## Required Code Changes

### 1. Enable VNC Plugin
File: `camofox.config.json`
```json
{
  "plugins": {
    "youtube": { "enabled": true },
    "persistence": { "enabled": true },
    "vnc": { "enabled": true, "resolution": "1920x1080" }
  }
}
```

### 2. Fix Viewport Issues
File: `server.js`

**Line 955** (Display initialization):
```javascript
vdDisplay = await localVirtualDisplay.get();  // Add await
```

**Line ~1180** (Session creation):
```javascript
const contextOptions = {
  viewport: null,  // Changed from { width: 1280, height: 720 }
  permissions: ['geolocation'],
};
```

**Line ~710** (Proxy probe):
```javascript
context = await candidateBrowser.newContext({
  viewport: null,  // Changed from { width: 1280, height: 720 }
  permissions: ['geolocation'],
});
```

### 3. Expose VNC Port
File: `Dockerfile`
```dockerfile
EXPOSE 9377 6080  # Add 6080 for noVNC
```

## Environment Variables That Matter

| Variable | Default | Purpose | Notes |
|----------|---------|---------|-------|
| `NODE_ENV` | development | App mode | Set to `production` |
| `CAMOFOX_PORT` | 9377 | API server port | |
| `VNC_BIND` | 127.0.0.1 | VNC listen interface | **Must be 0.0.0.0 for Docker** |
| `VNC_RESOLUTION` | 1920x1080 | Virtual display size | Affects rendering |
| `VNC_PASSWORD` | (none) | VNC authentication | Optional |
| `VNC_PORT` | 5900 | x11vnc server port | |
| `NOVNC_PORT` | 6080 | websockify proxy port | |

## Common Failure Modes

### 1. Health Probe Constantly Failing
**Symptom**: Browser restarts every 3 minutes  
**Cause**: Viewport schema error in health probe  
**Fix**: Apply viewport: null fix to ALL context creations

### 2. "Failed to connect to server" in noVNC
**Symptom**: Web UI loads but can't connect  
**Cause**: x11vnc not running or listening on wrong interface  
**Fix**: 
```bash
# Manually start x11vnc:
docker exec camofox-browser \
  x11vnc -display :0 -forever -rfbport 5900 \
  -nopw -noxdamage -quiet -bg
```

### 3. "Cannot open display: [object Promise]"
**Symptom**: Xvfb fails to initialize  
**Cause**: Missing await on async display initialization  
**Fix**: Add `await` keyword on line 955

### 4. CDP Schema Validation Errors
**Symptom**: Any context creation fails with viewport error  
**Cause**: Multiple viewport configurations with problematic viewport  
**Fix**: Grep for all `viewport: {` and set to `viewport: null`

## Verification Checklist

- [ ] Xvfb virtual display running (`ps aux | grep Xvfb`)
- [ ] Camofox browser process running (`ps aux | grep camoufox-bin`)
- [ ] x11vnc listening on 5900 (`netstat -tlnp | grep 5900`)
- [ ] websockify listening on 6080 (`netstat -tlnp | grep 6080`)
- [ ] API responds (`curl http://localhost:9377/health`)
- [ ] noVNC HTML loads (`curl http://localhost:6080/vnc.html | head -20`)
- [ ] Tab creation works (`curl -X POST http://localhost:9377/tabs/open ...`)
- [ ] No health probe errors in logs (`docker logs | grep health`)

## Testing

### API Test
```bash
# Create a tab
curl -X POST http://localhost:9377/tabs/open \
  -H "Content-Type: application/json" \
  -d '{"userId":"test","url":"https://example.com"}'
```

### VNC Test
```bash
# Direct VNC connection
vncviewer localhost:5901

# Or use noVNC
# Open: http://localhost:6080/vnc.html
```

### Health Check
```bash
# Should return {"ok": true, "browserConnected": true}
curl http://localhost:9377/health | jq .
```

## Production Deployment Notes

1. **Always set `VNC_BIND=0.0.0.0`** unless using SSH tunneling
2. **Use `VNC_PASSWORD`** for security in shared environments
3. **Monitor health checks** — browser restarts indicate unhealthy state
4. **Persistent storage** — mount `/root/.camofox/profiles` for state persistence
5. **Resource limits** — Camofox uses ~250MB RAM at startup, grows with tabs
6. **Gradual rollout** — Test configuration changes with small instance first

## Gaps in Official Documentation

The Camofox documentation covers:
- ✅ Basic browser automation API
- ✅ Plugin system
- ✅ Configuration format

But does NOT cover:
- ❌ Docker deployment specifics
- ❌ Environment variable requirements for production
- ❌ Health probe configuration
- ❌ VNC/noVNC setup (mentions but doesn't fully document)
- ❌ Known CDP schema incompatibilities
- ❌ Container startup sequence and initialization order

This guide fills those gaps.
