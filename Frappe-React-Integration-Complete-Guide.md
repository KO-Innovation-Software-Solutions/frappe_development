# Building a Modern React Frontend for Frappe Framework Apps

## A Complete Guide to Integrating React with Frappe Using the Frappe React SDK

---

## Table of Contents

1. [Introduction & Architecture](#1-introduction--architecture)
2. [Prerequisites & Setup](#2-prerequisites--setup)
3. [Project Structure](#3-project-structure)
4. [Vite Configuration](#4-vite-configuration-for-frappe)
5. [Development Proxy](#5-development-proxy-configuration)
6. [The Critical Post-Build Script](#6-the-critical-post-build-script)
7. [Frappe React SDK Setup](#7-frappe-react-sdk-setup)
8. [Boot Data & CSRF Tokens](#8-boot-data--csrf-token-handling)
9. [Backend Python Configuration](#9-backend-python-configuration)
10. [Frappe Hooks Configuration](#10-frappe-hooks-configuration)
11. [Backend API Architecture](#11-backend-api-architecture)
12. [Using Frappe React SDK](#12-using-the-frappe-react-sdk-in-practice)
13. [Build & Deployment](#13-build--deployment-process)
14. [Development Workflow](#14-development-workflow)
15. [Best Practices](#15-common-patterns--best-practices)
16. [Troubleshooting](#16-troubleshooting-guide)
17. [Conclusion](#17-conclusion--next-steps)

---

## 1. Introduction & Architecture

### Why a Custom React Frontend for Frappe?

Frappe Framework is a powerful full-stack framework with a built-in admin interface (Desk). However, you might want a custom React frontend for:

- **Modern UX/UI**: Create beautiful, branded user experiences
- **Custom Workflows**: Build specialized interfaces for specific business processes
- **Mobile-First**: Develop responsive Progressive Web Apps (PWAs)
- **Third-Party Integration**: Integrate with external libraries and services
- **Performance**: Optimize for specific use cases

### Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    React Frontend (SPA)                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Components  │  │    Hooks     │  │   Routing    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│           │                │                │            │
│           └────────────────┴────────────────┘            │
│                           │                              │
│              ┌────────────▼────────────┐                 │
│              │  Frappe React SDK       │                 │
│              │  (SWR + API Client)     │                 │
│              └────────────┬────────────┘                 │
└───────────────────────────┼──────────────────────────────┘
                            │
                   HTTP/WebSocket
                            │
┌───────────────────────────▼──────────────────────────────┐
│                  Frappe Backend Server                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  API Routes  │  │  Whitelisted │  │   Database   │  │
│  │  (www/*.py)  │  │  Functions   │  │  (MariaDB)   │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│           │                │                │            │
│           └────────────────┴────────────────┘            │
│                   Session & Permissions                  │
└──────────────────────────────────────────────────────────┘
```

### Tech Stack

**Frontend:**
- **React 18** - UI library
- **TypeScript** - Type safety
- **Vite** - Build tool and dev server
- **Frappe React SDK** - API integration
- **React Router** - Client-side routing
- **SWR** - Data fetching and caching
- **TailwindCSS** - Styling (optional)

**Backend:**
- **Frappe Framework** - Python full-stack framework
- **MariaDB/MySQL** - Database
- **Redis** - Caching and real-time events
- **Socket.IO** - WebSocket support

### Key Concepts

**1. CSRF Tokens**: Cross-Site Request Forgery protection. Every API request must include a valid CSRF token.

**2. Boot Data**: Initial application data loaded from the server, including user session, permissions, and configuration.

**3. SWR Caching**: Stale-While-Revalidate pattern for efficient data fetching and caching.

**4. Whitelisted Functions**: Python functions decorated with `@frappe.whitelist()` that can be called from the frontend.

**5. Single Page Application (SPA)**: The React app handles routing client-side, with Frappe serving one HTML page for all routes.

---

## 2. Prerequisites & Setup

### Required Software

Before starting, ensure you have:

1. **Python 3.10+** - Frappe framework requirement
2. **Node.js 18+** - For React development
3. **MariaDB 10.6+** - Database server
4. **Redis** - For caching and real-time features
5. **Frappe Bench** - Frappe development environment

### Installing Frappe Bench

```bash
# Install frappe-bench
pip install frappe-bench

# Initialize a new bench
bench init frappe-bench --frappe-branch version-15
cd frappe-bench

# Start a new site
bench new-site mysite.localhost
bench use mysite.localhost
```

### Creating Your Frappe App

```bash
# Create a new Frappe app
bench new-app myapp

# Install the app on your site
bench --site mysite.localhost install-app myapp

# Start development server
bench start
```

### Project Structure

After creating your app, structure it for React integration:

```
frappe-bench/
└── apps/
    └── myapp/
        ├── myapp/                    # Python backend
        │   ├── __init__.py
        │   ├── hooks.py             # Frappe configuration
        │   ├── api/                 # API endpoints
        │   │   ├── __init__.py
        │   │   ├── config.py
        │   │   └── dynamic_data.py
        │   ├── www/                 # Web pages
        │   │   ├── myapp.py        # Page handler
        │   │   └── myapp.html      # Jinja template
        │   └── public/             # Static assets
        │       └── ui_assets/      # Built React app
        │
        ├── dashboard/               # React frontend
        │   ├── src/
        │   │   ├── App.tsx
        │   │   ├── main.tsx
        │   │   ├── components/
        │   │   ├── hooks/
        │   │   └── pages/
        │   ├── scripts/
        │   │   └── post-build.js   # Build post-processing
        │   ├── package.json
        │   ├── vite.config.ts
        │   ├── proxyOptions.js
        │   └── tsconfig.json
        │
        ├── pyproject.toml
        └── README.md
```

### Initialize React Project

```bash
cd apps/myapp
mkdir dashboard
cd dashboard

# Create Vite + React + TypeScript project
npm create vite@latest . -- --template react-ts

# Install dependencies
npm install

# Install Frappe React SDK
npm install frappe-react-sdk

# Install additional dependencies
npm install react-router-dom axios js-cookie
npm install @types/js-cookie --save-dev
```

---

## 3. Project Structure

### Frontend Directory Structure

```
dashboard/
├── src/
│   ├── main.tsx                 # Entry point with boot data loading
│   ├── App.tsx                  # Root component with FrappeProvider
│   ├── index.css                # Global styles
│   │
│   ├── components/              # Reusable components
│   │   ├── layout/
│   │   │   ├── Header.tsx
│   │   │   ├── Sidebar.tsx
│   │   │   └── Layout.tsx
│   │   └── ui/                  # UI components
│   │
│   ├── pages/                   # Route pages
│   │   ├── Dashboard.tsx
│   │   ├── List.tsx
│   │   └── Form.tsx
│   │
│   ├── hooks/                   # Custom hooks
│   │   ├── use-doctype-crud.ts
│   │   ├── use-doctype-data.ts
│   │   └── use-dashboard-config.ts
│   │
│   ├── providers/               # Context providers
│   │   ├── index.tsx
│   │   └── theme-provider.tsx
│   │
│   ├── routes/                  # Routing configuration
│   │   └── index.tsx
│   │
│   └── types/                   # TypeScript types
│       └── index.ts
│
├── scripts/
│   └── post-build.js            # Post-build processing
│
├── public/                       # Static assets
│   └── vite.svg
│
├── index.html                    # HTML template
├── vite.config.ts               # Vite configuration
├── proxyOptions.js              # Development proxy
├── package.json                 # Dependencies and scripts
└── tsconfig.json                # TypeScript configuration
```

### Backend Directory Structure

```
myapp/
├── api/                         # API endpoints
│   ├── __init__.py
│   ├── config.py               # App configuration
│   ├── dashboard_config.py     # Dashboard management
│   ├── dynamic_data.py         # Generic CRUD operations
│   └── chat.py                 # Real-time chat (if needed)
│
├── www/                        # Web pages
│   ├── __init__.py
│   ├── myapp.py               # Page handler
│   └── myapp.html             # HTML template (generated)
│
├── public/                     # Static assets
│   └── ui_assets/             # Built React app (generated)
│       ├── assets/
│       │   ├── index-xxx.js
│       │   └── index-xxx.css
│       └── index.html
│
├── hooks.py                    # Frappe configuration
├── auth.py                     # Session hooks
└── __init__.py
```

---

## 4. Vite Configuration for Frappe

### vite.config.ts

Create a Vite configuration that works with Frappe's asset structure:

```typescript
import path from 'path';
import { defineConfig, loadEnv } from 'vite';
import react from '@vitejs/plugin-react-swc';
import proxyOptions from './proxyOptions';

export default defineConfig(({ command, mode }) => {
    const env = loadEnv(mode, process.cwd(), "");
    
    return {
        plugins: [react()],
        
        server: {
            port: 3000,
            proxy: proxyOptions  // Development proxy
        },
        
        resolve: {
            alias: {
                "@": path.resolve(__dirname, "./src")
            }
        },
        
        build: {
            // CRITICAL: Output to Frappe public directory
            outDir: "../myapp/public/ui_assets",
            emptyOutDir: true,
            target: "es2015",
            
            rollupOptions: {
                onwarn(warning, warn) {
                    if (warning.code === "MODULE_LEVEL_DIRECTIVE") {
                        return;
                    }
                    warn(warning);
                }
            }
        },
        
        define: {
            // Make env variables available to client
            'import.meta.env.VITE_SOCKET_PORT': JSON.stringify(env.VITE_SOCKET_PORT || '9000'),
            'import.meta.env.VITE_SITE_NAME': JSON.stringify(env.VITE_SITE_NAME || 'mysite.localhost'),
            'import.meta.env.VITE_BASE_NAME': JSON.stringify(env.VITE_BASE_NAME || 'myapp'),
        }
    };
});
```

### Key Configuration Points

**1. Base Path for Assets**

When running `npm run build`, you need to specify the base path:

```json
// package.json
{
  "scripts": {
    "build": "tsc && vite build --base=/assets/myapp/ui_assets/ && node scripts/post-build.js"
  }
}
```

The `--base=/assets/myapp/ui_assets/` ensures all asset references use the correct path when served by Frappe.

**2. Output Directory**

```typescript
outDir: "../myapp/public/ui_assets"
```

This places the built files where Frappe can serve them as static assets at `/assets/myapp/ui_assets/`.

**3. Path Aliases**

```typescript
resolve: {
    alias: {
        "@": path.resolve(__dirname, "./src")
    }
}
```

Allows imports like `import Button from '@/components/ui/button'` instead of relative paths.

---

## 5. Development Proxy Configuration

### proxyOptions.js

During development, your React app (port 3000) needs to communicate with Frappe backend (port 8000). The proxy handles this:

```javascript
import path from 'path';
import fs from 'fs';

function getCommonSiteConfig() {
    let currentDir = path.resolve('.');
    
    // Traverse up to find frappe-bench directory
    while (currentDir !== '/') {
        if (
            fs.existsSync(path.join(currentDir, 'sites')) &&
            fs.existsSync(path.join(currentDir, 'apps'))
        ) {
            let configPath = path.join(currentDir, 'sites', 'common_site_config.json');
            if (fs.existsSync(configPath)) {
                return JSON.parse(fs.readFileSync(configPath));
            }
            return null;
        }
        currentDir = path.resolve(currentDir, '..');
    }
    return null;
}

const config = getCommonSiteConfig();
const webserver_port = config ? config.webserver_port : 8000;

if (!config) {
    console.log('No common_site_config.json found, using default port 8000');
}

export default {
    // Proxy API, assets, files, and private endpoints
    '^/(app|api|assets|files|private)': {
        target: `http://127.0.0.1:${webserver_port}`,
        ws: true,                    // WebSocket support
        changeOrigin: true,
        secure: false,
        router: function (req) {
            // Multi-site support
            const site_name = req.headers.host.split(':')[0];
            return `http://${site_name}:${webserver_port}`;
        }
    }
};
```

### How the Proxy Works

```
┌─────────────────┐
│  React Dev      │
│  Server         │
│  localhost:3000 │
└────────┬────────┘
         │
         │ /api/method/myapp.api.config.get_app_config
         │
         ▼
┌─────────────────┐
│  Vite Proxy     │
│  (proxyOptions) │
└────────┬────────┘
         │
         │ Forwards to Frappe
         │
         ▼
┌─────────────────┐
│  Frappe Server  │
│  localhost:8000 │
└─────────────────┘
```

**What gets proxied:**
- `/api/*` - API endpoints
- `/assets/*` - Static assets
- `/files/*` - File uploads/downloads
- `/private/*` - Private files
- `/app/*` - Frappe Desk routes

**Why this works:**
- CORS issues avoided (same origin)
- Cookies preserved (authentication)
- WebSocket connections maintained
- Multi-site support via hostname routing

---

## 6. The Critical Post-Build Script

### Why Post-Build Processing?

When Vite builds your React app, it creates a static `index.html` with hardcoded values. But Frappe needs:
- Dynamic CSRF tokens
- Server-side boot data
- User session information
- Jinja template variables

The post-build script converts the static HTML into a Frappe-compatible Jinja template.

### scripts/post-build.js

```javascript
import fs from 'fs';
import path from 'path';

const buildDir = '../myapp/public/ui_assets';
const wwwDir = '../myapp/www';
const builtHtmlPath = path.join(buildDir, 'index.html');
const targetHtmlPath = path.join(wwwDir, 'myapp.html');

// Read the built HTML file
let htmlContent = fs.readFileSync(builtHtmlPath, 'utf8');

// Replace the hardcoded title with Jinja templating
htmlContent = htmlContent.replace(
    '<title>My App</title>',
    '<title>{{ app_name }}</title>'
);

// Add favicon templating
htmlContent = htmlContent.replace(
    '<link rel="icon" type="image/svg+xml" href="/vite.svg" />',
    `<link rel="icon" type="image/x-icon" href="{{ favicon_ico }}" />
    <link rel="icon" type="image/png" href="{{ icon_96 }}" sizes="96x96" />
    <link rel="icon" type="image/svg+xml" href="{{ favicon_svg }}" />
    <link rel="apple-touch-icon" sizes="180x180" href="{{ apple_touch_icon }}">
    <link rel="mask-icon" href="{{ mask_icon }}" color="#000000">`
);

// Add meta tags
htmlContent = htmlContent.replace(
    '<meta name="viewport" content="width=device-width, initial-scale=1.0" />',
    `<meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <meta name="description" content="My App Dashboard">
    <meta name="theme-color" content="#191919">
    
    <!-- PWA Meta Tags -->
    <meta name="apple-mobile-web-app-capable" content="yes" />
    <meta name="apple-mobile-web-app-title" content="{{ app_name }}" />
    
    <!-- Preload Links -->
    {{ preload_links | safe }}`
);

// 🔥 CRITICAL: Inject CSRF token and boot data
const developmentBootScript = htmlContent.match(/<script>[\s\S]*?<\/script>/);
if (developmentBootScript) {
    const frappeBootScript = `<script>
        // CRITICAL: Set CSRF token from server
        window.csrf_token = '{{ csrf_token }}';
        
        // Initialize frappe global object
        if (!window.frappe) window.frappe = {};
        window.app_name = "{{ app_name }}";
        
        // Parse boot data from server
        frappe.boot = JSON.parse({{ boot }});
    </script>`;
    
    htmlContent = htmlContent.replace(developmentBootScript[0], frappeBootScript);
}

// Write the processed HTML to www directory
fs.writeFileSync(targetHtmlPath, htmlContent);

console.log('✅ HTML template processed and copied to www/myapp.html');
console.log('✅ Frappe templating injected successfully');
```

### What This Script Does

**1. CSRF Token Injection**
```javascript
window.csrf_token = '{{ csrf_token }}';
```
The Frappe server will replace `{{ csrf_token }}` with the actual token for each user session.

**2. Boot Data Injection**
```javascript
frappe.boot = JSON.parse({{ boot }});
```
Server-side boot data (user info, permissions, configuration) is injected as JSON.

**3. Dynamic Title & Metadata**
```html
<title>{{ app_name }}</title>
```
Site name and branding come from Frappe settings.

**4. File Location**
- Input: `myapp/public/ui_assets/index.html` (static)
- Output: `myapp/www/myapp.html` (Jinja template)

---

## 7. Frappe React SDK Setup

### App.tsx - Root Component

```typescript
import { Toaster } from '@/components/ui/sonner';
import { FrappeProvider } from 'frappe-react-sdk';
import './index.css';
import AppProvider from './providers';
import AppRouter from './routes';

function App() {
    // Handle Frappe version compatibility for site name
    const getSiteName = () => {
        // @ts-ignore
        if (window.frappe?.boot?.versions?.frappe?.startsWith('14')) {
            return import.meta.env.VITE_SITE_NAME;
        } else {
            // @ts-ignore
            return window.frappe?.boot?.sitename ?? import.meta.env.VITE_SITE_NAME;
        }
    };

    // SWR caching with localStorage
    function localStorageProvider() {
        // Check if cache is recent (less than a week)
        const timestamp = localStorage.getItem('myapp-cache-timestamp');
        let cache = '[]';
        
        if (timestamp && Date.now() - parseInt(timestamp) < 7 * 24 * 60 * 60 * 1000) {
            const localCache = localStorage.getItem('myapp-cache');
            if (localCache) {
                cache = localCache;
            }
        }
        
        const map = new Map<string, any>(JSON.parse(cache));

        // Save cache on page unload
        window.addEventListener('beforeunload', () => {
            const entries = Array.from(map.entries());
            const appCache = JSON.stringify(entries);
            localStorage.setItem('myapp-cache', appCache);
            localStorage.setItem('myapp-cache-timestamp', Date.now().toString());
        });

        return map;
    }

    return (
        <div className="App">
            <FrappeProvider
                url={import.meta.env.VITE_FRAPPE_PATH ?? ''}
                socketPort={import.meta.env.VITE_SOCKET_PORT}
                siteName={getSiteName()}
                swrConfig={{
                    errorRetryCount: 2,
                    provider: localStorageProvider
                }}
            >
                <AppProvider>
                    <AppRouter />
                </AppProvider>
                <Toaster />
            </FrappeProvider>
        </div>
    )
}

export default App
```

### FrappeProvider Configuration

**url**: Base URL for API calls (empty string uses current domain)
**socketPort**: WebSocket port for real-time features
**siteName**: Frappe site name (for multi-tenancy)
**swrConfig**: SWR configuration for data fetching

### SWR Caching Strategy

```typescript
function localStorageProvider() {
    // 1. Load existing cache from localStorage
    const cache = localStorage.getItem('myapp-cache');
    const map = new Map(JSON.parse(cache || '[]'));
    
    // 2. Save cache before page unload
    window.addEventListener('beforeunload', () => {
        const entries = Array.from(map.entries());
        localStorage.setItem('myapp-cache', JSON.stringify(entries));
    });
    
    return map;
}
```

**Benefits:**
- Persists cache across page reloads
- Instant data on revisits
- Automatically revalidates stale data
- 7-day expiry for freshness

---

## 8. Boot Data & CSRF Token Handling

### Dual-Mode Architecture

The app works differently in development vs production:

### Development Mode (React Dev Server)

```typescript
// src/main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App.tsx';
import './index.css';

declare global {
    interface Window {
        frappe: {
            [key: string]: any;
            boot: any;
        };
    }
}

const renderApp = () => {
    // Set favicon from boot data
    const favicon = window.frappe.boot?.myapp_config?.favicon;
    if (favicon) {
        const link: HTMLLinkElement | null = document.querySelector("link[rel*='icon']");
        if (link) {
            link.href = favicon;
        }
    }

    ReactDOM.createRoot(document.getElementById('root')!).render(
        <React.StrictMode>
            <App />
        </React.StrictMode>
    );
};

if (import.meta.env.DEV) {
    // 🔥 DEVELOPMENT: Fetch boot data from API
    fetch('/api/method/myapp.www.myapp.get_context_for_dev', {
        method: 'POST',
    })
    .then(response => response.json())
    .then((values) => {
        const bootData = JSON.parse(values.message);
        
        if (!window.frappe) window.frappe = { boot: {} };
        window.frappe.boot = bootData;
        
        renderApp();
    })
    .catch(error => {
        console.error('Failed to load boot data:', error);
        
        // Fallback boot data for development
        if (!window.frappe) window.frappe = { boot: {} };
        window.frappe.boot = {
            versions: { frappe: '15.0.0' },
            sitename: 'mysite.localhost',
            user: { name: 'Administrator' }
        };
        renderApp();
    });
} else {
    // 🔥 PRODUCTION: Boot data already injected by server
    renderApp();
}
```

### Production Mode (Served by Frappe)

In production, boot data is injected directly into the HTML by the server:

```html
<!-- myapp/www/myapp.html (generated) -->
<script>
    window.csrf_token = 'AbC123XyZ...';
    if (!window.frappe) window.frappe = {};
    frappe.boot = JSON.parse({"user":"Administrator","sitename":"mysite.localhost",...});
</script>
```

### Comparison

| Aspect | Development | Production |
|--------|------------|------------|
| Boot Data Source | API call | Server injection |
| CSRF Token | Not needed (proxy) | Required |
| Speed | Slower (extra request) | Faster (embedded) |
| Hot Reload | ✅ Yes | ❌ No |

---

## 9. Backend Python Configuration

### www/myapp.py - Page Handler

This file handles the `/myapp` route and prepares data for the Jinja template:

```python
import json
import re

import frappe
import frappe.sessions
from frappe import _
from frappe.utils.telemetry import capture

no_cache = 1  # Disable caching

SCRIPT_TAG_PATTERN = re.compile(r"\<script[^<]*\</script\>")
CLOSING_SCRIPT_TAG_PATTERN = re.compile(r"</script\>")


def get_context(context):
    """
    Called by Frappe when rendering www/myapp.html
    Returns context dictionary for Jinja template
    """
    
    # 🔥 CRITICAL: Generate and commit CSRF token
    csrf_token = frappe.sessions.get_csrf_token()
    frappe.db.commit()  # Commit immediately
    
    # Get boot data (user session, permissions, etc.)
    if frappe.session.user == "Guest":
        boot = frappe.website.utils.get_boot_data()
    else:
        try:
            boot = frappe.sessions.get()
        except Exception as e:
            raise frappe.SessionBootFailed from e
    
    # Add custom boot data for your app
    boot["myapp_config"] = {
        "version": "1.0.0",
        "app_name": "My App Dashboard"
    }
    
    # Add server script configuration
    if "server_script_enabled" in frappe.conf:
        enabled = frappe.conf.server_script_enabled
    else:
        enabled = True
    boot["server_script_enabled"] = enabled
    
    # Convert boot data to JSON
    boot_json = frappe.as_json(boot, indent=None, separators=(",", ":"))
    
    # 🔥 SECURITY: Remove script tags to prevent XSS
    boot_json = SCRIPT_TAG_PATTERN.sub("", boot_json)
    boot_json = CLOSING_SCRIPT_TAG_PATTERN.sub("", boot_json)
    boot_json = json.dumps(boot_json)
    
    # Prepare context for Jinja template
    context.update({
        "build_version": frappe.utils.get_build_version(),
        "boot": boot_json,
        "csrf_token": csrf_token
    })
    
    # App name for title
    app_name = frappe.get_website_settings("app_name") or frappe.get_system_settings("app_name")
    if app_name and app_name != "Frappe":
        context["app_name"] = app_name + " | " + "My App"
    else:
        context["app_name"] = "My App Dashboard"
    
    # Favicon configuration
    try:
        favicon = frappe.db.get_single_value("Website Settings", "favicon")
    except Exception:
        favicon = "/assets/myapp/favicon.png"
    
    context["icon_96"] = favicon or "/assets/myapp/favicon-96x96.png"
    context["apple_touch_icon"] = favicon or "/assets/myapp/apple-touch-icon.png"
    context["mask_icon"] = favicon or "/assets/myapp/safari-pinned-tab.svg"
    context["favicon_svg"] = favicon or "/assets/myapp/favicon.svg"
    context["favicon_ico"] = favicon or "/assets/myapp/favicon.ico"
    context["sitename"] = boot.get("sitename")
    
    # Preload critical resources
    if frappe.session.user != "Guest":
        capture("active_site", "myapp")
        context["preload_links"] = """
            <link rel="preload" href="/api/method/frappe.auth.get_logged_user" as="fetch" crossorigin="use-credentials">
        """
    else:
        context["preload_links"] = ""
    
    return context


@frappe.whitelist(methods=["POST"], allow_guest=True)
def get_context_for_dev():
    """
    Development-only endpoint for React dev server
    Returns boot data as JSON
    """
    if not frappe.conf.developer_mode:
        frappe.throw(_("This method is only meant for developer mode"))
    return json.loads(get_boot())


def get_boot():
    """Helper to get boot data as JSON"""
    try:
        boot = frappe.sessions.get()
    except Exception as e:
        raise frappe.SessionBootFailed from e
    
    boot["myapp_config"] = {
        "version": "1.0.0",
        "app_name": "My App Dashboard"
    }
    
    boot_json = frappe.as_json(boot, indent=None, separators=(",", ":"))
    boot_json = SCRIPT_TAG_PATTERN.sub("", boot_json)
    boot_json = CLOSING_SCRIPT_TAG_PATTERN.sub("", boot_json)
    boot_json = json.dumps(boot_json)
    
    return boot_json
```

### Key Functions Explained

**1. get_context(context)**
- Called when Frappe serves `/myapp` route
- Generates CSRF token and commits to database
- Loads user session boot data
- Returns context dictionary for Jinja template

**2. get_context_for_dev()**
- Development-only endpoint
- Allows React dev server to fetch boot data via API
- Only works when `developer_mode = 1` in site_config.json

**3. Boot Data Structure**

```python
{
    "user": "administrator@example.com",
    "sitename": "mysite.localhost",
    "versions": {"frappe": "15.0.0"},
    "csrf_token": "...",
    "user_info": {...},
    "system_settings": {...},
    "myapp_config": {  # Custom app config
        "version": "1.0.0",
        "app_name": "My App Dashboard"
    }
}
```

---

## 10. Frappe Hooks Configuration

### hooks.py - Central Configuration

```python
app_name = "myapp"
app_title = "My App"
app_publisher = "Your Company"
app_description = "My App Dashboard"
app_email = "info@yourcompany.com"
app_license = "mit"

# 🔥 CRITICAL: Website routing for SPA
website_route_rules = [
    {"from_route": "/myapp/<path:app_path>", "to_route": "myapp"},
    {"from_route": "/myapp", "to_route": "myapp"},
]

# Add to Frappe apps launcher
add_to_apps_screen = [
    {
        "name": "myapp",
        "logo": "/assets/myapp/favicon.png",
        "title": "My App",
        "route": "/myapp",
        # Optional: permission check
        # "has_permission": "myapp.api.permission.has_app_permission"
    }
]

# Session hooks
on_session_creation = "myapp.auth.on_session_creation"
on_logout = "myapp.auth.on_logout"

# Document events (optional)
doc_events = {
    "*": {
        "on_update": "myapp.controllers.document_events.handle_update",
    }
}

# Scheduled tasks (optional)
# scheduler_events = {
#     "daily": [
#         "myapp.tasks.daily"
#     ],
# }

# Sounds for real-time notifications (optional)
sounds = [
    {
        'name': 'notification',
        'src': '/assets/myapp/sounds/notification.mp3',
        'volume': 0.2
    },
]
```

### Key Configurations Explained

**1. Website Route Rules** ⭐ Most Important

```python
website_route_rules = [
    {"from_route": "/myapp/<path:app_path>", "to_route": "myapp"},
    {"from_route": "/myapp", "to_route": "myapp"},
]
```

This tells Frappe:
- Map `/myapp` → `www/myapp.py`
- Map `/myapp/*` → `www/myapp.py` (catch-all)
- Enables client-side routing (React Router)

**Without this:** Page refresh breaks React Router!

**2. Add to Apps Screen**

```python
add_to_apps_screen = [{
    "name": "myapp",
    "logo": "/assets/myapp/favicon.png",
    "title": "My App",
    "route": "/myapp"
}]
```

Makes your app appear in Frappe's app launcher (desk homepage).

**3. Session Hooks**

```python
on_session_creation = "myapp.auth.on_session_creation"
on_logout = "myapp.auth.on_logout"
```

Run custom code on user login/logout.

Example `auth.py`:
```python
def on_session_creation(login_manager):
    """Called when user logs in"""
    frappe.logger().info(f"User {login_manager.user} logged in")

def on_logout():
    """Called when user logs out"""
    frappe.logger().info(f"User {frappe.session.user} logged out")
```

---

## 11. Backend API Architecture

### Overview of API Structure

```
myapp/api/
├── __init__.py
├── config.py             # App configuration endpoints
├── dashboard_config.py   # Dashboard/sidebar management
├── dynamic_data.py       # Generic CRUD for any DocType
└── chat.py              # Real-time chat (optional)
```

### Creating Whitelisted API Endpoints

**Basic Pattern:**

```python
import frappe

@frappe.whitelist()
def my_function(param1, param2):
    """
    This function can be called from frontend
    URL: /api/method/myapp.api.config.my_function
    """
    # Check permissions
    if not frappe.has_permission("DocType", "read"):
        frappe.throw("Not permitted")
    
    # Your logic here
    result = {"key": "value"}
    
    return result
```

**Frontend call:**

```typescript
import { useFrappeGetCall } from 'frappe-react-sdk';

const { data } = useFrappeGetCall('myapp.api.config.my_function', {
    param1: 'value1',
    param2: 'value2'
});
```

### API Decorator Options

```python
# Requires authentication
@frappe.whitelist()
def authenticated_function():
    pass

# Allows guest users
@frappe.whitelist(allow_guest=True)
def public_function():
    pass

# Restrict to specific HTTP methods
@frappe.whitelist(methods=["POST"])
def post_only_function():
    pass

@frappe.whitelist(methods=["GET", "POST"])
def get_or_post_function():
    pass
```

### api/config.py - Application Configuration

```python
import frappe

@frappe.whitelist(allow_guest=True)
def get_app_config():
    """Returns app-specific configuration"""
    config = {
        'socketio_port': frappe.conf.socketio_port,
        'user_email': frappe.session.user,
        'app_version': '1.0.0',
        'site_name': frappe.local.site
    }
    return config
```

**Usage in React:**

```typescript
import { useFrappeGetCall } from 'frappe-react-sdk';

const { data: config } = useFrappeGetCall('myapp.api.config.get_app_config');

console.log(config?.socketio_port);
```

### api/dynamic_data.py - Generic CRUD Engine

This file provides reusable CRUD operations for any DocType:

**Decorator for Error Handling:**

```python
from functools import wraps

def frappe_api_handler(func):
    """Consistent error handling across API functions"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except frappe.ValidationError:
            raise  # Re-raise validation errors
        except Exception as e:
            frappe.log_error(f"Error in {func.__name__}: {str(e)}")
            frappe.throw(_("An error occurred: {0}").format(str(e)))
    return wrapper
```

**Permission Validation Helper:**

```python
def validate_doctype_and_permissions(doctype_name, perm_type, name=None):
    """Validate doctype exists and check permissions"""
    if not frappe.db.exists("DocType", doctype_name):
        frappe.throw(_("DocType {0} does not exist").format(doctype_name))
    
    if not frappe.has_permission(doctype_name, perm_type, name):
        target = f"{doctype_name} {name}" if name else doctype_name
        frappe.throw(_("Not permitted to {0} {1}").format(perm_type, target))
    
    return frappe.get_meta(doctype_name)
```

**Get DocType Data (List):**

```python
@frappe.whitelist()
@frappe_api_handler
def get_doctype_data(doctype_name, fields=None, filters=None, limit=20, start=0, order_by=None):
    """Get data for any doctype with pagination"""
    
    # Validate permissions
    meta = validate_doctype_and_permissions(doctype_name, "read")
    
    # Parse fields
    if isinstance(fields, str):
        fields = [f.strip() for f in fields.split(",")]
    if not fields:
        fields = ["name", "modified"]
    
    # Parse filters
    if isinstance(filters, str):
        filters = json.loads(filters)
    filters = filters or {}
    
    # Get data
    data = frappe.get_all(
        doctype_name,
        fields=fields,
        filters=filters,
        limit_start=int(start),
        limit=int(limit),
        order_by=order_by or "modified desc"
    )
    
    # Get total count
    total = frappe.db.count(doctype_name, filters)
    
    return {
        "data": data,
        "total": total,
        "page_info": {
            "start": start,
            "limit": limit,
            "has_next": int(start) + int(limit) < total
        }
    }
```

**Usage in React:**

```typescript
const { data } = useFrappeGetCall('myapp.api.dynamic_data.get_doctype_data', {
    doctype_name: 'Customer',
    fields: 'name,customer_name,email',
    limit: 20,
    start: 0
});
```

**Get DocType Metadata:**

```python
@frappe.whitelist()
@frappe_api_handler
def get_doctype_meta(doctype_name):
    """Get metadata for a doctype including field definitions"""
    
    meta = validate_doctype_and_permissions(doctype_name, "read")
    
    # Get field information
    fields = []
    for field in meta.fields:
        if field.fieldtype not in ["Section Break", "Column Break", "HTML", "Button"]:
            fields.append({
                "fieldname": field.fieldname,
                "label": field.label,
                "fieldtype": field.fieldtype,
                "options": field.options,
                "reqd": field.reqd,
                "read_only": field.read_only,
                "hidden": field.hidden,
                "in_list_view": field.in_list_view
            })
    
    return {
        "name": meta.name,
        "module": meta.module,
        "title_field": meta.title_field,
        "is_submittable": meta.is_submittable,
        "fields": fields,
        "permissions": {
            "read": frappe.has_permission(doctype_name, "read"),
            "write": frappe.has_permission(doctype_name, "write"),
            "create": frappe.has_permission(doctype_name, "create"),
            "delete": frappe.has_permission(doctype_name, "delete")
        }
    }
```

**Create Record:**

```python
@frappe.whitelist(methods=["POST"])
@frappe_api_handler
def create_doctype_record(doctype_name, data):
    """Create a new record"""
    
    validate_doctype_and_permissions(doctype_name, "create")
    
    if isinstance(data, str):
        data = json.loads(data)
    
    doc = frappe.get_doc({
        "doctype": doctype_name,
        **data
    })
    doc.insert()
    
    return {
        "name": doc.name,
        "message": _("{0} created successfully").format(doctype_name)
    }
```

**Update Record:**

```python
@frappe.whitelist(methods=["PUT"])
@frappe_api_handler
def update_doctype_record(doctype_name, name, data):
    """Update an existing record"""
    
    validate_doctype_and_permissions(doctype_name, "write", name)
    
    if not frappe.db.exists(doctype_name, name):
        frappe.throw(_("{0} {1} does not exist").format(doctype_name, name))
    
    if isinstance(data, str):
        data = json.loads(data)
    
    doc = frappe.get_doc(doctype_name, name)
    
    for key, value in data.items():
        if hasattr(doc, key):
            doc.set(key, value)
    
    doc.save()
    
    return {
        "name": doc.name,
        "message": _("{0} updated successfully").format(doctype_name)
    }
```

**Delete Record:**

```python
@frappe.whitelist(methods=["DELETE"])
def delete_doctype_record(doctype_name=None, name=None):
    """Delete a record"""
    
    if not doctype_name or not name:
        return {"error": "doctype_name and name are required"}
    
    try:
        validate_doctype_and_permissions(doctype_name, "delete", name)
        
        if not frappe.db.exists(doctype_name, name):
            return {"error": "Record not found"}
        
        frappe.delete_doc(doctype_name, name)
        
        return {
            "message": _("{0} {1} deleted successfully").format(doctype_name, name)
        }
    except Exception as e:
        frappe.log_error(f"Error deleting: {str(e)}")
        return {"error": str(e)}
```

### API Path Convention

```
Python Module:    myapp/api/config.py
Function:         get_app_config()
API Endpoint:     /api/method/myapp.api.config.get_app_config

Python Module:    myapp/api/dynamic_data.py
Function:         get_doctype_data()
API Endpoint:     /api/method/myapp.api.dynamic_data.get_doctype_data
```

The pattern is: `/api/method/{app_name}.{module_path}.{function_name}`

### Security Best Practices

**1. Always Check Permissions**

```python
if not frappe.has_permission(doctype_name, "read"):
    frappe.throw("Not permitted")
```

**2. Validate Input**

```python
if not frappe.db.exists("DocType", doctype_name):
    frappe.throw("DocType does not exist")

# Validate email
frappe.utils.validate_email_address(email, throw=True)
```

**3. Use Parameterized Queries**

```python
# ❌ BAD - SQL injection risk
query = f"SELECT * FROM tabUser WHERE name = '{user}'"

# ✅ GOOD - parameterized
query = "SELECT * FROM tabUser WHERE name = %(user)s"
frappe.db.sql(query, {"user": user})
```

**4. Log Errors**

```python
try:
    # operation
except Exception as e:
    frappe.log_error(f"Error: {str(e)}", "My App Error")
    frappe.throw("Operation failed")
```

---

## 12. Using the Frappe React SDK in Practice

### Custom Hook: use-doctype-crud.ts

Create reusable hooks for common operations:

```typescript
import { 
    useFrappeCreateDoc, 
    useFrappeUpdateDoc, 
    useFrappeDeleteDoc,
    useFrappeGetDoc 
} from 'frappe-react-sdk';
import { toast } from 'sonner';

export interface DoctypeCrudOptions {
    doctype: string;
    onSuccess?: () => void;
    onError?: (error: any) => void;
}

export const useDoctypeCrud = (options: DoctypeCrudOptions) => {
    const { doctype, onSuccess, onError } = options;

    // Use native Frappe SDK hooks
    const { createDoc, loading: creating } = useFrappeCreateDoc();
    const { updateDoc, loading: updating } = useFrappeUpdateDoc();
    const { deleteDoc, loading: deleting } = useFrappeDeleteDoc();

    // Create with toast notifications
    const handleCreate = async (data: any) => {
        try {
            const result = await createDoc(doctype, data);
            toast.success(`${doctype} created successfully`);
            onSuccess?.();
            return result;
        } catch (error) {
            toast.error(`Failed to create ${doctype}`);
            onError?.(error);
            throw error;
        }
    };

    // Update with toast notifications
    const handleUpdate = async (name: string, data: any) => {
        try {
            const result = await updateDoc(doctype, name, data);
            toast.success(`${doctype} updated successfully`);
            onSuccess?.();
            return result;
        } catch (error) {
            toast.error(`Failed to update ${doctype}`);
            onError?.(error);
            throw error;
        }
    };

    // Delete with toast notifications
    const handleDelete = async (name: string) => {
        try {
            await deleteDoc(doctype, name);
            toast.success(`${doctype} deleted successfully`);
            onSuccess?.();
        } catch (error) {
            toast.error(`Failed to delete ${doctype}`);
            onError?.(error);
            throw error;
        }
    };

    // Get single record
    const useSingleRecord = (name?: string) => {
        const shouldFetch = Boolean(name && name.trim());
        
        if (!shouldFetch) {
            return {
                data: null,
                isLoading: false,
                error: null,
                mutate: () => {},
            };
        }
        
        return useFrappeGetDoc(doctype, name, {
            revalidateOnFocus: false,
        });
    };

    return {
        createRecord: handleCreate,
        updateRecord: handleUpdate,
        deleteRecord: handleDelete,
        useSingleRecord,
        isLoading: creating || updating || deleting,
        creating,
        updating,
        deleting,
    };
};
```

**Usage Example:**

```typescript
function CustomerForm() {
    const { createRecord, updateRecord, deleteRecord, useSingleRecord } = useDoctypeCrud({
        doctype: 'Customer',
        onSuccess: () => {
            console.log('Success!');
            navigate('/customers');
        }
    });
    
    // Get existing record for editing
    const { data: customer, isLoading } = useSingleRecord('CUST-001');
    
    const handleSubmit = async (formData) => {
        if (customer) {
            await updateRecord(customer.name, formData);
        } else {
            await createRecord(formData);
        }
    };
    
    return (
        <form onSubmit={handleSubmit}>
            {/* form fields */}
        </form>
    );
}
```

### Custom Hook: use-doctype-data.ts

For listing and fetching data:

```typescript
import { useFrappeGetDocList, useFrappeGetCall } from 'frappe-react-sdk';
import { useMemo } from 'react';

export interface DoctypeDataOptions {
    doctype: string;
    fields?: string[];
    filters?: Record<string, any>;
    limit?: number;
    orderBy?: string;
}

export const useDoctypeData = (options: DoctypeDataOptions) => {
    const { doctype, fields, limit = 20, orderBy } = options;

    // Get doctype metadata
    const { data: metaData } = useFrappeGetCall(
        'myapp.api.dynamic_data.get_doctype_meta',
        { doctype_name: doctype },
        {
            revalidateOnFocus: false,
            dedupingInterval: 300000, // 5 minutes cache
        }
    );

    // Convert orderBy to SDK format
    const orderByConfig = useMemo(() => {
        if (!orderBy) return { field: 'modified', order: 'desc' as const };
        const [field, order] = orderBy.split(' ');
        return {
            field,
            order: (order?.toLowerCase() === 'asc' ? 'asc' : 'desc') as 'asc' | 'desc'
        };
    }, [orderBy]);

    // Resolve fields from metadata
    const resolvedFields = useMemo(() => {
        if (fields && fields.length > 0) {
            return ['name', ...fields];
        }
        
        if (!metaData?.message?.fields) {
            return ['name', 'modified'];
        }
        
        // Get fields marked for list view
        const listViewFields = metaData.message.fields
            .filter(f => f.in_list_view && !f.hidden)
            .map(f => f.fieldname);
        
        return ['name', ...listViewFields, 'modified'];
    }, [fields, metaData]);

    // Fetch data using SDK
    const { data, isLoading, error, mutate } = useFrappeGetDocList(
        doctype,
        {
            fields: resolvedFields,
            limit: limit,
            orderBy: orderByConfig,
        }
    );

    return {
        data: data || [],
        total: data?.length || 0,
        fields: resolvedFields,
        isLoading,
        error,
        refetch: mutate,
    };
};
```

**Usage Example:**

```typescript
function CustomerList() {
    const { data, isLoading, refetch } = useDoctypeData({
        doctype: 'Customer',
        fields: ['customer_name', 'email', 'phone'],
        limit: 50,
        orderBy: 'modified desc'
    });
    
    if (isLoading) return <div>Loading...</div>;
    
    return (
        <div>
            <button onClick={refetch}>Refresh</button>
            <table>
                <thead>
                    <tr>
                        <th>Name</th>
                        <th>Email</th>
                        <th>Phone</th>
                    </tr>
                </thead>
                <tbody>
                    {data.map(customer => (
                        <tr key={customer.name}>
                            <td>{customer.customer_name}</td>
                            <td>{customer.email}</td>
                            <td>{customer.phone}</td>
                        </tr>
                    ))}
                </tbody>
            </table>
        </div>
    );
}
```

### Direct API Calls

For custom endpoints:

```typescript
import { useFrappeGetCall, useFrappePostCall } from 'frappe-react-sdk';

// GET request
function DashboardStats() {
    const { data, isLoading } = useFrappeGetCall('myapp.api.config.get_app_config');
    
    if (isLoading) return <div>Loading...</div>;
    
    return <div>Socket Port: {data?.socketio_port}</div>;
}

// POST request
function CreateCustomer() {
    const { call, loading } = useFrappePostCall('myapp.api.dynamic_data.create_doctype_record');
    
    const handleCreate = async () => {
        await call({
            doctype_name: 'Customer',
            data: {
                customer_name: 'John Doe',
                email: 'john@example.com'
            }
        });
    };
    
    return <button onClick={handleCreate} disabled={loading}>Create</button>;
}
```

### File Upload

```typescript
import { useFrappeFileUpload } from 'frappe-react-sdk';

function FileUploader() {
    const { upload, loading, error } = useFrappeFileUpload();
    
    const handleFileUpload = async (file: File) => {
        const result = await upload(file, {
            isPrivate: false,
            folder: 'Home/Attachments'
        });
        
        console.log('File uploaded:', result.file_url);
    };
    
    return (
        <input 
            type="file" 
            onChange={(e) => handleFileUpload(e.target.files[0])}
            disabled={loading}
        />
    );
}
```

---

## 13. Build & Deployment Process

### package.json Build Script

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build --base=/assets/myapp/ui_assets/ && node scripts/post-build.js",
    "preview": "vite preview"
  }
}
```

### Build Process Steps

```bash
npm run build
```

**What happens:**

1. **TypeScript Compilation** (`tsc`)
   - Checks for type errors
   - Generates `.d.ts` files if needed

2. **Vite Build** (`vite build --base=/assets/myapp/ui_assets/`)
   - Bundles React application
   - Optimizes assets (minify, tree-shake)
   - Outputs to `../myapp/public/ui_assets/`
   - All asset paths use `/assets/myapp/ui_assets/` prefix

3. **Post-Build Script** (`node scripts/post-build.js`)
   - Reads `myapp/public/ui_assets/index.html`
   - Injects Jinja templates for CSRF and boot data
   - Copies to `myapp/www/myapp.html`

### Output Structure

```
myapp/
├── public/
│   └── ui_assets/                    # Built React app
│       ├── assets/
│       │   ├── index-AbC123.js      # Main JS bundle
│       │   └── index-XyZ789.css     # CSS bundle
│       └── index.html               # Original build output
│
└── www/
    └── myapp.html                   # Jinja template (for Frappe)
```

### How Frappe Serves Your App

```
User Request:  http://mysite.localhost:8000/myapp

1. Frappe checks hooks.py for route mapping
   website_route_rules: /myapp → myapp

2. Frappe looks for www/myapp.py
   Executes get_context() function

3. Renders www/myapp.html with context data
   Injects CSRF token and boot data

4. Browser loads HTML and assets
   Assets served from: /assets/myapp/ui_assets/

5. React app boots and takes over routing
```

### Deploy to Production Server

```bash
# On production server
cd /path/to/frappe-bench

# Pull latest code
cd apps/myapp
git pull origin main

# Build React app
cd dashboard
npm install
npm run build

# Restart Frappe
cd /path/to/frappe-bench
bench --site mysite.localhost clear-cache
bench restart
```

### Automated Deployment

Create a deploy script:

```bash
#!/bin/bash
# deploy.sh

set -e

echo "🚀 Starting deployment..."

# Build frontend
echo "📦 Building React app..."
cd dashboard
npm install
npm run build
cd ..

# Clear cache
echo "🧹 Clearing cache..."
bench --site mysite.localhost clear-cache

# Restart
echo "🔄 Restarting services..."
bench restart

echo "✅ Deployment complete!"
```

---

## 14. Development Workflow

### Starting Development

**Terminal 1: Start Frappe Backend**

```bash
cd /path/to/frappe-bench
bench start
```

This starts:
- MariaDB (database)
- Redis (cache & realtime)
- Frappe web server (port 8000)
- SocketIO server (port 9000)
- Background workers

**Terminal 2: Start React Dev Server**

```bash
cd /path/to/frappe-bench/apps/myapp/dashboard
npm run dev
```

This starts:
- Vite dev server (port 3000)
- Hot Module Replacement (HMR)
- Development proxy to Frappe

**Access Your App:**

```
Development:  http://localhost:3000
Production:   http://mysite.localhost:8000/myapp
```

### Hot Reload Behavior

| Change Type | Reload Behavior |
|-------------|----------------|
| React component | ✅ Instant (HMR) |
| TypeScript file | ✅ Instant (HMR) |
| CSS/Tailwind | ✅ Instant (HMR) |
| Python API file | ❌ Requires bench restart |
| hooks.py | ❌ Requires bench restart |
| www/*.py | ❌ Requires bench restart |
| DocType changes | ❌ Requires bench migrate |

### Common Development Tasks

**Making API Changes:**

1. Edit Python file (`myapp/api/config.py`)
2. Restart Frappe: `bench restart` (or just restart web process)
3. Frontend will pick up changes automatically (via proxy)

**Updating DocType:**

```bash
# After changing DocType JSON
bench --site mysite.localhost migrate
```

**Clearing Cache:**

```bash
# Clear Redis cache
bench --site mysite.localhost clear-cache

# Clear specific doctype cache
bench --site mysite.localhost clear-cache --doctype "Customer"
```

**Debugging:**

```python
# Python backend
frappe.log_error(f"Debug: {variable}", "My App Debug")

# View logs
bench --site mysite.localhost console
```

```typescript
// React frontend
console.log('Debug:', variable);

// View in browser console (F12)
```

---

## 15. Common Patterns & Best Practices

### Router Setup

```typescript
// src/providers/index.tsx
import { BrowserRouter } from 'react-router-dom';

export default function AppProvider({ children }: { children: React.ReactNode }) {
    // ⭐ CRITICAL: basename must match Frappe route
    const basename = '/myapp';

    return (
        <BrowserRouter basename={basename}>
            <ErrorBoundary>
                {children}
            </ErrorBoundary>
        </BrowserRouter>
    );
}
```

**Why basename is required:**
- React Router thinks all routes are relative to `/`
- But Frappe serves your app at `/myapp`
- Without basename, navigation breaks

### Error Handling

**React Error Boundary:**

```typescript
import { ErrorBoundary, FallbackProps } from 'react-error-boundary';

const ErrorFallback = ({ error }: FallbackProps) => {
    return (
        <div className="error-page">
            <h2>Something went wrong</h2>
            <pre>{error.message}</pre>
            <button onClick={() => window.location.reload()}>
                Reload Page
            </button>
        </div>
    );
};

function App() {
    return (
        <ErrorBoundary FallbackComponent={ErrorFallback}>
            <YourApp />
        </ErrorBoundary>
    );
}
```

**API Error Handling:**

```typescript
const { data, error } = useFrappeGetCall('myapp.api.get_data');

if (error) {
    return <div className="error">Error: {error.message}</div>;
}

if (!data) {
    return <div>Loading...</div>;
}

return <div>Data: {JSON.stringify(data)}</div>;
```

### Caching Strategy

**SWR Configuration:**

```typescript
<FrappeProvider
    swrConfig={{
        // Retry failed requests
        errorRetryCount: 2,
        errorRetryInterval: 1000,
        
        // Revalidation
        revalidateOnFocus: true,
        revalidateOnReconnect: true,
        
        // Cache provider (localStorage)
        provider: localStorageProvider,
        
        // Deduplication interval (1 second)
        dedupingInterval: 1000,
    }}
>
```

**Per-Query Caching:**

```typescript
// Cache for 5 minutes
const { data } = useFrappeGetCall('api.method', params, {
    dedupingInterval: 300000,
    revalidateOnFocus: false,
});

// Never cache (always fresh)
const { data } = useFrappeGetCall('api.method', params, {
    dedupingInterval: 0,
});
```

### Loading States

```typescript
function CustomerList() {
    const { data, isLoading, error } = useDoctypeData({
        doctype: 'Customer',
        limit: 20
    });
    
    if (isLoading) {
        return (
            <div className="flex items-center justify-center p-8">
                <Spinner />
                <span className="ml-2">Loading customers...</span>
            </div>
        );
    }
    
    if (error) {
        return (
            <div className="text-red-500 p-4">
                <AlertCircle className="inline mr-2" />
                Failed to load customers: {error.message}
            </div>
        );
    }
    
    if (!data || data.length === 0) {
        return (
            <div className="text-gray-500 p-8 text-center">
                <FileText className="mx-auto mb-4" size={48} />
                <p>No customers found</p>
                <Button onClick={() => navigate('/customers/new')}>
                    Add First Customer
                </Button>
            </div>
        );
    }
    
    return (
        <div>
            {data.map(customer => (
                <CustomerCard key={customer.name} customer={customer} />
            ))}
        </div>
    );
}
```

### Form Handling

```typescript
import { useForm } from 'react-hook-form';

function CustomerForm() {
    const { register, handleSubmit, formState: { errors } } = useForm();
    const { createRecord, loading } = useDoctypeCrud({
        doctype: 'Customer',
        onSuccess: () => navigate('/customers')
    });
    
    const onSubmit = async (data) => {
        await createRecord(data);
    };
    
    return (
        <form onSubmit={handleSubmit(onSubmit)}>
            <input 
                {...register('customer_name', { required: 'Name is required' })}
                placeholder="Customer Name"
            />
            {errors.customer_name && (
                <span className="error">{errors.customer_name.message}</span>
            )}
            
            <input 
                {...register('email', { 
                    required: 'Email is required',
                    pattern: {
                        value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
                        message: 'Invalid email address'
                    }
                })}
                type="email"
                placeholder="Email"
            />
            {errors.email && (
                <span className="error">{errors.email.message}</span>
            )}
            
            <button type="submit" disabled={loading}>
                {loading ? 'Creating...' : 'Create Customer'}
            </button>
        </form>
    );
}
```

---

## 16. Troubleshooting Guide

### CSRF Token Issues

**Problem:** API calls fail with "Invalid CSRF Token" error

**Solutions:**

1. **Check post-build script ran:**
   ```bash
   # Verify myapp.html has token injection
   grep "csrf_token" myapp/www/myapp.html
   
   # Should see:
   # window.csrf_token = '{{ csrf_token }}';
   ```

2. **Verify db.commit() in get_context:**
   ```python
   # myapp/www/myapp.py
   csrf_token = frappe.sessions.get_csrf_token()
   frappe.db.commit()  # ⭐ MUST commit!
   ```

3. **Check browser console:**
   ```javascript
   console.log(window.csrf_token);
   // Should print a long token string
   ```

4. **Clear browser cache:**
   - Hard refresh: Ctrl+Shift+R (Chrome/Firefox)
   - Or: Clear site data in DevTools

### Boot Data Problems

**Problem:** `window.frappe.boot` is undefined or missing data

**Solutions:**

1. **Development mode:**
   ```bash
   # Ensure developer_mode is enabled
   bench --site mysite.localhost set-config developer_mode 1
   
   # Check API endpoint works
   curl -X POST http://localhost:8000/api/method/myapp.www.myapp.get_context_for_dev
   ```

2. **Production mode:**
   ```bash
   # Rebuild and verify HTML
   npm run build
   
   # Check myapp.html has boot data
   grep "frappe.boot" myapp/www/myapp.html
   ```

3. **Check for script tag issues:**
   ```python
   # Ensure XSS sanitization doesn't break JSON
   boot_json = SCRIPT_TAG_PATTERN.sub("", boot_json)
   boot_json = CLOSING_SCRIPT_TAG_PATTERN.sub("", boot_json)
   ```

### Proxy Issues

**Problem:** API calls fail in development with connection refused

**Solutions:**

1. **Verify Frappe is running:**
   ```bash
   curl http://localhost:8000/api/method/ping
   # Should return: {"message":"pong"}
   ```

2. **Check proxyOptions.js:**
   ```bash
   # Verify port matches Frappe
   cat dashboard/proxyOptions.js | grep webserver_port
   ```

3. **Check common_site_config.json:**
   ```bash
   cat sites/common_site_config.json
   # Should have: "webserver_port": 8000
   ```

4. **Test proxy manually:**
   ```bash
   cd dashboard
   npm run dev
   
   # In another terminal
   curl http://localhost:3000/api/method/ping
   ```

### Build Issues

**Problem:** Build fails or assets not found

**Solutions:**

1. **Check base path:**
   ```json
   // package.json
   "build": "vite build --base=/assets/myapp/ui_assets/"
   ```

2. **Verify output directory:**
   ```bash
   ls myapp/public/ui_assets/
   # Should have: assets/, index.html
   ```

3. **Check permissions:**
   ```bash
   # Ensure build script can write
   chmod -R 755 myapp/public
   chmod -R 755 myapp/www
   ```

4. **Clear cache and rebuild:**
   ```bash
   cd dashboard
   rm -rf node_modules/.vite
   rm -rf ../myapp/public/ui_assets
   npm run build
   ```

### Route Not Found (404)

**Problem:** Page refresh returns 404 on React routes

**Solutions:**

1. **Verify hooks.py routing:**
   ```python
   website_route_rules = [
       {"from_route": "/myapp/<path:app_path>", "to_route": "myapp"},
       {"from_route": "/myapp", "to_route": "myapp"},
   ]
   ```

2. **Restart Frappe after hooks.py changes:**
   ```bash
   bench restart
   ```

3. **Check basename in React Router:**
   ```typescript
   <BrowserRouter basename="/myapp">
   ```

4. **Verify www/myapp.py exists:**
   ```bash
   ls myapp/www/
   # Should have: myapp.py, myapp.html
   ```

### Permission Denied Errors

**Problem:** API calls fail with "Not Permitted"

**Solutions:**

1. **Check DocType permissions:**
   ```bash
   # In Frappe Desk
   Setup > Permissions > [Your DocType]
   # Grant permissions to your role
   ```

2. **Check user has role:**
   ```python
   # Python console
   frappe.get_roles("user@example.com")
   ```

3. **Whitelist function properly:**
   ```python
   @frappe.whitelist()  # ⭐ Don't forget this!
   def my_function():
       pass
   ```

4. **Check permission in function:**
   ```python
   if not frappe.has_permission("Customer", "read"):
       frappe.throw("Not permitted")
   ```

### WebSocket Connection Failed

**Problem:** Real-time features don't work

**Solutions:**

1. **Check SocketIO port:**
   ```bash
   # common_site_config.json
   "socketio_port": 9000
   ```

2. **Verify socket server running:**
   ```bash
   lsof -i :9000
   # Should show node process
   ```

3. **Check CORS settings:**
   ```python
   # site_config.json
   "socketio_cors_allowed_origins": ["http://localhost:3000"]
   ```

4. **Test WebSocket connection:**
   ```javascript
   // Browser console
   const socket = io('http://localhost:9000');
   socket.on('connect', () => console.log('Connected!'));
   ```

---

## 17. Conclusion & Next Steps

### What We Covered

You've learned how to:

✅ **Set up a complete React + Frappe project structure**
- Vite configuration for Frappe
- Development proxy setup
- Build process configuration

✅ **Handle critical security aspects**
- CSRF token generation and validation
- Boot data injection
- Session management

✅ **Create a robust API architecture**
- Whitelisted Python functions
- Generic CRUD operations
- Permission validation

✅ **Integrate Frappe React SDK effectively**
- Custom hooks for common operations
- SWR caching strategy
- Error handling patterns

✅ **Deploy and troubleshoot**
- Build and deployment process
- Common issues and solutions

### Architecture Recap

```
┌─────────────────────────────────────────┐
│  React SPA (Vite + TypeScript)          │
│                                         │
│  Components → Hooks → Frappe React SDK  │
└──────────────────┬──────────────────────┘
                   │
          HTTP + WebSocket
                   │
┌──────────────────▼──────────────────────┐
│  Frappe Backend (Python)                │
│                                         │
│  hooks.py → www/*.py → api/*.py         │
│                  ↓                      │
│            MariaDB + Redis              │
└─────────────────────────────────────────┘
```

### Key Takeaways

**1. Post-Build Script is Critical**
Without it, CSRF tokens and boot data won't work. Always run after `vite build`.

**2. Routing Must Be Configured in Two Places**
- `hooks.py`: Server-side route mapping
- React Router: Client-side routing with `basename`

**3. Use Frappe React SDK Hooks**
Don't reinvent the wheel. The SDK provides excellent hooks for common operations.

**4. Always Validate Permissions**
Never trust the frontend. Always check permissions in your API functions.

**5. Cache Wisely**
Use SWR caching for performance, but invalidate when needed.

### Next Steps

**Enhance Your App:**

1. **Add Real-Time Features**
   - Implement WebSocket for live updates
   - Use `frappe.publish_realtime()` on backend
   - Listen with Socket.IO on frontend

2. **Implement PWA Features**
   - Add service worker
   - Enable offline mode
   - Add to home screen

3. **Advanced Permissions**
   - Row-level permissions
   - Field-level permissions
   - Custom permission checks

4. **Optimize Performance**
   - Code splitting with React.lazy()
   - Image optimization
   - Bundle analysis

5. **Testing**
   - Unit tests with Jest
   - Integration tests with Playwright
   - E2E testing

### Resources

**Official Documentation:**
- [Frappe Framework Docs](https://frappeframework.com/docs)
- [Frappe React SDK](https://github.com/nikkothari22/frappe-react-sdk)
- [Vite Documentation](https://vitejs.dev/)
- [React Documentation](https://react.dev/)

**Community:**
- [Frappe Forum](https://discuss.frappe.io/)
- [Frappe Discord](https://discord.gg/frappe)
- [GitHub Discussions](https://github.com/frappe/frappe/discussions)

**Example Projects:**
- [Raven](https://github.com/The-Commit-Company/Raven) - Chat application
- [Gameplan](https://github.com/frappe/gameplan) - Project management
- [Your Project Here!]

### Final Thoughts

Building a custom React frontend for Frappe gives you the best of both worlds:
- **Frappe's powerful backend** (DocTypes, permissions, workflow)
- **React's modern frontend** (component architecture, rich ecosystem)

The initial setup requires understanding several moving parts, but once configured, development becomes very productive. The patterns shown in this guide are battle-tested and used in production applications.

**Good luck building amazing applications with Frappe and React!** 🚀

---

## Appendix: Complete File Reference

### Minimal Working Example

Here's a minimal setup to get started:

**1. hooks.py**
```python
app_name = "myapp"

website_route_rules = [
    {"from_route": "/myapp/<path:app_path>", "to_route": "myapp"},
    {"from_route": "/myapp", "to_route": "myapp"},
]
```

**2. www/myapp.py**
```python
import frappe

def get_context(context):
    csrf_token = frappe.sessions.get_csrf_token()
    frappe.db.commit()
    boot = frappe.sessions.get() if frappe.session.user != "Guest" else {}
    
    context.update({
        "csrf_token": csrf_token,
        "boot": json.dumps(frappe.as_json(boot)),
        "app_name": "My App"
    })
    return context
```

**3. vite.config.ts**
```typescript
export default defineConfig({
    build: {
        outDir: "../myapp/public/ui_assets"
    }
});
```

**4. package.json**
```json
{
    "scripts": {
        "build": "vite build --base=/assets/myapp/ui_assets/ && node scripts/post-build.js"
    }
}
```

**5. src/App.tsx**
```typescript
import { FrappeProvider } from 'frappe-react-sdk';

function App() {
    return (
        <FrappeProvider>
            <div>Hello Frappe + React!</div>
        </FrappeProvider>
    );
}
```

Start with this minimal setup, then add features as needed!

---

**Document Version:** 1.0  
**Last Updated:** 2025  
**Author:** Based on the working CohenixUI application  
**License:** MIT


