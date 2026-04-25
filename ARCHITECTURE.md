# App Architecture & API Connections

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      USER BROWSER                            │
│                      (port 3000)                             │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│              NEXT.JS FRONTEND                                │
│  • Recipe pages                                              │
│  • Pantry scanner                                            │
│  • User dashboard                                            │
└──────────┬────────────────┬──────────────┬──────────────┬───┘
           │                │              │              │
     ┌─────▼─────┐   ┌─────▼────────┐ ┌──▼───┐   ┌─────▼─────┐
     │  CLERK    │   │ GOOGLE       │ │ARCJET│   │ UNSPLASH  │
     │ (Auth)    │   │ GEMINI       │ │      │   │ (Images)  │
     │           │   │ (AI)         │ │      │   │           │
     │ ✅ Users  │   │ ✅ Recipes   │ │✅Bot │   │ ✅ Photos │
     │ ✅ Login  │   │ ✅ Scanning  │ │detection  │ ✅ Search │
     │ ✅ Signup │   │ ✅ Ideas     │ │✅Rate │   │           │
     └─────┬─────┘   └──────┬───────┘ │limiting  └─────┬─────┘
           │                │         └─────┬────┘     │
           │                │               │         │
           └────────────────┼───────────────┼─────────┘
                            │               │
                   ┌────────▼───────┐       │
                   │ STRAPI API     │       │
                   │ (Backend)      │◄──────┘
                   │ (port 1337)    │
                   │                │
                   │ • Users data   │
                   │ • Recipes      │
                   │ • Pantry items │
                   │ • Saved recipes│
                   └────────┬───────┘
                            │
                   ┌────────▼─────────┐
                   │   NEON DB        │
                   │ PostgreSQL       │
                   │                  │
                   │ • User profiles  │
                   │ • Recipe data    │
                   │ • Pantry scans   │
                   │ • Preferences    │
                   └──────────────────┘
```

---

## 📡 API Data Flow

### Flow 1: User Authentication
```
User → Clerk Login Form → Clerk Auth API → Frontend Stores Token → Logged In ✅
```

### Flow 2: Scan Pantry
```
1. User takes photo
   ↓
2. Frontend sends to Gemini API (with GEMINI_API_KEY)
   ↓
3. Gemini analyzes image → Lists ingredients
   ↓
4. Frontend saves to Strapi API (with STRAPI_API_TOKEN)
   ↓
5. Strapi saves to PostgreSQL database
   ↓
6. User sees pantry items ✅
```

### Flow 3: Get Recipe Ideas
```
1. Frontend requests recipes with pantry items
   ↓
2. Frontend sends to Gemini API
   ↓
3. Gemini generates recipes based on ingredients
   ↓
4. Frontend fetches images from Unsplash API
   ↓
5. Frontend saves recipes to Strapi API
   ↓
6. User sees recipe ideas ✅
```

### Flow 4: Security Check
```
Every API request → Arcjet checks request
                    ↓
                    Is it legitimate? → Rate limit user (if free tier)
                    ↓
                    Block if bot detected or limit exceeded
```

---

## 🔐 Security Layers

```
User Request
    ↓
ARCJET (Shield WAF + Bot Detection)
    ↓ Passes?
CLERK (Authentication)
    ↓ Valid user?
Request Handler
    ↓
Internal Services (Gemini, Strapi, etc.)
    ↓
Database
```

---

## 📊 Environment Variables by Layer

### Layer 1: Authentication
```
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY ─→ Public (browser)
CLERK_SECRET_KEY ─────────────────→ Secret (server)
```

### Layer 2: Security
```
ARCJET_KEY ──────────────────────→ Public (middleware)
```

### Layer 3: Backend API
```
NEXT_PUBLIC_STRAPI_URL ──────────→ Public (browser)
STRAPI_API_TOKEN ────────────────→ Secret (server)
```

### Layer 4: External APIs
```
GEMINI_API_KEY ───────────────────→ Secret (server action)
UNSPLASH_ACCESS_KEY ──────────────→ Secret (server action)
```

### Layer 5: Database
```
DATABASE_URL ─────────────────────→ Secret (backend only)
```

---

## 🔌 API Endpoints Used

### Clerk
```
POST   /api/auth/callback/clerk       (OAuth callback)
GET    /api/user/profile              (Get user info)
```

### Strapi
```
GET    /api/users?filters[clerkId][$eq]=USER_ID
POST   /api/users                      (Create user)
PUT    /api/users/{id}                 (Update user)
GET    /api/recipes                    (Get recipes)
POST   /api/recipes                    (Create recipe)
GET    /api/pantry-items               (Get pantry)
POST   /api/pantry-items               (Add item)
```

### Google Gemini
```
POST   /v1beta/models/gemini-2.5-flash-lite:generateContent
       (Generate recipes/analyze images)
```

### Unsplash
```
GET    /api/search/photos?query=RECIPE_NAME
       (Search recipe images)
```

### Arcjet
```
POST   /api/arcjet.protect()           (Internal JS library)
```

---

## 🔄 Request Lifecycle Example: "Scan My Pantry"

```
┌─ USER CLICKS SCAN BUTTON ──────────────────────────────┐
│                                                         │
│  1. Photo uploaded to frontend                          │
│  │                                                      │
│  2. Arcjet checks: "Is this a real user?"              │
│     ARCJET_KEY → arcjet.protect()                      │
│     ✅ Pass                                            │
│  │                                                      │
│  3. Frontend authenticates user                         │
│     NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY + CLERK_SECRET   │
│     ✅ Valid user                                      │
│  │                                                      │
│  4. Check user tier (Free or Pro)                      │
│     via Clerk user metadata                            │
│     ✅ Has scan quota                                  │
│  │                                                      │
│  5. Send image to Gemini API                           │
│     GEMINI_API_KEY → gemini-2.5-flash-lite             │
│     Request: "What's in this image?"                   │
│     Response: ["tomato", "lettuce", "cheese"]          │
│  │                                                      │
│  6. Save to Strapi database                            │
│     STRAPI_API_TOKEN → POST /api/pantry-items          │
│     Save ingredients with user ID                      │
│  │                                                      │
│  7. Strapi saves to PostgreSQL                         │
│     DATABASE_URL → INSERT pantry record                │
│  │                                                      │
│  8. Return to user                                      │
│     ✅ "Scanned 3 items"                              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🚀 Environment-Specific Setup

### Development (Your machine)
```
Frontend:
  NEXT_PUBLIC_STRAPI_URL = http://localhost:1337    (local)
  GEMINI_API_KEY = Free tier key                     (limited)
  UNSPLASH_ACCESS_KEY = Free tier key                (limited)
  
Backend:
  DATABASE_URL = Local PostgreSQL or Neon dev        (non-prod)
  JWT secrets = Generated values                      (temporary)
```

### Production (Deployed)
```
Frontend:
  NEXT_PUBLIC_STRAPI_URL = https://api.yourapp.com  (production)
  GEMINI_API_KEY = Production key                    (billing enabled)
  UNSPLASH_ACCESS_KEY = Production key               (quota managed)
  
Backend:
  DATABASE_URL = Production PostgreSQL              (backed up)
  JWT secrets = Secure, rotated regularly           (long-term)
  All keys stored in secrets manager                 (not in .env)
```

---

## ⚡ Performance Considerations

### Caching
- Recipe generation: Cached for 1 hour
- Pantry scans: Stored in database
- User data: Clerk handles caching

### Rate Limiting (Arcjet)
- Free tier: 10 pantry scans/month, 5 recommendations/month
- Pro tier: 1000 requests/day

### Database
- PostgreSQL (Neon) chosen for reliability
- Connection pooling configured
- Indexes on frequently queried fields

---

## 🔍 Debugging Tools

### Check if services are running:
```bash
# Frontend
curl http://localhost:3000

# Backend
curl http://localhost:1337/admin

# Database (Neon)
psql postgresql://user:pass@host/db -c "SELECT 1"
```

### Check API keys are loaded:
```bash
# Frontend
echo $NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY

# View logs:
npm run dev  # Shows errors

# Check browser console:
F12 → Console tab → See errors
```

### Monitor usage:
- Clerk: https://dashboard.clerk.com → Usage
- Gemini: Google Cloud Console → Quotas
- Unsplash: https://unsplash.com/account/applications → Usage
- Arcjet: https://arcjet.com → Analytics

