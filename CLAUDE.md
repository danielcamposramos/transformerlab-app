# CLAUDE.md - AI Assistant Guide for Transformer Lab

> **Version**: 0.25.0
> **Last Updated**: 2025-11-18
> **Purpose**: Comprehensive guide for AI assistants working with the Transformer Lab codebase

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Codebase Structure](#codebase-structure)
4. [Development Workflows](#development-workflows)
5. [Key Technologies & Frameworks](#key-technologies--frameworks)
6. [Coding Conventions](#coding-conventions)
7. [State Management](#state-management)
8. [API Integration](#api-integration)
9. [Build System](#build-system)
10. [Testing](#testing)
11. [Component Development](#component-development)
12. [Common Tasks](#common-tasks)
13. [Deployment & Publishing](#deployment--publishing)
14. [Important Notes](#important-notes)

---

## Project Overview

**Transformer Lab** is a cross-platform desktop application built with Electron and React that provides a complete toolkit for working with Large Language Models (LLMs). It allows users to download, train, fine-tune, chat with, and evaluate LLMs locally or remotely.

### Key Features
- One-click model downloads from HuggingFace
- Fine-tuning and training (MLX on Apple Silicon, Huggingface on GPU)
- RLHF and preference optimization (DPO, ORPO, SIMPO)
- Chat and completions interface
- RAG (Retrieval Augmented Generation)
- Multiple inference engines (MLX, FastChat, vLLM, Llama CPP, SGLang)
- Image diffusion models support
- Plugin system
- Full REST API
- Cloud/remote execution support

### Tech Stack at a Glance
- **Desktop Framework**: Electron 26.2.1
- **UI Framework**: React 18.2.0
- **Language**: TypeScript 5.1.3
- **UI Library**: MUI Joy UI 5.0.0-beta.48
- **Bundler**: Webpack 5.98.0
- **State**: React Context + SWR (data fetching)
- **Node**: >=14.x (recommend v22, NOT v23)

---

## Architecture

### Dual-Mode Application

Transformer Lab has a unique dual-mode architecture:

1. **Desktop Mode** (Electron): Full-featured native application
   - Uses `src/main/preload.ts`
   - IPC communication with main process
   - electron-store for persistence
   - System integration (file dialogs, SSH, auto-updates)

2. **Cloud/Web Mode**: Browser-based deployment
   - Uses `src/main/preload-cloud.ts` (stub implementation)
   - localStorage for persistence
   - Same codebase as desktop, different build

**Runtime Detection**:
```typescript
window.platform.appmode // 'desktop' or 'cloud'
(window as any).TransformerLab?.API_URL // API endpoint
```

### Three-Process Architecture (Desktop)

```
┌─────────────────────────────────────────────────────┐
│                   Main Process                       │
│              (src/main/main.ts)                      │
│                                                       │
│  - Window management                                 │
│  - IPC handlers                                      │
│  - Server lifecycle (start/stop Python backend)      │
│  - Conda/system checks                               │
│  - SSH client                                        │
│  - Auto-updater                                      │
│  - electron-store (persistent storage)               │
└──────────────────┬──────────────────────────────────┘
                   │
                   │ IPC Bridge (preload.ts)
                   │
┌──────────────────▼──────────────────────────────────┐
│              Renderer Process                        │
│           (src/renderer/App.tsx)                     │
│                                                       │
│  - React UI                                          │
│  - State management (Context + SWR)                  │
│  - REST API client                                   │
│  - Component tree                                    │
└──────────────────┬──────────────────────────────────┘
                   │
                   │ REST API (HTTP/HTTPS)
                   │
┌──────────────────▼──────────────────────────────────┐
│             Python Backend                           │
│          (separate repository)                       │
│                                                       │
│  - FastAPI REST server                               │
│  - Model operations                                  │
│  - Training/inference                                │
│  - File management                                   │
└─────────────────────────────────────────────────────┘
```

### Component Organization

```
App (grid layout)
├── Header
│   ├── Connection status
│   ├── Theme switcher
│   └── User menu
├── Sidebar Navigation
│   ├── Experiment selector
│   ├── Main navigation menu
│   └── Secondary actions
├── MainAppPanel (dynamic content)
│   ├── Experiment views
│   │   ├── Foundation (model selection)
│   │   ├── Interact (chat interface)
│   │   ├── Train (fine-tuning)
│   │   ├── Eval (evaluation)
│   │   ├── Workflows (DAG editor)
│   │   ├── RAG
│   │   ├── Diffusion
│   │   └── API
│   ├── ModelZoo
│   ├── Data (datasets)
│   ├── Plugins
│   └── Settings
└── OutputTerminal (collapsible logs drawer)
```

---

## Codebase Structure

### Directory Layout

```
transformerlab-app/
├── src/
│   ├── main/                    # Electron main process
│   │   ├── main.ts              # Entry point, IPC handlers, window management
│   │   ├── util.ts              # Server management, conda detection
│   │   ├── preload.ts           # Desktop IPC bridge
│   │   ├── preload-cloud.ts     # Cloud stub bridge
│   │   ├── menu.ts              # Application menu
│   │   ├── ssh-client.js        # SSH connectivity
│   │   ├── file-watcher.js      # File system watching
│   │   └── shell_commands/      # Shell utilities
│   │
│   ├── renderer/                # React UI (renderer process)
│   │   ├── index.tsx            # App entry with providers
│   │   ├── App.tsx              # Main app component
│   │   ├── store.js             # easy-peasy store (minimal usage)
│   │   ├── preload.d.ts         # TypeScript definitions for IPC
│   │   ├── styles.css           # Global styles
│   │   │
│   │   ├── lib/                 # Shared utilities
│   │   │   ├── api-client/      # REST API integration
│   │   │   │   ├── endpoints.ts # API endpoint definitions
│   │   │   │   ├── hooks.ts     # useAPI, useModelStatus, etc.
│   │   │   │   ├── functions.ts # Auth, file operations
│   │   │   │   ├── chat.ts      # Chat/completions API
│   │   │   │   └── urls.ts      # API URL helpers
│   │   │   ├── authContext.ts   # Authentication context
│   │   │   ├── ExperimentInfoContext.js  # Experiment context
│   │   │   ├── theme.ts         # MUI Joy theme
│   │   │   └── utils.ts         # Helper functions
│   │   │
│   │   └── components/          # React components (201 files)
│   │       ├── Nav/             # Navigation components
│   │       ├── Experiment/      # Core LLM features (40+ components)
│   │       │   ├── Foundation/
│   │       │   ├── Interact/
│   │       │   ├── Train/
│   │       │   ├── Eval/
│   │       │   ├── Workflows/   # Xyflow-based workflow editor
│   │       │   ├── Rag/
│   │       │   ├── Diffusion/
│   │       │   └── Audio/
│   │       ├── ModelZoo/
│   │       ├── Data/
│   │       ├── Plugins/
│   │       ├── Settings/
│   │       ├── Shared/          # Reusable components
│   │       └── User/
│   │
│   └── __tests__/               # Jest tests
│       └── App.test.tsx
│
├── .erb/                        # Electron React Boilerplate config
│   ├── configs/                 # Webpack configurations
│   │   ├── webpack.config.base.ts
│   │   ├── webpack.config.main.prod.ts
│   │   ├── webpack.config.renderer.prod.ts
│   │   ├── webpack.config.cloud.prod.ts
│   │   ├── webpack.config.*.dev.ts
│   │   └── webpack.paths.ts
│   ├── scripts/                 # Build scripts
│   │   ├── check-native-dep.js
│   │   ├── check-port-in-use.js
│   │   ├── clean.js
│   │   └── notarize.js
│   └── mocks/                   # Test mocks
│
├── scripts/                     # Project scripts
│   ├── server.js               # Static file server for cloud build
│   ├── openapi.json            # Backend API schema
│   ├── bump_version.js
│   └── orval/                  # TypeScript SDK generation
│       ├── generateSDK.sh
│       └── orval.config.js
│
├── release/
│   ├── app/                    # Runtime dependencies
│   │   └── package.json        # (ssh2, etc.)
│   ├── build/                  # Build output (platform packages)
│   └── cloud/                  # Cloud build output
│
├── assets/                     # App resources (icons, logos)
├── .github/workflows/          # CI/CD pipelines
├── package.json                # Root dependencies
├── tsconfig.json               # TypeScript configuration
├── .eslintrc.js                # ESLint rules
└── .prettierrc.json            # Prettier formatting
```

### Key Directories Explained

| Directory | Purpose |
|-----------|---------|
| `src/main/` | Electron main process: system operations, IPC, server lifecycle |
| `src/renderer/` | React application: UI components, API client, contexts |
| `src/renderer/lib/api-client/` | REST API integration layer |
| `src/renderer/components/` | All React components organized by feature |
| `src/renderer/components/Experiment/` | Core LLM functionality |
| `src/renderer/components/Shared/` | Reusable UI components |
| `.erb/configs/` | Webpack build configurations |
| `scripts/orval/` | OpenAPI-based SDK generation |
| `release/app/` | Runtime-only dependencies |

---

## Development Workflows

### Getting Started

```bash
# Prerequisites: Node.js 22 (NOT v23), npm >=7.x

# Install dependencies
npm install

# Start development server (renderer)
npm start

# Start main process (in separate terminal)
npm run start:main

# Start cloud variant
npm start:cloud
```

### Development Commands

```bash
# Linting
npm run lint                    # Run ESLint
npm run format                  # Format with Prettier
npm run format:check            # Check formatting

# Testing
npm test                        # Run Jest tests
npm exec tsc                    # TypeScript type checking

# Building
npm run build                   # Build all (main + renderer + cloud)
npm run build:main              # Build main process only
npm run build:renderer          # Build renderer only
npm run build:cloud             # Build cloud variant only

# Packaging
npm run package                 # Package for local platform
```

### Git Workflow

1. **Branch Naming**: Use descriptive branch names (e.g., `feature/workflow-editor`, `fix/chat-streaming`)
2. **Commits**: Write clear commit messages following the existing style
3. **PR Process**:
   - All PRs trigger CI (test, lint, prettier check)
   - Requires passing tests on macOS, Windows, Ubuntu
   - Code must pass ESLint and Prettier checks

### CI/CD Pipeline

**GitHub Actions Workflows** (`.github/workflows/`):

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `test.yml` | Push, PR | Runs `npm run package`, lint, tsc, test on all platforms |
| `lint.yml` | Push | ESLint check |
| `prettier.yml` | Push to main, PR to main | Formatting validation |
| `publish.yml` | Tag push (`v*`), manual | Build and publish releases |
| `codeql-analysis.yml` | Schedule | Security analysis |

### Release Process

1. Update version in `package.json`
2. Create git tag: `git tag v0.25.0`
3. Push tag: `git push origin v0.25.0`
4. GitHub Actions builds and publishes:
   - macOS DMG (notarized)
   - Windows EXE (signed with SSL.com eSigner)
   - Linux AppImage
   - Cloud build tarball (`transformerlab_web.tar.gz`)

---

## Key Technologies & Frameworks

### Core Stack

#### Electron (26.2.1)
- **Main Process**: Node.js backend (system access, IPC)
- **Renderer Process**: Chromium-based UI (React)
- **Preload Script**: Security bridge between main and renderer

#### React (18.2.0)
- **JSX Transform**: `react-jsx` (no need to import React)
- **Router**: react-router-dom 6.11.2 (HashRouter in desktop, BrowserRouter in cloud)
- **Hooks**: Extensive use of custom hooks

#### TypeScript (5.1.3)
- **Target**: ES2021
- **Strict Mode**: Enabled
- **Base URL**: `./src` (allows clean imports)

### UI & Visualization

#### MUI Joy UI (5.0.0-beta.48)
- Primary component library
- Custom theme in `src/renderer/lib/theme.ts`
- Emotion CSS-in-JS styling

#### Specialized Libraries
- **Monaco Editor** (4.7.0): Code editor (for plugins, scripts)
- **Xyflow** (12.4.4): Workflow DAG editor
- **Nivo** (0.88.0): Charts (bar, line, radar)
- **WaveSurfer.js** (7.10.1): Audio visualization
- **xterm** (5.5.0): Terminal emulator
- **Three.js** (0.175.0): 3D model visualization
- **React Markdown** (8.0.7): Markdown rendering

### Data Management

#### SWR (2.3.2)
- Primary data fetching library
- Stale-while-revalidate caching
- Auto-revalidation on focus
- Error boundary integration

#### easy-peasy (6.1.0)
- Redux alternative (minimal usage)
- Store in `src/renderer/store.js`
- Consider migrating to pure React Context if extending state

#### React Context API
- **AuthContext**: Token management, user info
- **ExperimentInfoContext**: Current experiment metadata
- **AnalyticsContext**: Event tracking

### File Uploads

#### Uppy (3.8.0+)
- Dashboard, drag-drop, progress bar, status bar
- XHR upload with authentication
- Used for dataset and document uploads

### Development Tools

#### Webpack (5.98.0)
- Three separate configurations (main, renderer, cloud)
- Hot reload in development
- Tree shaking and minification in production
- Bundle analyzer available (`--ANALYZE` flag)

#### Orval
- Generates TypeScript SDK from OpenAPI spec
- Source: `scripts/openapi.json`
- Output: `src/renderer/lib/transformerlab-api-sdk.ts`
- Run: `scripts/orval/generateSDK.sh`

#### ESLint + Prettier
- Config: `erb` preset (Electron React Boilerplate)
- TypeScript support
- Auto-fixes available: `npm run format`

#### Jest + React Testing Library
- Test runner: Jest 29.5.0
- Component testing: @testing-library/react 14.0.0
- Environment: jsdom
- Transform: ts-jest

---

## Coding Conventions

### File Naming

- **Components**: PascalCase (e.g., `ModelZoo.tsx`, `ChatInterface.tsx`)
- **Utilities**: camelCase (e.g., `authContext.ts`, `utils.ts`)
- **Hooks**: camelCase with `use` prefix (e.g., `useAPI.ts`)
- **Tests**: `*.test.tsx` or `*.test.ts`

### TypeScript

```typescript
// Import styles
import styles from './Component.module.scss'

// Props interface
interface ComponentProps {
    title: string
    onSubmit?: () => void
}

// Function components (arrow functions preferred)
export default function Component({ title, onSubmit }: ComponentProps) {
    // Implementation
}

// Or named export
export const Component = ({ title }: ComponentProps) => {
    // Implementation
}
```

### Formatting (Prettier)

```json
{
  "trailingComma": "es5",
  "tabWidth": 4,
  "semi": false,
  "singleQuote": true
}
```

**Key Points**:
- **4 spaces** for indentation
- **No semicolons**
- **Single quotes** for strings
- **Trailing commas** (ES5 style)

### ESLint Rules

Important overrides:
- React in JSX scope: OFF (using JSX transform)
- No unused vars: ERROR (TypeScript version)
- No shadow: ERROR (TypeScript version)
- Prop types: OFF (using TypeScript)
- No plusplus: OFF (allow `i++`)

### Import Organization

```typescript
// 1. External dependencies
import React from 'react'
import { Box, Button } from '@mui/joy'
import { useNavigate } from 'react-router-dom'

// 2. Internal utilities/contexts
import { useAPI } from 'lib/api-client/hooks'
import { useExperimentInfo } from 'lib/ExperimentInfoContext'

// 3. Components
import ModelCard from './ModelCard'

// 4. Styles
import styles from './Component.module.scss'
```

### Component Structure

```typescript
export default function ExampleComponent({ prop1, prop2 }: Props) {
    // 1. Hooks
    const navigate = useNavigate()
    const { experimentId } = useExperimentInfo()
    const { data, error, isLoading } = useAPI('models', ['list'])

    // 2. State
    const [localState, setLocalState] = useState('')

    // 3. Effects
    useEffect(() => {
        // Side effects
    }, [dependency])

    // 4. Event handlers
    const handleClick = () => {
        // Handler logic
    }

    // 5. Conditional early returns
    if (isLoading) return <div>Loading...</div>
    if (error) return <div>Error: {error}</div>

    // 6. Render
    return (
        <Box>
            {/* JSX */}
        </Box>
    )
}
```

---

## State Management

### Global State Layers

1. **URL State** (React Router)
   - Current route/tab
   - Experiment ID in some views

2. **Context State** (React Context)
   - Authentication (access token, user info)
   - Current experiment metadata
   - Analytics tracking

3. **Server State** (SWR)
   - API data (models, datasets, jobs, etc.)
   - Automatic caching and revalidation
   - Error handling

4. **Local Storage**
   - electron-store (desktop) or localStorage (cloud)
   - Persistent settings
   - Last selected experiment

5. **Component State** (useState)
   - Local UI state (form inputs, modals, etc.)

### Using SWR for Data Fetching

**Basic Pattern**:
```typescript
import { useAPI } from 'lib/api-client/hooks'

function ModelList() {
    const { data, error, isLoading, mutate } = useAPI('models', ['list'])

    if (isLoading) return <div>Loading...</div>
    if (error) return <div>Error loading models</div>

    return (
        <div>
            {data.map(model => (
                <div key={model.id}>{model.name}</div>
            ))}
        </div>
    )
}
```

**Mutate for Optimistic Updates**:
```typescript
const { mutate } = useAPI('models', ['list'])

const deleteModel = async (modelId: string) => {
    // Optimistic update
    mutate(
        currentData => currentData.filter(m => m.id !== modelId),
        false // Don't revalidate yet
    )

    // Perform actual deletion
    await fetch(`/api/models/${modelId}`, { method: 'DELETE' })

    // Revalidate from server
    mutate()
}
```

### Authentication Context

```typescript
import { useAuth } from 'lib/authContext'

function MyComponent() {
    const { accessToken, setAccessToken, userInfo } = useAuth()

    // Use accessToken for authenticated requests
    // userInfo contains user details
}
```

### Experiment Context

```typescript
import { useExperimentInfo } from 'lib/ExperimentInfoContext'

function MyComponent() {
    const { experimentId, experimentInfo } = useExperimentInfo()

    // experimentId: current experiment ID
    // experimentInfo: full experiment metadata
}
```

---

## API Integration

### Backend Architecture

The frontend communicates with a **separate Python backend** (not in this repository) via REST API. The backend is typically:
- FastAPI-based
- Runs on localhost:8000 (default) or remote server
- Provides endpoints for models, experiments, jobs, RAG, etc.

### API Client Structure

All API interaction is centralized in `src/renderer/lib/api-client/`:

```
api-client/
├── endpoints.ts    # All endpoint definitions
├── hooks.ts        # SWR hooks (useAPI, useModelStatus, etc.)
├── functions.ts    # Helper functions (login, authenticatedFetch, etc.)
├── chat.ts         # Chat/completion streaming
└── urls.ts         # API URL resolution
```

### Endpoint Organization

Endpoints are organized by entity in `endpoints.ts`:

```typescript
import Endpoints from 'lib/api-client/endpoints'

// Examples:
Endpoints.Models.LocalList()
Endpoints.Experiment.GetAll()
Endpoints.Jobs.List(experimentId)
Endpoints.Rag.Query(experimentId, model, query, settings)
Endpoints.Workflows.Run(experimentId, workflowId)
```

**Major Entities**:
- `Models`: Model gallery, local models, download, delete
- `Experiment`: CRUD, config, scripts
- `Jobs`: List, create, stop, output
- `Datasets`: List, upload, process
- `Rag`: Query, re-index, documents
- `Workflows`: List, create, run, update
- `Plugins`: Gallery, install, list
- `Tools`: MCP tool calling

### Making API Requests

#### Using Hooks (Recommended)

```typescript
import { useAPI } from 'lib/api-client/hooks'

// Simple GET
const { data, error, isLoading } = useAPI('models', ['list'])

// With parameters
const { data } = useAPI('experiment', ['get'], { id: experimentId })

// Custom options
const { data } = useAPI(
    'jobs',
    ['list'],
    { experimentId },
    { refreshInterval: 5000 } // Poll every 5s
)
```

#### Direct Fetch (for mutations)

```typescript
import { authenticatedFetch } from 'lib/api-client/functions'
import Endpoints from 'lib/api-client/endpoints'

// POST request
const createExperiment = async (name: string) => {
    const url = Endpoints.Experiment.Create(name)
    const response = await authenticatedFetch(url, { method: 'POST' })
    return response.json()
}

// DELETE request
const deleteModel = async (modelId: string) => {
    const url = Endpoints.Models.Delete(modelId)
    await authenticatedFetch(url, { method: 'DELETE' })
}
```

#### Streaming Responses

For chat completions and log streaming:

```typescript
import { streamingChatQuery } from 'lib/api-client/chat'

const response = await streamingChatQuery(
    experimentId,
    modelId,
    messages,
    onChunk, // Callback for each chunk
    settings
)
```

### Authentication

All requests automatically include the access token:

```typescript
// Login flow
import { login, setAccessToken, getAccessToken } from 'lib/api-client/functions'

// Login
const token = await login(username, password)

// Token is automatically used in all authenticatedFetch calls
const data = await authenticatedFetch('/api/protected-endpoint')
```

### API URL Configuration

The API URL is dynamically determined:

```typescript
// Desktop: Set by main process based on connection (local/SSH/remote)
// Cloud: Set via query parameter or environment variable

const apiUrl = API_URL() // from urls.ts
```

### Error Handling

```typescript
const { data, error } = useAPI('models', ['list'])

if (error) {
    // Handle error
    // 401: Triggers auth context to show login modal
    // Others: Display error message
}
```

---

## Build System

### Webpack Configuration

Three separate webpack configs for different targets:

#### 1. Main Process (`webpack.config.main.prod.ts`)
- Target: Electron main process
- Entry: `src/main/main.ts`
- Output: `dist/main/main.js` (CommonJS)
- Node externals, no minification

#### 2. Renderer Process (`webpack.config.renderer.prod.ts`)
- Target: Web/Electron renderer
- Entry: `src/renderer/index.tsx`
- Output: `dist/renderer/` (index.html, main.js, style.css)
- Minification, source maps, CSS extraction

#### 3. Cloud Build (`webpack.config.cloud.prod.ts`)
- Target: Browser-only
- Entry: `src/renderer/index.tsx` + `preload-cloud.ts`
- Output: `release/cloud/`
- Similar to renderer but uses cloud preload

### Build Process

```bash
# Production build (all three)
npm run build

# Development mode
npm run start:renderer  # Webpack dev server on :1212
npm run start:main      # Electronmon (auto-restart)
npm run start:cloud     # Webpack dev server for cloud variant
```

### Key Webpack Plugins

- **HtmlWebpackPlugin**: Generates HTML from `src/renderer/index.ejs`
- **MiniCssExtractPlugin**: Extracts CSS to separate file
- **TerserPlugin**: JavaScript minification
- **CssMinimizerPlugin**: CSS minification
- **BundleAnalyzerPlugin**: Bundle size analysis (use `--ANALYZE` flag)

### Environment Variables

Set via `webpack.EnvironmentPlugin`:
- `NODE_ENV`: 'production' or 'development'
- `VERSION`: from package.json

### TypeScript Compilation

```bash
# Type checking (no emit)
npm exec tsc

# Webpack handles transpilation via ts-loader
```

### Packaging for Distribution

```bash
# Package for current platform
npm run package

# Outputs to release/build/
# - macOS: .dmg
# - Windows: .exe (NSIS installer)
# - Linux: .AppImage
```

**electron-builder** configuration in `package.json`:
- macOS: Notarization, code signing
- Windows: Code signing via SSL.com eSigner
- Linux: AppImage with arm64 + x64

---

## Testing

### Test Stack

- **Jest**: 29.5.0 (test runner)
- **ts-jest**: TypeScript support
- **React Testing Library**: Component testing
- **jest-dom**: DOM matchers

### Running Tests

```bash
npm test              # Run all tests
npm test -- --watch   # Watch mode
npm test -- --coverage # Coverage report
```

### Test File Location

- Tests live in `src/__tests__/`
- Co-located tests also allowed (`.test.tsx` next to component)

### Example Test

```typescript
import { render, screen } from '@testing-library/react'
import App from '../App'

describe('App', () => {
    it('should render', () => {
        render(<App />)
        expect(screen.getByText(/Transformer Lab/i)).toBeInTheDocument()
    })
})
```

### Mocking

**Module Mocks** (jest.config.json):
```json
{
  "moduleNameMapper": {
    "\\.(jpg|jpeg|png|...)$": "<rootDir>/.erb/mocks/fileMock.js",
    "\\.(css|less|sass|scss)$": "identity-obj-proxy"
  }
}
```

**API Mocks**:
```typescript
import { rest } from 'msw'
import { setupServer } from 'msw/node'

const server = setupServer(
    rest.get('/api/models/list', (req, res, ctx) => {
        return res(ctx.json([{ id: '1', name: 'Test Model' }]))
    })
)

beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

---

## Component Development

### Creating a New Component

1. **Choose Location**:
   - Feature-specific: `src/renderer/components/Experiment/<Feature>/`
   - Shared/reusable: `src/renderer/components/Shared/`

2. **File Structure**:
   ```
   MyComponent/
   ├── index.tsx          # Component logic
   ├── MyComponent.scss   # Styles (if needed)
   └── SubComponent.tsx   # Sub-components (if complex)
   ```

3. **Component Template**:
   ```typescript
   import { Box, Button } from '@mui/joy'
   import { useState } from 'react'
   import { useAPI } from 'lib/api-client/hooks'

   interface MyComponentProps {
       title: string
       onAction?: () => void
   }

   export default function MyComponent({ title, onAction }: MyComponentProps) {
       const [state, setState] = useState('')
       const { data, isLoading } = useAPI('entity', ['endpoint'])

       if (isLoading) return <div>Loading...</div>

       return (
           <Box>
               <h1>{title}</h1>
               <Button onClick={onAction}>Action</Button>
           </Box>
       )
   }
   ```

### Using MUI Joy Components

```typescript
import {
    Box,           // Container
    Button,        // Button
    Input,         // Text input
    Select,        // Dropdown
    Option,        // Select option
    Typography,    // Text
    Modal,         // Modal dialog
    Sheet,         // Card/panel
    Stack,         // Flex container
    Grid,          // Grid layout
    Divider,       // Separator
    IconButton,    // Icon button
    Chip,          // Badge/chip
    LinearProgress // Progress bar
} from '@mui/joy'
```

### Styling

**Option 1: MUI sx prop** (preferred for simple styles)
```typescript
<Box sx={{ padding: 2, backgroundColor: 'primary.main' }}>
    Content
</Box>
```

**Option 2: CSS Modules**
```typescript
import styles from './Component.module.scss'

<div className={styles.container}>Content</div>
```

**Option 3: Emotion CSS-in-JS**
```typescript
import { css } from '@emotion/react'

const style = css`
    padding: 16px;
    background: red;
`

<div css={style}>Content</div>
```

### Routing

```typescript
import { useNavigate, useParams, useLocation } from 'react-router-dom'

function MyComponent() {
    const navigate = useNavigate()
    const { experimentId } = useParams()
    const location = useLocation()

    const goToHome = () => navigate('/')
    const goToExperiment = (id: string) => navigate(`/experiment/${id}`)
}
```

### Forms

```typescript
import { useState } from 'react'
import { Input, Button, FormControl, FormLabel } from '@mui/joy'

function MyForm() {
    const [value, setValue] = useState('')

    const handleSubmit = async (e: React.FormEvent) => {
        e.preventDefault()
        // Submit logic
    }

    return (
        <form onSubmit={handleSubmit}>
            <FormControl>
                <FormLabel>Name</FormLabel>
                <Input
                    value={value}
                    onChange={(e) => setValue(e.target.value)}
                    required
                />
            </FormControl>
            <Button type="submit">Submit</Button>
        </form>
    )
}
```

---

## Common Tasks

### Adding a New API Endpoint

1. **Update OpenAPI Schema** (if backend changed):
   ```bash
   # Update scripts/openapi.json with new endpoint definition
   ```

2. **Regenerate SDK**:
   ```bash
   cd scripts/orval
   ./generateSDK.sh
   ```

3. **Add to Endpoints** (`src/renderer/lib/api-client/endpoints.ts`):
   ```typescript
   Endpoints.MyEntity = {
       NewEndpoint: (param: string) => `${API_URL()}my-entity/${param}/action`,
   }
   ```

4. **Use in Component**:
   ```typescript
   const { data } = useAPI('my-entity', ['action'], { param: 'value' })
   ```

### Adding a New Route

1. **Update Routes** (in `App.tsx` or `MainAppPanel.tsx`):
   ```typescript
   <Route path="/my-new-page" element={<MyNewPage />} />
   ```

2. **Add Navigation Link** (in `Nav/` component):
   ```typescript
   <Button onClick={() => navigate('/my-new-page')}>
       My New Page
   </Button>
   ```

### Adding a New IPC Handler (Desktop Only)

1. **Add to Main Process** (`src/main/main.ts`):
   ```typescript
   ipcMain.handle('my-custom-action', async (event, arg) => {
       // Perform system-level operation
       return result
   })
   ```

2. **Expose in Preload** (`src/main/preload.ts`):
   ```typescript
   contextBridge.exposeInMainWorld('myAPI', {
       doAction: (arg: string) => ipcRenderer.invoke('my-custom-action', arg),
   })
   ```

3. **Add TypeScript Definition** (`src/renderer/preload.d.ts`):
   ```typescript
   interface Window {
       myAPI: {
           doAction: (arg: string) => Promise<any>
       }
   }
   ```

4. **Use in Renderer**:
   ```typescript
   const result = await window.myAPI.doAction('test')
   ```

### Adding a Plugin

Plugins extend Transformer Lab's functionality. They live in the Python backend but are managed from the UI.

1. **Create Plugin** (in backend repo):
   - Follow plugin structure (metadata.json, main.py, etc.)

2. **Add to Gallery** (backend):
   - Update plugin registry

3. **Install from UI**:
   - Navigate to Plugins → Gallery
   - Click Install on your plugin

### Updating Dependencies

```bash
# Check outdated packages
npm outdated

# Update specific package
npm install <package>@latest

# Update all (careful!)
npm update

# Audit for vulnerabilities
npm audit
npm audit fix
```

### Debugging

**Renderer Process** (Chrome DevTools):
- In desktop app: View → Toggle Developer Tools
- In browser: F12

**Main Process** (Node.js debugger):
```bash
# Add to .vscode/launch.json:
{
    "type": "node",
    "request": "launch",
    "name": "Electron: Main",
    "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron",
    "runtimeArgs": ["--remote-debugging-port=9223", "."],
    "protocol": "inspector"
}
```

**Network Requests**:
- Chrome DevTools → Network tab
- Check Authorization headers
- Inspect request/response payloads

---

## Deployment & Publishing

### Version Bumping

```bash
# Update version in package.json
node scripts/bump_version.js 0.26.0

# Or manually edit package.json
```

### Creating a Release

```bash
# Commit version bump
git add package.json
git commit -m "Bump version to 0.26.0"

# Create tag
git tag v0.26.0

# Push tag (triggers publish workflow)
git push origin v0.26.0
```

### Publish Workflow (Automated)

When a tag is pushed:
1. GitHub Actions builds on macOS (matrix includes Windows, Linux)
2. Runs `npm run build`
3. Packages with electron-builder:
   - macOS: Notarized DMG
   - Windows: Signed EXE
   - Linux: AppImage (arm64 + x64)
4. Generates `latest.yml` for auto-update
5. Uploads assets to GitHub Release
6. Builds cloud variant (`release/cloud/`)
7. Creates `transformerlab_web.tar.gz` for web deployment

### Manual Package

```bash
# Build for current platform
npm run package

# Output in release/build/
```

### Auto-Update

Desktop app uses `electron-updater`:
- Checks for updates on startup
- Downloads in background
- Prompts user to install
- Uses `latest.yml` from GitHub Releases

---

## Important Notes

### Node.js Version

**Use Node.js v22, NOT v23**. The current build has compatibility issues with v23.

```bash
# Check version
node -v  # Should be v22.x.x

# Use nvm to switch
nvm use 22
```

### API Backend Separation

The Python backend is **separate** from this repository. This repo only contains the Electron/React frontend. To fully run Transformer Lab locally:

1. Clone and run the backend (transformerlab-api repo)
2. Start this frontend
3. Frontend connects to backend via REST API

### External Data Directories

The app creates data directories outside the app bundle:
- **macOS**: `~/Library/Application Support/transformerlab/`
- **Windows**: `%APPDATA%/transformerlab/`
- **Linux**: `~/.config/transformerlab/`

Stores:
- Models
- Datasets
- Experiments
- Plugins
- User settings (electron-store)

### IPC vs REST API

**IPC** (Main ↔ Renderer):
- System operations (file dialogs, SSH, conda checks)
- Storage access (electron-store)
- Auto-updates
- Desktop-specific features

**REST API** (Renderer ↔ Backend):
- All LLM operations
- Model/dataset management
- Training/inference
- Works in both desktop and cloud modes

### Security Considerations

1. **Context Isolation**: Enabled (preload script is the only bridge)
2. **Node Integration**: Disabled in renderer
3. **Authentication**: Bearer token in Authorization header
4. **CORS**: Backend must allow frontend origin
5. **CSP**: Consider adding Content-Security-Policy headers

### Performance Tips

1. **SWR Caching**: Leverage automatic caching, avoid redundant fetches
2. **React.memo**: Memoize expensive components
3. **useMemo/useCallback**: Prevent unnecessary re-renders
4. **Virtualization**: For long lists (react-window)
5. **Code Splitting**: Use React.lazy() for heavy components
6. **Bundle Analysis**: Run build with `--ANALYZE` flag

### Known Issues & Workarounds

- **SSH Connections**: electron-ssh2 can be flaky; handle disconnects gracefully
- **File Watchers**: chokidar may hit OS limits on large directories
- **Electron Store**: Can't be used in tests; mock `window.storage`
- **Monaco Editor**: Heavy; lazy load it
- **Cloud Mode**: Some features disabled (SSH, auto-update, file dialogs)

### Contributing Guidelines

1. Follow existing code style (ESLint + Prettier)
2. Write tests for new features
3. Update this CLAUDE.md if adding major features
4. Keep commits atomic and well-described
5. Ensure CI passes before merging

### Resources

- **Official Docs**: https://transformerlab.ai/docs/
- **Discord**: https://discord.gg/transformerlab
- **GitHub**: https://github.com/transformerlab/transformerlab-app
- **Issues**: https://github.com/transformerlab/transformerlab-app/issues

---

## Quick Reference

### Essential Commands
```bash
npm install          # Install dependencies
npm start            # Start dev server
npm run lint         # Lint code
npm run format       # Format code
npm test             # Run tests
npm run build        # Production build
npm run package      # Create distributable
```

### Key Files
- `src/main/main.ts` - Electron entry point
- `src/renderer/App.tsx` - React root component
- `src/renderer/lib/api-client/endpoints.ts` - API definitions
- `src/renderer/lib/authContext.ts` - Authentication
- `.erb/configs/webpack.config.*.ts` - Build configs
- `package.json` - Dependencies & scripts

### Contexts
```typescript
useAuth()            // Access token, user info
useExperimentInfo()  // Current experiment
```

### API Hooks
```typescript
useAPI(entity, path, params, options)  // Fetch data
useModelStatus(modelId)                // Model status
```

### Navigation
```typescript
useNavigate()        // Programmatic navigation
useParams()          // URL parameters
useLocation()        // Current location
```

---

**End of CLAUDE.md**

> This document is a living guide. Update it as the codebase evolves.
> Last updated: 2025-11-18 (v0.25.0)
