# 🏗 ChefMind AI — Architecture & System Design

> How ChefMind AI is built: its layers, data flows, data model, security design and the reasons behind key decisions.
> 📚 Related docs: [API Reference](api.md) · [Deployment](deployment.md) · [Back to README](../README.md)

---

## Contents

1. [Architectural Overview](#1-architectural-overview)
2. [System Architecture](#2-system-architecture)
3. [Frontend Component Architecture](#3-frontend-component-architecture)
4. [Backend Middleware Pipeline](#4-backend-middleware-pipeline)
5. [End-to-End User Journey](#5-end-to-end-user-journey)
6. [Request Lifecycle: Recipe Generation](#6-request-lifecycle-recipe-generation)
7. [Image Ingredient Detection Flow](#7-image-ingredient-detection-flow)
8. [Recipe Chat Flow](#8-recipe-chat-flow)
9. [Gemini Fallback Chain](#9-gemini-fallback-chain)
10. [Data Model](#10-data-model)
11. [Authentication Flow](#11-authentication-flow)
12. [Security Model](#12-security-model)
13. [Key Design Decisions](#13-key-design-decisions)

---

## 1. Architectural Overview

ChefMind AI is a **decoupled, three-tier web application**:

| Tier | Technology | Responsibilities |
|---|---|---|
| **Presentation** | React 19 SPA (Vite) | UI and form state, voice input (Web Speech API), camera capture, client-side recipe scaling, session in `localStorage` |
| **Application** | Node.js + Express 5 (MVC) | Routing, JWT auth, CORS, security headers, tiered rate limiting, input checks, consistent error responses |
| **Data** | MongoDB Atlas via Mongoose 9 | Users and recipes, with every recipe tied to its owner |
| **AI service** | Google Gemini REST via native `fetch()` | Recipe generation, ingredient detection in photos, recipe chat, with model fallback and response checks |

---

## 2. System Architecture

```mermaid
flowchart TB
    subgraph Client ["🖥️ Client Tier — Browser"]
        direction LR
        UI["⚛️ React 19 SPA<br/>(Vite build)"]
        LS[("💾 localStorage<br/>JWT + user")]
        SP["🎙️ Web Speech API"]
        CAM["📷 MediaDevices<br/>(camera)"]
        SC["⚖️ recipeScaler.js"]
        UI <--> LS
        UI <--> SP
        UI <--> CAM
        UI <--> SC
    end

    subgraph Server ["⚙️ Application Tier — Node.js + Express 5"]
        direction TB
        SEC["🛡️ CORS + Security Headers"]
        HP["❤️ /health · /health/ready"]
        RL["🚦 Rate Limiters<br/>global · auth · AI"]
        RT["🧭 Routers<br/>auth · recipes · ingredients"]
        AU["🔑 JWT Auth"]
        UP["📎 Multer Upload<br/>(memory, ≤5×5 MB)"]
        CT["🎯 Controllers"]
        GS["🤖 geminiService.js"]
        EH["🧯 Central Error Handler"]
        SEC --> HP
        SEC --> RL --> RT --> AU --> UP --> CT
        CT --> GS
        CT -.errors.-> EH
    end

    subgraph External ["☁️ Data & AI Tier"]
        DB[("🍃 MongoDB Atlas<br/>users · recipes")]
        GM["✨ Google Gemini API<br/>v1beta generateContent"]
    end

    UI -- "HTTPS JSON / multipart<br/>Authorization: Bearer" --> SEC
    CT -- "Mongoose ODM" --> DB
    GS -- "fetch + 30 s timeout" --> GM
```

---

## 3. Frontend Component Architecture

```mermaid
flowchart TD
    main["main.jsx"] --> App["App.jsx<br/>(session, state, routing)"]
    App --> Auth["Auth.jsx<br/>login / register"]
    App --> Navbar
    App --> Hero
    App --> IS["IngredientSection<br/>text · voice · upload"]
    IS --> CCM["CameraCaptureModal"]
    App --> IC["IngredientConfirmation<br/>review & edit"]
    App --> PS["PreferencesSection<br/>meal · cuisine · diet"]
    App --> RR["RecipeResult"]
    RR --> RC["RecipeChat"]
    RR -.uses.-> SCL["utils/recipeScaler.js"]
    App --> CB["Cookbook"]
    App --> Footer

    subgraph Services ["services/"]
        AS["authService"]
        INS["ingredientService"]
        RS["recipeService"]
        CFG["apiConfig<br/>(VITE_API_BASE_URL)"]
    end

    Auth --> AS
    IS --> INS
    RR --> RS
    RC --> RS
    CB --> RS
    AS & INS & RS --> CFG
```

---

## 4. Backend Middleware Pipeline

Every request passes through these stages in order, and each stage can stop it early:

```mermaid
flowchart LR
    R([Request]) --> A[CORS +<br/>security headers]
    A -->|OPTIONS| X1([204])
    A --> B[express.json<br/>≤ 1 MB]
    B --> H{Health route?}
    H -->|yes| X2([200 / 503])
    H -->|no| C[apiLimiter<br/>300 / 15 min]
    C --> D{Route}
    D -->|/auth/login·register| E[authLimiter<br/>15 / 15 min]
    D -->|/recipes/* · /ingredients/image*| F[authenticateToken]
    F -->|AI route| G[aiLimiter<br/>25 / 15 min]
    G --> M[Multer<br/>if upload]
    E --> K[Controller]
    F --> K
    M --> K
    K --> OK([JSON response])
    K -. throw .-> EH[Error handler] --> ERR([4xx / 5xx JSON])
```

---

## 5. End-to-End User Journey

```mermaid
flowchart LR
    S([Sign in]) --> I{How to add<br/>ingredients?}
    I -->|Type| T[Text list]
    I -->|Speak| V[Voice input]
    I -->|Photo| P[Upload / camera]
    P --> VA[Gemini Vision<br/>detects items]
    T & V & VA --> RV[Review & edit<br/>ingredients]
    RV --> PR[Set meal, cuisine,<br/>diet, time, servings]
    PR --> G[Generate recipe]
    G --> RC[Recipe card]
    RC --> SC[Scale servings]
    RC --> CH[Chat / modify]
    CH -->|recipe_update| SV[Save as new recipe]
    RC --> CB[(Cookbook:<br/>favourite · delete)]
    SV --> CB
```

---

## 6. Request Lifecycle: Recipe Generation

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant FE as React Frontend
    participant RL as aiLimiter
    participant AU as Auth Middleware
    participant CT as recipeController
    participant GS as geminiService
    participant GM as Gemini API
    participant DB as MongoDB Atlas

    User->>FE: Confirm ingredients + preferences
    FE->>FE: Prevent double submit (loading state)
    FE->>AU: POST /api/recipes/generate (Bearer JWT)
    AU->>AU: Verify JWT → req.user.userId
    AU->>RL: Next
    RL->>CT: Under 25 req / 15 min
    CT->>CT: Check inputs (≤ 50 items, servings 1–20, time 5–360)
    CT->>GS: generateRecipeWithAI(options)
    GS->>GM: generateContent (gemini-3.8-flash)
    alt Overloaded / 429 / 503 / timeout
        GM-->>GS: Error
        GS->>GM: Retry with gemini-3.7-flash → gemini-3.5-flash
    end
    GM-->>GS: 200 + JSON candidate
    GS->>GS: Strip code fences, parse & check structure
    GS-->>CT: Checked recipe
    CT->>DB: Recipe.create({ ...recipe, owner: userId })
    DB-->>CT: Saved document
    CT-->>FE: 200 { recipe }
    FE-->>User: Render RecipeResult
```

---

## 7. Image Ingredient Detection Flow

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant FE as IngredientSection / Camera
    participant MU as Multer (memory)
    participant CT as ingredientController
    participant GS as geminiService
    participant GM as Gemini Vision

    User->>FE: Pick or capture 1–5 photos
    FE->>MU: POST /api/ingredients/image-analyze (multipart)
    MU->>MU: JPG/PNG/WebP only, ≤ 5 MB each
    MU->>CT: req.files (buffers)
    CT->>CT: Check file signatures, ≤ 15 MB combined
    CT->>GS: analyzeImageIngredients(images)
    GS->>GM: Inline base64 images + prompt
    GM-->>GS: { ingredients, observations, uncertainItems }
    GS->>GS: validateVisionResponse()
    GS-->>CT: Clean result
    CT-->>FE: 200 JSON
    FE-->>User: IngredientConfirmation (edit before generating)
```

> Images stay **in memory only** and are never written to disk or the database.

---

## 8. Recipe Chat Flow

The chat assistant returns one of three response types, and the UI handles each one differently:

```mermaid
flowchart TD
    Q[User message<br/>≤ 500 chars + ≤ 10 history items] --> O{Recipe source}
    O -->|recipeId / _id| OWN[Load recipe<br/>where owner = user]
    O -->|unsaved recipe object| CTX[Use object as context]
    OWN -->|not found| N404([404])
    OWN --> AI[chatAboutRecipeWithAI]
    CTX --> AI
    AI --> T{response.type}
    T -->|answer| A1[Show answer]
    T -->|clarification| A2[Show follow-up question]
    T -->|recipe_update| A3[Show changes +<br/>updated recipe preview]
    A3 -->|User accepts| SAVE[POST /api/recipes/save<br/>→ new cookbook entry]
```

---

## 9. Gemini Fallback Chain

```mermaid
flowchart LR
    REQ([AI request]) --> M1["gemini-3.8-flash<br/>(primary)"]
    M1 -->|200 OK| V
    M1 -->|429 / 503 / overloaded / 30 s timeout| M2["gemini-3.7-flash"]
    M2 -->|200 OK| V
    M2 -->|fails| M3["gemini-3.5-flash"]
    M3 -->|200 OK| V
    M3 -->|fails| E([500: all models failed])
    V[Strip code fences →<br/>parse JSON → check structure] --> OK([Return to controller])
```

- Every call uses `AbortSignal.timeout(30000)`, so no request can hang.
- The backend checks every model response before using it. Model output is never trusted unchecked.
- The API key goes in a header from the server and never reaches the browser.

---

## 10. Data Model

```mermaid
erDiagram
    USER ||--o{ RECIPE : owns
    USER {
        ObjectId _id PK
        string name
        string email UK "lowercase, unique"
        string password "bcrypt hash, select:false"
        date createdAt
        date updatedAt
    }
    RECIPE {
        ObjectId _id PK
        ObjectId owner FK "indexed"
        string title
        string description
        mixed[] ingredients "name, quantity, unit, optional"
        mixed[] instructions "step, title, description"
        string prepTime
        string cookTime
        number prepTimeMinutes
        number cookTimeMinutes
        string difficulty
        number servings
        mixed[] substitutions
        string[] dietaryNotes
        mixed nutrition
        string[] tips
        string meal
        string cuisine
        string diet
        boolean isFavorite
        date createdAt
        date updatedAt
    }
```

**Indexes:** `users.email` (unique) · `recipes.owner` · `recipes { owner: 1, createdAt: -1 }`. The compound index makes loading the newest recipes in a cookbook fast.

---

## 11. Authentication Flow

```mermaid
sequenceDiagram
    autonumber
    participant FE as Frontend
    participant API as /api/auth
    participant DB as MongoDB

    FE->>API: POST /register { name, email, password }
    API->>API: Check (name ≤ 100, password 8–128, email format)
    API->>DB: Is email taken?
    API->>DB: Create user (bcrypt hash, cost 12)
    API-->>FE: 201 Created
    FE->>API: POST /login { email, password }
    API->>DB: findOne(email).select("+password")
    API->>API: bcrypt.compare
    API-->>FE: 200 { token (1 h), user }
    FE->>FE: Store in localStorage
    Note over FE: On reload: GET /api/auth/me<br/>restores the session or logs out on 401
```

---

## 12. Security Model

```mermaid
flowchart TB
    subgraph L1 ["Layer 1 — Transport & Browser"]
        a1[HSTS in production] --- a2[CORS allow-list] --- a3[X-Frame-Options DENY<br/>nosniff · COOP · Referrer-Policy]
    end
    subgraph L2 ["Layer 2 — Abuse Prevention"]
        b1[Global 300 / 15 min] --- b2[Auth 15 / 15 min] --- b3[AI 25 / 15 min]
    end
    subgraph L3 ["Layer 3 — Input Checks"]
        c1[JSON ≤ 1 MB] --- c2[Image type + signature checks] --- c3[Field length & range limits]
    end
    subgraph L4 ["Layer 4 — Identity & Access"]
        d1[JWT, 1 h expiry] --- d2[bcrypt cost 12] --- d3[Every query scoped to owner]
    end
    subgraph L5 ["Layer 5 — Secrets & Errors"]
        e1[Env checked at startup] --- e2[Secrets server-side only] --- e3[No stack traces in responses]
    end
    L1 --> L2 --> L3 --> L4 --> L5
```

| Threat | Mitigation |
|---|---|
| Brute-force login | `authLimiter` (15 / 15 min / IP) and the same error message for unknown email and wrong password |
| Password-hash CPU DoS | Passwords are capped at 128 characters |
| Reading another user's recipes | Every query includes `owner: req.user.userId`, and someone else's recipe returns **404** |
| AI cost abuse | AI routes need login and are limited to 25 requests per 15 min per IP |
| Spoofed uploads | MIME allow-list plus a check of the file's real signature bytes, with size and count limits |
| Leaked secrets | The Gemini key, JWT secret and DB URI stay in backend env vars only, and `.env` is git-ignored |
| Weak production secret | Startup fails if `JWT_SECRET` is shorter than 32 characters in production |

---

## 13. Key Design Decisions

| Decision | Why | Trade-off |
|---|---|---|
| **Native `fetch()` instead of the Gemini SDK** | No extra dependency, full control over timeouts and fallback | Responses have to be parsed by hand |
| **Generated recipes saved automatically** | A recipe is never lost, and the cookbook always matches what the user saw | Unwanted recipes have to be deleted |
| **Serving scaler runs in the browser** | Instant, free and works offline, with no AI call | Uses heuristics, e.g. salt and spices scale non-linearly |
| **Ingredient review step** | The user checks AI-detected items before generating, so hallucinated ingredients don't reach the recipe | One extra click |
| **Chat edits saved as *new* recipes** | The original recipe is never overwritten | The cookbook can contain several versions of a dish |
| **JWT in `localStorage`** | Simple and works across separately hosted frontend and backend | Readable by any XSS. Moving to `HttpOnly` cookies is on the roadmap. |
| **Separate frontend and backend hosting** | Each part can scale on its own, and the frontend can be served from a CDN | Needs CORS and `VITE_API_BASE_URL` configuration |
