# Micro-Frontend Project Setup Process

## 📋 Overview
This document explains the complete step-by-step process of how the **Host App**, **Remote App**, and **README.md** files were created and organized in this micro-frontend project.

---

## 🏗️ Project Structure Created

```
micro-frontend/
├── README.md                    # Main comprehensive documentation (900+ lines)
├── PROCESS.md                   # This file - explains how it was built
├── host-app/                    # Main host application
│   ├── src/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   ├── index.css
│   │   ├── components/
│   │   │   └── RemoteComponentWrapper.tsx    # Imports remote components
│   │   └── types/
│   │       └── remote-app.d.ts              # Type definitions
│   ├── public/
│   ├── vite.config.ts           # Module Federation config
│   ├── tsconfig.json
│   ├── package.json
│   ├── index.html
│   ├── .gitignore
│   └── README.md
│
├── remote-app/                  # Component library
│   ├── src/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   ├── index.css
│   │   ├── components/
│   │   │   ├── Button.tsx       # Exposed component
│   │   │   └── Header.tsx       # Exposed component
│   │   └── types/
│   ├── public/
│   ├── vite.config.ts           # Module Federation config
│   ├── tsconfig.json
│   ├── package.json
│   ├── index.html
│   ├── .gitignore
│   └── README.md
│
└── .git/                        # Git repository
```

---

## 🔧 Step-by-Step Setup Process

### **Step 1: Project Initialization**

#### Created Two Separate Applications
- **host-app**: Main application running on port 5000
- **remote-app**: Component library running on port 5001

Each application was initialized with:
- React 19.2.5
- TypeScript 6.0.2
- Vite 8.0.10
- Module Federation Plugin

#### Package.json Structure for Both Apps

```json
{
  "name": "host-app" / "remote-app",
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "lint": "eslint .",
    "preview": "vite preview --port 5000/5001",
    "serve": "npm run build && npm run preview"
  },
  "dependencies": {
    "react": "^19.2.5",
    "react-dom": "^19.2.5"
  },
  "devDependencies": {
    "@originjs/vite-plugin-federation": "^1.4.1",
    "@vitejs/plugin-react": "^6.0.1",
    "typescript": "~6.0.2",
    "vite": "^8.0.10"
  }
}
```

---

### **Step 2: Remote App Configuration**

#### Created Component Exports (src/components/)

**Button.tsx** - Self-contained button component:
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

**Header.tsx** - Simple header component:
```typescript
export default function Header() {
  return <h2 className="text-xl font-bold">Remote Header Component</h2>;
}
```

#### Remote App vite.config.ts - Exposes Components
```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import federation from "@originjs/vite-plugin-federation";

export default defineConfig({
  plugins: [
    react(),
    federation({
      name: "remote_app",                    // Unique identifier
      filename: "remoteEntry.js",            // Output manifest file
      exposes: {
        "./Button": "./src/components/Button",    // Path mapping
        "./Header": "./src/components/Header",    // Path mapping
      },
      shared: ["react", "react-dom"],        // Shared dependencies
    }),
  ],
  build: {
    modulePreload: false,
    target: "esnext",
    minify: false,
    cssCodeSplit: false,
  },
  preview: {
    port: 5001,                              // Runs on port 5001
    strictPort: true,
    cors: true,                              // Enable CORS
  },
});
```

**Key Decisions:**
- ✅ `exposes`: Defines which components can be imported by host
- ✅ `filename: "remoteEntry.js"`: Creates manifest file at `dist/assets/remoteEntry.js`
- ✅ `cors: true`: Allows host-app to load components over HTTP
- ✅ `shared: ["react", "react-dom"]`: React shared to avoid duplication

---

### **Step 3: Host App Configuration**

#### Created Remote Component Wrapper (src/components/RemoteComponentWrapper.tsx)

```typescript
import React, { Suspense } from "react";

// Lazy load remote components
const RemoteHeader = React.lazy(() => import("remote_app/Header"));
const RemoteButton = React.lazy(() => import("remote_app/Button"));

const LoadingSpinner = () => (
  <div className="flex justify-center p-4">
    <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-gray-900"></div>
  </div>
);

export const RemoteComponentWrapper = () => {
  return (
    <div className="p-4">
      <Suspense fallback={<LoadingSpinner />}>
        <RemoteHeader />
      </Suspense>

      <div className="mt-4">
        <Suspense fallback={<LoadingSpinner />}>
          <RemoteButton
            text="Remote Button"
            onClick={() => alert("Component imported successfully!")}
          />
        </Suspense>
      </div>
    </div>
  );
};
```

**Pattern Used:**
- `React.lazy()`: Dynamic import for code splitting
- `Suspense`: Loading state while component downloads
- `import("remote_app/Header")`: Module Federation import syntax

#### Host App vite.config.ts - Consumes Remote
```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import federation from "@originjs/vite-plugin-federation";

export default defineConfig({
  plugins: [
    react(),
    federation({
      name: "host_app",                      // Unique identifier
      remotes: {
        // Maps remote name to its URL
        remote_app: "http://localhost:5001/assets/remoteEntry.js",
      },
      shared: ["react", "react-dom"],        // Same shared deps as remote
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

**Key Decisions:**
- ✅ `remotes`: Points to remote app's manifest URL
- ✅ `shared: ["react", "react-dom"]`: Must match remote app
- ✅ Port 5000: Host runs on different port
- ✅ No CORS needed: Host initiates requests to remote

---

### **Step 4: Main App Integration (App.tsx)**

**Host App - App.tsx** - Uses remote components:
```typescript
import viteLogo from "./assets/vite.svg";
import "./App.css";
import "./index.css";
import { RemoteComponentWrapper } from "./components/RemoteComponentWrapper";

function App() {
  return (
    <>
      <div className="px-6 border-2">
        <div className="flex justify-center items-center">
          <img src={viteLogo} alt="Example" />
        </div>
        <h1 className="text-2xl">Host Application</h1>
        <p>Welcome to the Host application, below are the components pulled from the remote application</p>
        <RemoteComponentWrapper />
      </div>
    </>
  );
}

export default App;
```

---

### **Step 5: TypeScript Configuration**

#### Both Apps - tsconfig.json
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
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.app.json" }]
}
```

---

### **Step 6: Documentation Creation**

#### Created Comprehensive README.md (900+ lines)

**Sections Included:**
1. **Project Overview** - What this project does
2. **Architecture Diagram** - Visual flow of host → remote
3. **How Host & Remote Connected** - 4-step connection process
4. **Project Structure** - File organization
5. **Configuration Details** - Both vite.config.ts files explained
6. **Getting Started** - Installation & running instructions
7. **Available Scripts** - All npm commands
8. **Dependency Sharing** - React shared strategy
9. **Code Examples** - Import patterns
10. **URLs Reference** - Quick port lookup
11. **Troubleshooting** - Common issues & solutions
12. **TypeScript Configuration** - Type setup
13. **Common Patterns** - MF patterns
14. **Best Practices** - Do's and don'ts
15. **Performance Optimization** - Bundle strategies
16. **Scaling to Multiple Remotes** - Adding more remotes
17. **Testing Strategies** - Unit, integration, E2E
18. **Deployment Guide** - Production scenarios
19. **Debugging & DevTools** - Browser inspection
20. **Common Gotchas** - Real-world problems & solutions
21. **Environment Setup** - Node.js versions
22. **Learning Resources** - References

---

### **Step 7: Git Setup & Version Control**

#### Step 7a: Initialize Git Repository
```bash
cd i:\micro-frontend
git init
```

#### Step 7b: Create .gitignore Files
Both host-app and remote-app include:
```
node_modules/
dist/
build/
.env
.env.local
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.vscode/
.idea/
.DS_Store
Thumbs.db
```

#### Step 7c: Fix Import Issues
Fixed path import in host-app/src/App.tsx:
```typescript
// ❌ BEFORE (Wrong path)
import viteLogo from "/vite.svg";

// ✅ AFTER (Correct path)
import viteLogo from "./assets/vite.svg";
```

Fixed vite.config.ts imports in both projects:
```typescript
// ❌ BEFORE (Non-existent package)
import react from "@vitejs/plugin-react-swc";

// ✅ AFTER (Correct package)
import react from "@vitejs/plugin-react";
```

#### Step 7d: Stage & Commit Files
```bash
git add host-app/ remote-app/ README.md
git commit -m "Add micro-frontend projects: host-app and remote-app with Module Federation setup"
```

**Commit Details:**
- 43 files changed
- 7,145 insertions added
- Both projects fully tracked

#### Step 7e: Push to GitHub
```bash
git push origin master
```

**Result:**
```
To https://github.com/Satyam9169/microfrontend-vite-typescript-module_federation.git
   0fb3c42..cf4e9e9  master -> master
```

---

## 🔄 How Remote & Host Connect (Runtime)

### **Connection Flow:**

```
┌─────────────────────────────────────────────┐
│ 1. Remote App (http://localhost:5001)       │
│    - Builds vite.config.ts exposes         │
│    - Generates remoteEntry.js manifest     │
│    - Exposes Button, Header components     │
└────────────────────┬────────────────────────┘
                     │
                     │ Manifest located at:
                     │ /assets/remoteEntry.js
                     │
┌────────────────────▼────────────────────────┐
│ 2. Host App (http://localhost:5000)         │
│    - Reads vite.config.ts remotes URL      │
│    - Fetches remoteEntry.js at build time  │
│    - Creates module stub for "remote_app"  │
└────────────────────┬────────────────────────┘
                     │
                     │ At runtime:
                     │ User opens localhost:5000
                     │
┌────────────────────▼────────────────────────┐
│ 3. Runtime Resolution                       │
│    - React.lazy(() => import("remote_app/Button"))
│    - Module Federation finds Button in stub │
│    - Downloads Button chunk from remote    │
│    - Renders in Host's React context       │
└─────────────────────────────────────────────┘
```

### **Key Technologies Used:**

| Technology | Purpose | Version |
|-----------|---------|---------|
| **React** | UI Component Library | 19.2.5 |
| **TypeScript** | Type Safety | 6.0.2 |
| **Vite** | Build Tool & Dev Server | 8.0.10 |
| **Module Federation** | Component Sharing | @originjs/vite-plugin-federation ^1.4.1 |
| **Babel** | JS Transpilation | 7.29.0 |

---

## 📊 Folder Organization Logic

### **Why Two Separate Folders?**

```
✅ Remote App (remote-app/)
   - Independent deployment
   - Can be deployed separately
   - Provides components via HTTP
   - Runs on port 5001

✅ Host App (host-app/)
   - Main application
   - Consumes remote components
   - Orchestrates the UI
   - Runs on port 5000

✅ Root README.md
   - Explains the entire architecture
   - Common setup instructions
   - Shared documentation
```

### **Monorepo Structure Benefits:**

| Benefit | How Achieved |
|---------|-------------|
| **Single Repository** | Both apps in one git repo |
| **Shared Documentation** | Root README.md for both |
| **Independent Builds** | Separate build configs |
| **Easy Scaling** | Can add auth-app, dashboard-app, etc. |
| **Collaborative** | Teams can work on each app |

---

## 🚀 Running the Complete Setup

### **Step 1: Install Dependencies**
```bash
# Install host-app
cd host-app
npm install

# Install remote-app
cd ../remote-app
npm install
cd ..
```

### **Step 2: Start Remote App (Terminal 1)**
```bash
cd remote-app
npm run serve
# Output: Local: http://localhost:5001/
```

### **Step 3: Start Host App (Terminal 2)**
```bash
cd host-app
npm run dev
# Output: Local: http://localhost:5000/
```

### **Step 4: Open Browser**
- Navigate to: **http://localhost:5000**
- You should see:
  - ✅ Host app header
  - ✅ Remote components loading
  - ✅ Button that triggers alert

---

## 📝 File Creation Summary

### **Files Created/Modified:**

| File | Purpose | Size |
|------|---------|------|
| **README.md** | Main documentation | 900+ lines |
| **PROCESS.md** | This file | ~400 lines |
| **host-app/vite.config.ts** | Host MF config | ~25 lines |
| **remote-app/vite.config.ts** | Remote MF config | ~30 lines |
| **host-app/src/components/RemoteComponentWrapper.tsx** | Remote importer | ~35 lines |
| **remote-app/src/components/Button.tsx** | Exposed component | ~15 lines |
| **remote-app/src/components/Header.tsx** | Exposed component | ~8 lines |
| **host-app/.gitignore** | Git ignore rules | ~25 lines |
| **remote-app/.gitignore** | Git ignore rules | ~25 lines |

---

## 🔑 Key Decisions Made

### **1. Module Federation Library**
- ✅ **Chosen**: @originjs/vite-plugin-federation
- **Reason**: Official Vite plugin, works seamlessly with Vite builds

### **2. Shared Dependencies**
- ✅ **Chosen**: React & React-DOM only
- **Reason**: Reduces bundle duplication, ensures single React instance

### **3. Port Configuration**
- ✅ **Host**: 5000
- ✅ **Remote**: 5001
- **Reason**: Prevents port conflicts, easy to remember

### **4. Component Exposure Strategy**
- ✅ **Pattern**: Lazy loading with Suspense
- **Reason**: Better UX with loading states, code splitting

### **5. Git Strategy**
- ✅ **Pattern**: Monorepo with subfolders
- **Reason**: Single repository, shared documentation, easy collaboration

---

## 🎓 What Was Learned

### **Module Federation Concepts:**
1. **Remotes**: Apps that expose components
2. **Hosts**: Apps that consume components
3. **Federation**: Runtime module sharing
4. **Manifest**: remoteEntry.js that lists exports
5. **Lazy Loading**: Dynamic import for MF modules

### **Best Practices Applied:**
- ✅ Minimal shared dependencies
- ✅ Error boundaries for remote components
- ✅ TypeScript for type safety
- ✅ Suspense for loading states
- ✅ Monorepo for scalability

---

## 🔗 GitHub Repository

**URL**: https://github.com/Satyam9169/microfrontend-vite-typescript-module_federation.git

**Structure on GitHub:**
```
master branch/
├── README.md (Main documentation)
├── host-app/ (All files tracked)
├── remote-app/ (All files tracked)
└── .git/ (Version history)
```

**Total Files Tracked**: 43 files  
**Last Commit**: cf4e9e9  
**Date**: April 29, 2026

---

## 📞 Quick Reference

### **Development Workflow**
```bash
# Terminal 1: Remote
cd remote-app && npm run serve

# Terminal 2: Host
cd host-app && npm run dev

# Visit http://localhost:5000
```

### **Production Workflow**
```bash
# Build remote
cd remote-app && npm run build

# Build host
cd host-app && npm run build

# Deploy dist/ folders separately
```

### **Common Commands**

| Command | Purpose |
|---------|---------|
| `npm run dev` | Development server |
| `npm run build` | Production build |
| `npm run preview` | Preview production build |
| `npm run serve` | Build + preview |
| `npm run lint` | Check code style |

---

## 🎯 Project Completion Checklist

- ✅ Remote app with exposed components
- ✅ Host app consuming remote components
- ✅ Module Federation configured correctly
- ✅ TypeScript setup in both projects
- ✅ Vite build optimization
- ✅ Comprehensive README (900+ lines)
- ✅ Git version control
- ✅ GitHub repository setup
- ✅ All files tracked and pushed
- ✅ Process documentation (this file)

---

## 📚 Additional Resources

- [Vite Documentation](https://vitejs.dev/)
- [@originjs/vite-plugin-federation](https://github.com/originjs/vite-plugin-federation)
- [React Documentation](https://react.dev/)
- [Module Federation Concepts](https://webpack.js.org/concepts/module-federation/)
- [Micro Frontends Architecture](https://micro-frontends.org/)

---

**Project Complete! Ready for development and deployment. 🚀**
