# Micro-Frontend Architecture with React, TypeScript, Vite & Module Federation

A production-ready micro-frontend setup using **Module Federation** to enable dynamic component sharing between a **Host Application** and a **Remote Application**.

---

## 📋 Project Overview

This project demonstrates a **scalable micro-frontend architecture** where:

- **Host App** (Port 5000): Main application that dynamically imports and renders components
- **Remote App** (Port 5001): Exposes reusable React components for the Host to consume
- **Module Federation**: Enables runtime component sharing without build-time coupling
- **Shared Dependencies**: React and React-DOM are shared to reduce bundle size

---

## 🏗️ Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     HOST APPLICATION                         │
│                      (Port 5000)                             │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  App.tsx                                             │   │
│  │  ├─ RemoteComponentWrapper (Lazy loaded)            │   │
│  │  │  ├─ Remote Header (from remote_app/Header)       │   │
│  │  │  └─ Remote Button (from remote_app/Button)       │   │
│  │  └─ Loading Spinner (Suspense fallback)             │   │
│  └──────────────────────────────────────────────────────┘   │
│                           │                                  │
│         Imports from remoteEntry.js                         │
│                           ▼                                  │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │  Module Federation │  Dynamic Import   │
        │  over HTTP         │  at Runtime       │
        └───────────────────┼───────────────────┘
                            │
        ┌───────────────────▼───────────────────┐
        │   Remote Entry (remoteEntry.js)       │
        │   http://localhost:5001/assets/       │
        │         remoteEntry.js                │
        └───────────────────┬───────────────────┘
                            │
        ┌───────────────────▼───────────────────┐
┌─────────────────────────────────────────────────────────────┐
│                  REMOTE APPLICATION                          │
│                     (Port 5001)                             │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Exposed Components:                                 │   │
│  │  ├─ ./Button   → src/components/Button              │   │
│  │  └─ ./Header   → src/components/Header              │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔗 How Host & Remote Are Connected

### Step-by-Step Connection Flow

#### **1. Remote App Exposes Components**

Remote app's `vite.config.ts` declares which components can be shared:

```typescript
federation({
  name: "remote_app",
  filename: "remoteEntry.js", // Entry point file
  exposes: {
    "./Button": "./src/components/Button", // Export path
    "./Header": "./src/components/Header", // Export path
  },
  shared: ["react", "react-dom"], // Shared dependencies
});
```

This generates:

- `remoteEntry.js` (manifest file listing all exports)
- Component bundles at build time

#### **2. Host App Declares Remote Location**

Host app's `vite.config.ts` tells where to find the remote:

```typescript
federation({
  name: "host_app",
  remotes: {
    remote_app: "http://localhost:5001/assets/remoteEntry.js",
  }, // ↑ Points to Remote's manifest
  shared: ["react", "react-dom"],
});
```

#### **3. Host App Dynamically Imports Components**

At runtime, `RemoteComponentWrapper.tsx` imports remote components:

```typescript
import React, { Suspense } from "react";

// Dynamic imports - resolved at runtime via remoteEntry.js
const RemoteHeader = React.lazy(() => import("remote_app/Header"));
const RemoteButton = React.lazy(() => import("remote_app/Button"));

// Render with Suspense for loading states
export const RemoteComponentWrapper = () => {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <RemoteHeader />
    </Suspense>
  );
};
```

#### **4. Module Federation Runtime Resolution**

When host app runs:

1. **Build Time**: Host compiles with placeholder for `remote_app` modules
2. **Runtime**:
   - Host loads remoteEntry.js from `http://localhost:5001/assets/remoteEntry.js`
   - Module Federation resolves `remote_app/Header` → actual component URL
   - Component is downloaded and executed in host's context
   - Shared dependencies (React, React-DOM) are used from shared pool

---

## 📁 Project Structure

```
micro-frontend/
├── host-app/                          # Main application
│   ├── src/
│   │   ├── components/
│   │   │   └── RemoteComponentWrapper.tsx    # Imports remote components
│   │   ├── App.tsx                   # Main component
│   │   ├── main.tsx
│   │   └── index.css
│   ├── vite.config.ts                # MF Config - declares remotes
│   ├── tsconfig.json
│   ├── package.json
│   └── index.html
│
├── remote-app/                        # Component library
│   ├── src/
│   │   ├── components/
│   │   │   ├── Button.tsx            # Exposed component
│   │   │   └── Header.tsx            # Exposed component
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   └── index.css
│   ├── vite.config.ts                # MF Config - declares exposes
│   ├── tsconfig.json
│   ├── package.json
│   └── index.html
│
└── README.md                          # This file
```

---

## ⚙️ Configuration Details

### **Host App - vite.config.ts**

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import federation from "@originjs/vite-plugin-federation";

export default defineConfig({
  plugins: [
    react(),
    federation({
      name: "host_app", // Unique name for this MF
      remotes: {
        // Map remote app name to its remoteEntry.js URL
        remote_app: "http://localhost:5001/assets/remoteEntry.js",
      },
      shared: ["react", "react-dom"], // Dependencies shared with remote
    }),
  ],
  build: {
    modulePreload: false,
    target: "esnext",
    minify: false,
    cssCodeSplit: false,
  },
});
```

**Key Points:**

- `name`: Unique identifier for this micro-frontend
- `remotes`: Maps remote names to their entry point URLs
- `shared`: Dependencies that should be shared (avoids duplication)

### **Remote App - vite.config.ts**

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import federation from "@originjs/vite-plugin-federation";

export default defineConfig({
  plugins: [
    react(),
    federation({
      name: "remote_app", // Unique name for this MF
      filename: "remoteEntry.js", // Name of manifest file
      exposes: {
        // Export path → source file mapping
        "./Button": "./src/components/Button",
        "./Header": "./src/components/Header",
      },
      shared: ["react", "react-dom"], // Dependencies to share
    }),
  ],
  build: {
    modulePreload: false,
    target: "esnext",
    minify: false,
    cssCodeSplit: false,
  },
  preview: {
    port: 5001,
    strictPort: true,
    cors: true, // Enable CORS for host-app
  },
});
```

**Key Points:**

- `exposes`: Components that can be imported by other MFs
- `filename`: Path where remoteEntry.js will be generated
- `preview.cors`: Allows host-app to load remote components

---

## 🚀 Getting Started

### Prerequisites

- Node.js 18+
- npm or yarn

### Installation & Setup

```bash
# 1. Install dependencies for both apps
cd host-app && npm install
cd ../remote-app && npm install
cd ..
```

### Running the Applications

**Terminal 1 - Remote App (Component Library)**

```bash
cd remote-app
npm run serve
# Output: Local: http://localhost:5001/
```

**Terminal 2 - Host App (Main Application)**

```bash
cd host-app
npm run dev
# Output: Local: http://localhost:5000/
```

Then open: **http://localhost:5000/**

You should see:

- Host app header
- Remote components (Header & Button) loaded dynamically with loading spinner
- Clicking the button triggers an alert

---

## 📦 Available Scripts

### Host App

```json
{
  "dev": "vite", // Dev server on port 5000
  "build": "tsc -b && vite build", // Build for production
  "preview": "vite preview --port 5000", // Preview build locally
  "serve": "npm run build && npm run preview", // Build + preview
  "lint": "eslint ." // Run linter
}
```

### Remote App

```json
{
  "dev": "vite", // Dev server on port 5001
  "build": "tsc -b && vite build", // Build for production
  "preview": "vite preview --port 5001", // Preview build locally
  "serve": "npm run build && npm run preview", // Build + preview
  "lint": "eslint ." // Run linter
}
```

---

## 🔄 Dependency Sharing Strategy

### Shared Dependencies

Both apps declare the same shared dependencies:

```typescript
shared: ["react", "react-dom"];
```

**Benefits:**

- ✅ Single React instance (avoids state conflicts)
- ✅ Reduced bundle size (no duplication)
- ✅ Automatic version negotiation

**Versioning:**

```json
{
  "dependencies": {
    "react": "^19.2.5",
    "react-dom": "^19.2.5"
  }
}
```

Module Federation ensures:

- Both apps use the highest compatible version
- Types remain consistent across MFs

---

## 📝 Code Examples

### How Host Imports Remote Components

**RemoteComponentWrapper.tsx:**

```typescript
import React, { Suspense } from "react";

// Lazy load from remote - syntax: "remote_name/expose_path"
const RemoteHeader = React.lazy(() => import("remote_app/Header"));
const RemoteButton = React.lazy(() => import("remote_app/Button"));

export const RemoteComponentWrapper = () => {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <RemoteHeader />
      <RemoteButton onClick={() => alert("Clicked!")} />
    </Suspense>
  );
};
```

**Key Points:**

- `import("remote_app/Header")`: Maps to remote's expose path `"./Header"`
- `React.lazy()`: Enables code splitting and dynamic loading
- `<Suspense>`: Shows fallback while component loads over network

### How Remote Exposes Components

**src/components/Button.tsx:**

```typescript
interface ButtonProps {
  text?: string;
  onClick?: () => void;
}

export default function Button({ text = "Click me", onClick }: ButtonProps) {
  return (
    <button onClick={onClick} className="px-4 py-2 bg-blue-500 text-white">
      {text}
    </button>
  );
}
```

**src/components/Header.tsx:**

```typescript
export default function Header() {
  return <h2 className="text-xl font-bold">Remote Header Component</h2>;
}
```

These are automatically exposed via `vite.config.ts` configuration.

---

## 🌐 URLs Reference

| Service            | Port | URL                                         |
| ------------------ | ---- | ------------------------------------------- |
| Host App Dev       | 5000 | http://localhost:5000                       |
| Host App Preview   | 5000 | http://localhost:5000                       |
| Remote App Dev     | 5001 | http://localhost:5001                       |
| Remote Entry Point | 5001 | http://localhost:5001/assets/remoteEntry.js |

---

## 🔍 Troubleshooting

### Issue: "Cannot find module 'remote_app/Button'"

**Solution:** Ensure remote-app is running on port 5001 and built successfully.

### Issue: Components show loading spinner indefinitely

**Solution:** Check browser console for CORS errors. Ensure `preview.cors: true` in remote-app config.

### Issue: Shared dependency version mismatch

**Solution:** Ensure both apps have compatible versions of react/react-dom in package.json.

### Issue: Build fails with TypeScript errors

**Solution:** Run `tsc -b` separately to check for type errors before Vite build.

---

## 📚 Technology Stack

| Technology               | Version | Purpose                        |
| ------------------------ | ------- | ------------------------------ |
| React                    | ^19.2.5 | UI library                     |
| TypeScript               | ~6.0.2  | Type safety                    |
| Vite                     | ^8.0.10 | Build tool & dev server        |
| Module Federation Plugin | ^1.4.1  | Enables micro-frontend sharing |
| Babel                    | ^7.29.0 | JavaScript transpilation       |
| React Compiler           | ^1.0.0  | Optimizes React renders        |

---

## 🎯 Key Benefits of This Architecture

✅ **Independent Deployment**: Each app can be deployed separately  
✅ **Technology Agnostic**: Each MF can use different versions/libraries (if needed)  
✅ **Code Sharing**: Components shared without monorepo constraints  
✅ **Reduced Bundle Size**: Shared dependencies loaded once  
✅ **Scalability**: Easy to add more remote apps  
✅ **Development Experience**: Teams can work independently

---

## 📖 Further Reading

- [Vite Module Federation Plugin](https://github.com/originjs/vite-plugin-federation)
- [Webpack Module Federation](https://webpack.js.org/concepts/module-federation/)
- [React Suspense](https://react.dev/reference/react/Suspense)
- [Micro Frontends](https://micro-frontends.org/)

---

## 📝 Notes

- **Port Configuration**: Host runs on 5000, Remote on 5001. Both must be running for full functionality.
- **CORS**: Remote app has CORS enabled to allow cross-origin requests from host.
- **Hot Reload**: During development, changes to either app auto-refresh in browser.
- **Production Build**: Both apps can be deployed to different servers/CDNs with only URL config changes.

---

## 🔧 TypeScript Configuration

### **tsconfig.json (Both Apps)**

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.app.json" }]
}
```

**Key Options:**

- `jsx: "react-jsx"`: Auto-import React for JSX
- `strict: true`: Enforces type safety
- `moduleResolution: "bundler"`: Vite-compatible resolution

---

## 🎯 Common Module Federation Patterns

### **Pattern 1: Exposing Multiple Components**

```typescript
// vite.config.ts - Remote App
exposes: {
  "./Button": "./src/components/Button",
  "./Header": "./src/components/Header",
  "./Modal": "./src/components/Modal",
  "./Card": "./src/components/Card",
  "./utils": "./src/utils",  // Can also expose utilities
}
```

### **Pattern 2: Host App with Multiple Remotes**

```typescript
// vite.config.ts - Host App
remotes: {
  remote_app: "http://localhost:5001/assets/remoteEntry.js",
  auth_app: "http://localhost:5002/assets/remoteEntry.js",
  dashboard_app: "http://localhost:5003/assets/remoteEntry.js",
}
```

### **Pattern 3: Bidirectional Sharing (Host & Remote expose)**

```typescript
// Remote app can also consume from Host
federation({
  name: "remote_app",
  exposes: { "./Button": "./src/components/Button" },
  remotes: {
    host_app: "http://localhost:5000/assets/remoteEntry.js",
  },
});
```

---

## 💡 Best Practices

### **1. Component Design**

```typescript
// ✅ GOOD: Self-contained, prop-based
export default function Button({
  onClick,
  text,
  variant = 'primary'
}: ButtonProps) {
  return <button>{text}</button>;
}

// ❌ BAD: Depends on external state
export default function Button() {
  const state = useGlobalStore();  // Tightly coupled
  return <button>{state.text}</button>;
}
```

### **2. Error Boundaries**

```typescript
// Always wrap remote components with error boundary
import { ErrorBoundary } from 'react-error-boundary';

<ErrorBoundary fallback={<div>Component failed to load</div>}>
  <Suspense fallback={<LoadingSpinner />}>
    <RemoteComponent />
  </Suspense>
</ErrorBoundary>
```

### **3. Version Pinning**

```json
{
  "dependencies": {
    "react": "19.2.5", // Exact version
    "react-dom": "19.2.5"
  }
}
```

### **4. Minimize Shared Dependencies**

```typescript
// ✅ Share only critical dependencies
shared: ["react", "react-dom"];

// ❌ Avoid over-sharing
shared: ["react", "react-dom", "lodash", "axios", "date-fns"];
```

### **5. Naming Conventions**

```typescript
// Use underscores for MF names (avoid hyphens)
name: "remote_app"; // ✅ Correct
name: "remote-app"; // ❌ Requires quotes
```

---

## ⚡ Performance Optimization

### **1. Code Splitting**

```typescript
// Remote app - split by feature
const Modal = lazy(() => import("./components/Modal"));
const Form = lazy(() => import("./components/Form"));
```

### **2. Lazy Loading with Prefetch**

```typescript
// Preload remoteEntry.js at idle time
if ("requestIdleCallback" in window) {
  requestIdleCallback(() => {
    fetch("http://localhost:5001/assets/remoteEntry.js");
  });
}
```

### **3. Bundle Analysis**

```bash
# Install rollup visualizer
npm install --save-dev rollup-plugin-visualizer

# Then add to vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [visualizer({ open: true })]
});
```

### **4. Reduce Shared Dependencies Size**

```typescript
// Instead of sharing entire lodash
shared: ["react", "react-dom"];

// Import only needed utilities
import { debounce } from "lodash-es";
```

---

## 🚀 Scaling to Multiple Remote Apps

### **Example: Adding Auth App**

**auth-app/vite.config.ts:**

```typescript
federation({
  name: "auth_app",
  filename: "remoteEntry.js",
  exposes: {
    "./LoginForm": "./src/components/LoginForm",
    "./ProtectedRoute": "./src/components/ProtectedRoute",
  },
  shared: ["react", "react-dom"],
});
```

**host-app/vite.config.ts:**

```typescript
federation({
  name: "host_app",
  remotes: {
    remote_app: "http://localhost:5001/assets/remoteEntry.js",
    auth_app: "http://localhost:5002/assets/remoteEntry.js",
  },
  shared: ["react", "react-dom"],
});
```

**host-app/src/App.tsx:**

```typescript
import { lazy } from 'react';

const LoginForm = lazy(() => import('auth_app/LoginForm'));
const Header = lazy(() => import('remote_app/Header'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <LoginForm />
      <Header />
    </Suspense>
  );
}
```

---

## 🧪 Testing Strategies

### **1. Unit Testing Remote Components**

```typescript
// remote-app/src/components/__tests__/Button.test.tsx
import { render, screen } from '@testing-library/react';
import Button from '../Button';

describe('Button Component', () => {
  it('renders correctly', () => {
    render(<Button text="Click me" />);
    expect(screen.getByText('Click me')).toBeInTheDocument();
  });

  it('calls onClick handler', () => {
    const onClick = vi.fn();
    render(<Button onClick={onClick} />);
    screen.getByRole('button').click();
    expect(onClick).toHaveBeenCalled();
  });
});
```

### **2. Integration Testing with Host**

```typescript
// host-app/__tests__/RemoteComponentIntegration.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import { RemoteComponentWrapper } from '../components/RemoteComponentWrapper';

describe('Remote Component Integration', () => {
  it('loads remote components', async () => {
    render(<RemoteComponentWrapper />);
    await waitFor(() => {
      expect(screen.getByText(/Remote/)).toBeInTheDocument();
    });
  });
});
```

### **3. E2E Testing**

```typescript
// e2e/remote-integration.spec.ts (Playwright/Cypress)
test("should load and interact with remote components", async ({ page }) => {
  await page.goto("http://localhost:5000");
  await page.waitForSelector("button");
  await page.click("button");
  await expect(page).toHaveTitle("Host Application");
});
```

---

## 📦 Deployment Guide

### **Production Build Steps**

```bash
# 1. Build remote app first
cd remote-app
npm run build
# Output: dist/ folder with remoteEntry.js

# 2. Build host app
cd ../host-app
npm run build
# Output: dist/ folder referencing remote via URL
```

### **Deployment Scenarios**

#### **Scenario 1: Same Domain**

```typescript
// vite.config.ts - Host App
remotes: {
  remote_app: "https://myapp.com/remote/assets/remoteEntry.js",
}
```

#### **Scenario 2: Different Domains**

```typescript
// vite.config.ts - Host App
remotes: {
  remote_app: "https://remote.example.com/assets/remoteEntry.js",
  auth_app: "https://auth.example.com/assets/remoteEntry.js",
}

// Ensure CORS headers on remote servers
// Access-Control-Allow-Origin: https://host.example.com
```

#### **Scenario 3: CDN Deployment**

```typescript
// vite.config.ts - Remote App
remotes: {
  remote_app: "https://cdn.example.com/v1.0.0/remoteEntry.js",
}
```

### **Environment Variables**

```typescript
// vite.config.ts
federation({
  remotes: {
    remote_app:
      import.meta.env.VITE_REMOTE_URL ||
      "http://localhost:5001/assets/remoteEntry.js",
  },
});
```

**.env.production**

```
VITE_REMOTE_URL=https://myapp.com/remote/assets/remoteEntry.js
```

---

## 🔍 Debugging & DevTools

### **1. Check RemoteEntry.js in Browser**

```javascript
// Open DevTools Console on host app
fetch("http://localhost:5001/assets/remoteEntry.js")
  .then((r) => r.text())
  .then(console.log);
```

### **2. Inspect Module Federation Globals**

```javascript
// In browser console
console.log(__webpack_share_scopes__); // Shared modules
console.log(__webpack_require__); // Module system
```

### **3. Check Network Tab**

- Open DevTools → Network tab
- Look for `remoteEntry.js` download
- Verify component chunk downloads
- Check for 404 or CORS errors

### **4. TypeScript Intellisense**

```typescript
// Create type definitions for remotes
// host-app/types/remotes.d.ts

declare module "remote_app/Header" {
  import { FC } from "react";
  const Header: FC;
  export default Header;
}

declare module "remote_app/Button" {
  import { FC } from "react";
  interface ButtonProps {
    text?: string;
    onClick?: () => void;
  }
  const Button: FC<ButtonProps>;
  export default Button;
}
```

---

## ⚠️ Common Gotchas & Solutions

### **Gotcha 1: "Shared Module Not Available"**

```
Error: Shared module is not available for eager consumption
```

**Solution:** Use `eager: true` if shared module is needed immediately

```typescript
shared: {
  react: { singleton: true, eager: true },
  'react-dom': { singleton: true, eager: true }
}
```

### **Gotcha 2: "Cannot Find Module" in TypeScript**

```
TS2307: Cannot find module 'remote_app/Button'
```

**Solution:** Create type declarations or use `skip-types` in tsconfig

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "skipLibCheck": true,
    "allowJs": true
  }
}
```

### **Gotcha 3: Different React Instances**

```
Error: Invalid hook call. Hooks can only be called inside the body of a function component
```

**Solution:** Ensure React is marked as `singleton` in shared

```typescript
shared: {
  react: { singleton: true, requiredVersion: '^19.2.5' },
  'react-dom': { singleton: true, requiredVersion: '^19.2.5' }
}
```

### **Gotcha 4: CORS Errors in Production**

```
Access to XMLHttpRequest blocked by CORS policy
```

**Solution:** Configure CORS headers on remote server

```nginx
# nginx.conf
add_header Access-Control-Allow-Origin "https://host.example.com";
add_header Access-Control-Allow-Methods "GET, POST, OPTIONS";
```

### **Gotcha 5: CSS Conflicts Between MFs**

**Solution:** Use CSS Modules or scoped styles

```typescript
// Button.module.css
.button {
  padding: 8px 16px;
  background: blue;
}

// Button.tsx
import styles from './Button.module.css';

export default function Button() {
  return <button className={styles.button}>Click</button>;
}
```

---

## 🌍 Environment Setup

### **Required Versions**

```bash
node --version     # v18.0.0 or higher
npm --version      # v9.0.0 or higher
```

### **Installation Verification**

```bash
# Check Node.js
node -v            # Should be v18+

# Check npm
npm -v             # Should be v9+

# Verify module federation support
npm ls @originjs/vite-plugin-federation
```

### **Initial Setup Checklist**

- [ ] Node.js 18+ installed
- [ ] npm 9+ installed
- [ ] Both apps have same React version
- [ ] Ports 5000 and 5001 are available
- [ ] CORS enabled in remote app
- [ ] Both vite.config.ts files configured correctly

---

## 📊 Quick Reference: File-to-File Connection

| Host File                  | Purpose                | Connects To            | Remote File            |
| -------------------------- | ---------------------- | ---------------------- | ---------------------- |
| vite.config.ts             | Declares remotes       | remoteEntry.js         | vite.config.ts         |
| App.tsx                    | Uses remote components | RemoteComponentWrapper | Button.tsx, Header.tsx |
| RemoteComponentWrapper.tsx | Lazy loads remotes     | import()               | Button.tsx, Header.tsx |
| package.json               | Declares dependencies  | React version          | package.json           |

---

## 🎓 Learning Resources

### **Articles**

- [Micro Frontends Architecture](https://micro-frontends.org/)
- [Module Federation Best Practices](https://webpack.js.org/concepts/module-federation/)
- [React with Module Federation](https://dev.to/marais/getting-started-with-module-federation-in-react-c1e)

### **Video Tutorials**

- Webpack Module Federation concepts
- Vite with Module Federation setup
- Micro-frontend scaling strategies

### **Official Docs**

- [Vite Documentation](https://vitejs.dev/)
- [@originjs/vite-plugin-federation](https://github.com/originjs/vite-plugin-federation)
- [React Documentation](https://react.dev/)

---

## 📞 Support & Contribution

### **Getting Help**

1. Check Troubleshooting section above
2. Review browser console for errors
3. Check Network tab in DevTools
4. Verify both apps are running on correct ports

### **Contributing**

To add features or fix issues:

1. Create feature branch
2. Test with both host and remote apps
3. Verify all npm scripts pass
4. Submit PR with clear description

---

## 📄 License & Credits

This project demonstrates Module Federation concepts using:

- **Vite**: Fast build tool
- **React**: UI library
- **TypeScript**: Type safety
- **@originjs/vite-plugin-federation**: Module Federation for Vite

---

**Happy coding! 🚀**
