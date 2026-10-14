# 📚 ChefMind AI — Technical Documentation

Comprehensive technical documentation, API specifications, and operational guides for ChefMind AI. For high-level product highlights and the local quick start, see the [main README](../README.md).

---

## 📑 Contents
- [Architecture & Diagrams](#-architecture--diagram-index)
- [API Reference](#-api-reference)
- [Deployment Guide](#-deployment-guide)
- [Testing & Continuous Integration](#-testing--continuous-integration)

---

## 🏗 Architecture & Diagram Index

Detailed system design, data flow sequences, and component hierarchies are maintained in [**`architecture.md`**](architecture.md).

| Diagram | Description | Section Link |
|---|---|---|
| **System Architecture** | Full-stack topology, client-server boundary, DB & AI connections | [architecture.md §2](architecture.md#2-system-architecture) |
| **Frontend Components** | React hierarchy, state lifts, modal overlays, audio/camera bridges | [architecture.md §3](architecture.md#3-frontend-component-architecture) |
| **Middleware Pipeline** | Request lifecycle from Helmet and CORS to rate limiters and JWT | [architecture.md §4](architecture.md#4-backend-middleware-pipeline) |
| **User Journey** | End-to-end flow from ingredient input to cookbook management | [architecture.md §5](architecture.md#5-end-to-end-user-journey) |
| **Recipe Generation Sequence** | Gemini prompt construction, fallback cascades, and sanitization | [architecture.md §6](architecture.md#6-request-lifecycle-recipe-generation) |
| **Image Detection Flow** | Multer buffer validation, magic byte checks, and Gemini Vision | [architecture.md §7](architecture.md#7-image-ingredient-detection-flow) |
| **Recipe Chat Flow** | Live context injection, recipe updates, and clarification flows | [architecture.md §8](architecture.md#8-recipe-chat-flow) |
| **Gemini Fallback Chain** | Automated model failover circuit (3.8 Flash ➔ 3.7 Flash ➔ 3.5 Flash) | [architecture.md §9](architecture.md#9-gemini-fallback-chain) |
| **Data Model (ERD)** | User identity and Recipe Mongoose schema specifications | [architecture.md §10](architecture.md#10-data-model) |
| **Authentication Flow** | Registration, bcrypt hashing, JWT issuance, and bearer verification | [architecture.md §11](architecture.md#11-authentication-flow) |
| **Security Model** | Defense-in-depth security layers, headers, and isolation controls | [architecture.md §12](architecture.md#12-security-model) |

---

## 🔌 API Reference

All API responses follow consistent JSON envelopes:
- **Success:** `{ success: true, ...data }`
- **Error:** `{ success: false, message: "Human-readable explanation" }`

Endpoints marked with 🔒 require an `Authorization: Bearer <token>` header issued during login.

| Method | Endpoint | Auth | Description |
|:---:|---|:---:|---|
| `POST` | `/api/auth/register` | Public | Register new user account with email & password |
| `POST` | `/api/auth/login` | Public | Authenticate user credentials and receive a signed JWT |
| `GET` | `/api/auth/me` | 🔒 Bearer | Fetch profile and authentication state of current session |
| `POST` | `/api/ingredients/text` | Public | Parse and clean a raw, comma-separated text ingredient string |
| `POST` | `/api/ingredients/image` | 🔒 Bearer | Validate single image upload (utility endpoint) |
| `POST` | `/api/ingredients/image-analyze` | 🔒 Bearer | Analyze up to 5 uploaded pantry photos via Gemini Vision |
| `POST` | `/api/recipes/generate` | 🔒 Bearer | Generate a structured recipe (supports English & Hindi, custom instructions) |
| `POST` | `/api/recipes/chat` | 🔒 Bearer | Conversational AI chat over an active, unsaved recipe |
| `POST` | `/api/recipes/:id/chat` | 🔒 Bearer | Conversational AI chat scoped to a persisted recipe |
| `POST` | `/api/recipes/save` | 🔒 Bearer | Save an AI-generated or user-customized recipe to user's cookbook |
| `GET` | `/api/recipes` | 🔒 Bearer | Retrieve all recipes owned by the authenticated user |
| `PATCH` | `/api/recipes/:id/favorite` | 🔒 Bearer | Toggle favorite bookmark on a recipe card |
| `DELETE` | `/api/recipes/:id` | 🔒 Bearer | Permanently delete a recipe from the user's cookbook |
| `GET` | `/health` · `/api/health` | Public | Liveness probe returning process uptime and timestamp |
| `GET` | `/health/ready` · `/api/health/ready` | Public | Readiness probe checking active MongoDB database connectivity |

---

## ☁️ Deployment Guide

ChefMind AI is engineered to deploy cleanly as a modern decoupled full-stack application:

| Component | Recommended Hosts | Build & Runtime Configuration |
|---|---|---|
| **Backend API** | [Render](https://render.com) · [Railway](https://railway.app) · [Fly.io](https://fly.io) | Start Command: `npm start`<br/>Environment: `NODE_ENV=production`, `PORT`, `MONGODB_URI`, `JWT_SECRET`, `GEMINI_API_KEY`, `CLIENT_URL` |
| **Frontend SPA** | [Vercel](https://vercel.com) · [Netlify](https://netlify.com) · [Cloudflare Pages](https://pages.cloudflare.com) | Root Directory: `frontend`<br/>Build Command: `npm run build`<br/>Output Directory: `dist`<br/>Environment: `VITE_API_BASE_URL=https://your-backend.onrender.com` |

### Environment Variables Matrix
- `NODE_ENV`: Set to `production` in production hosting environments.
- `PORT`: Automatically provided by Render/Railway (defaults to `3000` locally).
- `CLIENT_URL`: Comma-separated list of allowed frontend domains for CORS (e.g., `https://chefmind.vercel.app`).
- `TRUST_PROXY`: Set to `true` when behind reverse proxies (Render, Cloudflare, AWS ALB) for accurate rate-limiting by client IP.
- `MONGODB_URI`: Production connection string to your MongoDB Atlas cluster.
- `JWT_SECRET`: Cryptographically secure secret (minimum 32 random characters).
- `GEMINI_API_KEY`: Secret Google Gemini API key from Google AI Studio.
- `VITE_API_BASE_URL`: (Frontend only) Target URL of your deployed backend service.

---

## 🧪 Testing & Continuous Integration

ChefMind AI uses Node's built-in, zero-dependency `node:test` framework for deterministic, blisteringly fast tests:

```bash
npm test
```

### Test Suite Highlights (69 Passing Tests):
- **Auth Suite**: Registration validation, duplicate email handling (409), password hashing verification, and JWT issuance.
- **Recipe Engine**: Structured JSON schema adherence, language normalization (English & Devanagari Hindi), serving scaling calculations, and custom preparation directives.
- **Multi-Modal Vision**: Magic byte file verification, image size limits (5MB per file, 15MB total), and mocked Gemini Vision responses.
- **Sous-Chef Chat**: In-context recipe mutations, ambiguous clarification handling, and user ownership isolation.
- **Security & Reliability**: Rate limiting ceilings, XSS header enforcement, sensitive key/path error redaction, and health probes.

Every commit and pull request to `main` triggers our automated [GitHub Actions CI Workflow](../.github/workflows/ci.yml) which validates the backend tests, frontend ESLint quality, and client production build across **Node.js 20 and Node.js 22**.
