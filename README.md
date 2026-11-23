# WebXR Multi-Format VR Video Platform & Station Manager

A serverless, browser-based virtual reality video platform built for **Google Cardboard** headsets on Android Chrome. Users scan one of 10 dedicated QR codes to launch an isolated VR experience—automatically configured for its specific projection and 3D stereo layout—while an edge API manages a 3-concurrent-viewer capacity limit and generates short-lived signed URLs for private cloud video streaming.

## Key Features

* **Multi-Format WebXR Rendering:** Dynamically adapts Three.js scene geometries and eye-layer UV mappings (`Layer 1` Left Eye / `Layer 2` Right Eye) to support four distinct media formats:

  * **360° Over-Under (3D):** Inverted sphere with top/bottom vertical eye separation.

  * **360° Monoscopic:** Inverted sphere rendered to both eyes.

  * **Flat / Wide-Angle Side-by-Side (3D):** Virtual cinema plane with left/right horizontal eye separation.

  * **Flat Monoscopic:** Virtual cinema plane centered at eye level.

* **Native WebXR Cardboard Integration:** Uses `navigator.xr.requestSession('immersive-vr')` over HTTPS for hardware-accelerated head tracking, IPD eye separation, and lens distortion calibration without requiring native app installations.

* **Secure Private Media Streaming:** Integrates with the **Backblaze B2 v4 API** via Cloudflare Pages Functions to issue temporary, 1-hour signed download tokens per session—keeping the storage bucket private while supporting progressive HTTP `206` byte-range streaming for multi-gigabyte 4K MP4s.

* **3-Station Concurrency Control & Admin Panel:** Uses **Cloudflare KV** to enforce a maximum of 3 simultaneous viewers with automatic heartbeat tracking, 45-second stale-session cleanup, and a live operator dashboard (`/admin.html`) for manual slot releases.

* **Built-In Diagnostics & QR Generator:** Includes a standalone storage/CORS diagnostic harness (`/test.html`) and a Node.js script (`generate-qr.js`) to batch-generate printable QR codes for all 10 experiences.

## System Architecture

```
┌─────────────────────────┐         1. POST /api/session/claim          ┌──────────────────────────────────┐
│   Android Chrome        │ ──────────────────────────────────────────► │  Cloudflare Pages Functions      │
│   (WebXR + Three.js)    │                                             │  (/api/* & /video/* routing)     │
│                         │ ◄────────────────────────────────────────── │                                  │
│   • Scans QR (/video/N) │         2. Slot Confirmed + Signed URL      └────────┬────────────────┬────────┘
│   • Renders Stereo VR   │                                                      │                │
└────────────┬────────────┘                                                      │                │
             │                                                       3a. Reserve │                │ 3b. Sign URL
             │ 4. Direct HTTPS Stream (206 Byte-Range + CORS)          Slot (1-3)│                │ (B2 v4 API)
             ▼                                                                   ▼                ▼
┌─────────────────────────┐                                             ┌─────────────────┐ ┌────────────────────┐
│  Backblaze B2 Storage   │                                             │  Cloudflare KV  │ │  Backblaze B2 API  │
│  (Private MP4 Bucket)   │                                             │  (VR_SESSIONS)  │ │  (Auth & Token)    │
└─────────────────────────┘                                             └─────────────────┘ └────────────────────┘

```

## Project Structure

```
vr-cloudflare-pages/
├── package.json
├── generate-qr.js              # CLI script to generate QR codes for /video/1..10
├── functions/
│   ├── api/
│   │   └── [[path]].js         # Edge API: session claiming, heartbeats, admin & B2 URL signing
│   └── video/
│       └── [[id]].js           # Edge rewrite routing /video/1..10 to /video.html
└── public/
    ├── video.html              # WebXR player entry page with Cardboard UI trigger
    ├── admin.html              # Operator dashboard for monitoring/releasing 3 viewer slots
    ├── test.html               # Standalone Backblaze B2 & CORS stream diagnostic tool
    ├── css/
    │   └── style.css           # Fullscreen viewport and overlay styling
    ├── js/
    │   └── player.js           # Three.js WebXR renderer, stereo UV mapper & session client
    └── vendor/
        └── three/              # Self-hosted Three.js ES module bundle (three.module.js + core)

```

## Video Projection Matrix

Each route (`/video/1` through `/video/10`) maps to a pre-configured rendering profile in `public/js/player.js`:

| Route | Experience | Projection | Stereo Layout | 
| ----- | ----- | ----- | ----- | 
| `/video/1` | 3D 360 Skydiving (Vuze 4K) | `360` Sphere | Over-Under (`ou`) | 
| `/video/2` | 3D Drone FPV Freestyle | `flat` Plane | Side-by-Side (`sbs`) | 
| `/video/3` | 3D SBS FPV Freestyle | `flat` Plane | Side-by-Side (`sbs`) | 
| `/video/4` | 360 Solar System VR | `360` Sphere | Monoscopic (`mono`) | 
| `/video/5` | Eagle POV: Disney Castle | `360` Sphere | Monoscopic (`mono`) | 
| `/video/6` | Explore Mars in 360 VR | `360` Sphere | Monoscopic (`mono`) | 
| `/video/7` | Fly Around the World 360 | `360` Sphere | Monoscopic (`mono`) | 
| `/video/8` | Ground to Space Hyperlapse | `360` Sphere | Monoscopic (`mono`) | 
| `/video/9` | Total Solar Eclipse 360 | `360` Sphere | Monoscopic (`mono`) | 
| `/video/10` | Solar System Experience | `flat` Plane | Monoscopic (`mono`) | 

## Setup & Deployment

### 1. Local Installation & Vendor Assets

Install dependencies and copy the complete Three.js build (including `three.core.js`) into `public/vendor/three/`:

```
npm install three qrcode wrangler
mkdir -p public/vendor/three qr-codes
cp node_modules/three/build/* public/vendor/three/

```

### 2. Configure Private Storage (Backblaze B2)

1. Create a **Private** bucket in Backblaze B2 and upload `video-01.mp4` through `video-10.mp4`.

2. Under **Bucket Settings → CORS Rules**, enable **Share everything in this bucket with every origin** so WebGL `THREE.VideoTexture` can read cross-origin video frames.

3. Create a **Read-Only** Application Key restricted to that single bucket and save the `keyID` and `applicationKey`.

### 3. Deploy to Cloudflare Pages

1. Authenticate Wrangler and deploy the project:

   ```
   npx wrangler login
   npx wrangler pages deploy public --project-name=vr-cardboard-player --branch=main
   
   ```

2. Add your Backblaze credentials as encrypted Pages secrets:

   ```
   npx wrangler pages secret put B2_KEY_ID --project-name=vr-cardboard-player
   npx wrangler pages secret put B2_APP_KEY --project-name=vr-cardboard-player
   
   ```

3. *(Optional)* Create a KV namespace (`npx wrangler kv namespace create VR_SESSIONS`) and bind it in **Cloudflare Dashboard → Workers & Pages → Settings → Bindings** under the variable name `VR_SESSIONS` to enforce the 3-station concurrency limit.

4. Redeploy once secrets and bindings are saved:

   ```
   npx wrangler pages deploy public --project-name=vr-cardboard-player --branch=main
   
   ```

### 4. Generate QR Codes

Generate PNG QR codes pointing to your live `*.pages.dev` deployment:

```
node generate-qr.js https://vr-cardboard-player.pages.dev

```

## Operations & Testing

* **Player URLs:** `https://<your-project>.pages.dev/video/1` through `/video/10`

* **Operator Admin Panel:** `https://<your-project>.pages.dev/admin.html` — displays active stations in real time and allows manual slot release.

* **Storage & CORS Diagnostic:** `https://<your-project>.pages.dev/test.html` — verifies Backblaze B2 token signing, CORS headers, and HTML5 video metadata loading independently of WebXR.
