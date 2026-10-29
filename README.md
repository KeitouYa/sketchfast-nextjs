# Sketchfast — AI Image Generation & Chat (Next.js full-stack)

A full-stack SaaS application for AI image generation and chat, with a pay-per-use credit system, built on Next.js 15 with a layered backend.

![Next.js](https://img.shields.io/badge/Next.js-15.5.7-black?logo=next.js)
![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue?logo=typescript)
![Mongoose](https://img.shields.io/badge/Mongoose-8-green?logo=mongodb)
![React](https://img.shields.io/badge/React-19-61DAFB?logo=react)

> This is the full-stack Next.js version: pages and a TypeScript backend in one app, with PayPal for credits. The version with a Spring Boot backend and Stripe is in [sketchfast](https://github.com/KeitouYa/sketchfast).

## Technical Highlights

### Architecture & Design Patterns

- **Layered Backend**: `Route / Server Action → Service → Repository → Model`. Chat goes through an API route and a controller that validates input; image generation and credits are Server Actions.
- **Factory Pattern**: AI providers (Gemini, OpenAI) sit behind one interface and are picked at runtime
- **Atomic Operations**: credits are deducted with a single conditional MongoDB update (`credits: { $gte: cost }`), so concurrent requests can't overspend
- **Request Tracing**: chat requests carry a `requestId` (nanoid) from the route through controller, service and repository logs

### Performance & Reliability

- **Multi-Provider Failover**: Gemini → OpenAI fallback, each call with a 30s timeout
- **Refund on Failure**: if image generation fails after the credit was taken, or every chat provider fails, the credit is refunded
- **Redis Caching**: chat replies are cached for 24 hours in Upstash Redis
- **Timeouts**: 30s per AI chat call, 120s for image generation; a prediction that times out is cancelled
- **Rate Limiting**: per-user daily cap on image generation (20/day, Upstash fixed window)

### Security

- **Server-side Payment Verification**: PayPal orders are priced and captured on the server (Orders v2 API); the browser only picks a package. A unique order id plus a `created → completed` status guard, applied in one MongoDB transaction with the credit update, means an order is credited exactly once
- **Clerk Authentication**: pages and API routes are protected in middleware
- **Role-Based Access**: a role hierarchy (ADMIN > SUBSCRIBER > USER) is checked inside admin API routes; admin features are API-only
- **Input Validation**: Zod schemas for chat messages and image prompts

## Tech Stack

| Category | Technologies |
|----------|-------------|
| Frontend | Next.js 15 (App Router), React 19, TypeScript 5 |
| Styling | Tailwind CSS 4, Radix UI, shadcn/ui |
| Backend | Next.js API Routes and Server Actions (Node.js) |
| Database | MongoDB (Atlas), Mongoose 8 |
| Caching / Rate Limiting | Upstash Redis |
| Auth | Clerk |
| AI Services | Google Gemini, OpenAI (fallback), Replicate (FLUX Schnell) |
| Storage | Cloudinary |
| Payment | PayPal (Orders v2 REST API) |
| Testing | Jest |

## Architecture

```
Client (React)
  │
  ├─ fetch ─────────────► API Routes /api/chat, /api/paypal/* ─► controllers (Zod)
  │                                                     │
  └─ Server Actions ────► image.service / payment.service
     Server Components                                  │
                                                        ▼
                                  Service layer: credits, rate limit,
                                  provider fallback, refunds, caching
                                                        │
              ┌─────────────────────────────────────────┼──────────────────────┐
              ▼                                         ▼                      ▼
   Repositories → MongoDB                      Upstash Redis          Gemini · OpenAI ·
   (users, credits, images, costs)             (cache, rate limit)    Replicate · Cloudinary
```

## Key Implementation Details

### AI Provider Failover

```typescript
// src/services/chat.service.ts (simplified)
for (const provider of providers) {            // e.g. ["gemini", "openai"]
  try {
    const ai = await getAIProvider(provider);
    const reply = await withTimeout(ai.chat(message), TIMEOUTS.AI_CHAT, `AI provider ${provider} timeout`);
    return { reply, providerUsed: provider, fallbackUsed: provider !== primary };
  } catch (err) {
    if (provider === providers[providers.length - 1]) {
      throw new ProviderError("All AI providers failed");
    }
  }
}
```

### Atomic Credit Deduction and Refund

```typescript
// src/repositories/credits.repository.ts
// Only matches when the balance covers the cost, so two concurrent requests can't both spend the last credit
const result = await Credit.findOneAndUpdate(
  { userEmail, credits: { $gte: cost } },
  { $inc: { credits: -cost } },
  { new: true }
).lean();

// src/services/image.service.ts
// If anything after the deduction fails (Replicate, download, upload, save), the credit is given back
if (charged) {
  await creditRepository.refundCredits(userEmail, IMAGE_GENERATION_CREDIT_COST, requestId);
}
```

### Crediting a Purchase Exactly Once

```typescript
// src/repositories/payment.repository.ts
// The CREATED → COMPLETED guard makes a repeated capture of the same order a no-op
await session.withTransaction(async () => {
  const payment = await Payment.findOneAndUpdate(
    { orderId, status: PaymentStatus.CREATED },
    { $set: { status: PaymentStatus.COMPLETED, captureId, completedAt: new Date() } },
    { new: true, session }
  );
  if (!payment) return; // already credited

  await Credit.findOneAndUpdate(
    { userEmail: payment.userEmail },
    { $inc: { credits: payment.credits, amount: Number(payment.amount) } },
    { upsert: true, new: true, session }
  );
});
```

### Image Generation with a Timeout

```typescript
// src/services/image.service.ts
const prediction = await replicate.predictions.create({ model: "black-forest-labs/flux-schnell", input });
try {
  finished = await withTimeout(replicate.wait(prediction), TIMEOUTS.IMAGE_GENERATION, "Replicate prediction timeout");
} catch (err) {
  await replicate.predictions.cancel(prediction.id).catch(() => {}); // stop paying for a stuck prediction
  throw err;
}
```

### Request Tracing

```typescript
// app/api/chat/route.ts
const requestId = req.headers.get("x-request-id") || nanoid();
logger.info("➡️ POST /api/chat", { requestId });
// passed on to the controller, service and repository, so every log line of the request carries it
```

## Project Structure

```
├── app/                    # Next.js App Router
│   ├── api/
│   │   ├── chat/           # AI chat endpoint
│   │   ├── paypal/         # Server-side PayPal order creation and capture
│   │   └── admin/          # Admin-only stats and user list
│   ├── chat/               # Chat page
│   ├── dashboard/          # The user's images
│   ├── image/[_id]/        # Public share page
│   └── buy-credits/        # Credit purchase (PayPal)
│
├── src/
│   ├── controllers/        # Input validation and response shape (chat)
│   ├── services/           # Business logic; image and payment services are Server Actions
│   ├── repositories/       # Data access, atomic credit updates
│   ├── models/             # Mongoose schemas
│   ├── providers/          # AI providers (Gemini, OpenAI) and the MongoDB connection
│   ├── middlewares/        # Rate limiting, RBAC
│   ├── schemas/            # Zod schemas
│   ├── lib/                # Logger, cache, errors, timeouts
│   └── __tests__/          # Jest tests
│
├── context/                # React contexts (credits, image generation, PayPal)
├── hooks/                  # React hooks
└── components/             # UI, layout, form and image components
```

## Getting Started

### Prerequisites

- Node.js 18+
- A MongoDB database (e.g. MongoDB Atlas) and an Upstash Redis database
- API keys for: Clerk, Google Gemini, OpenAI, Replicate, Cloudinary
- A PayPal client id, for buying credits

### Installation

```bash
git clone https://github.com/KeitouYa/sketchfast-nextjs.git
cd sketchfast-nextjs
npm install --legacy-peer-deps
```

### Environment Variables

Copy `.env.example` to `.env.local` and fill in:

```env
# Database
DATABASE_URL=mongodb+srv://...

# Authentication
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_...
CLERK_SECRET_KEY=sk_...

# AI Services
GEMINI_API_KEY=...
OPENAI_API_KEY=...
REPLICATE_API_TOKEN=...

# Storage
CLOUDINARY_CLOUD_NAME=...
CLOUDINARY_API_KEY=...
CLOUDINARY_API_SECRET=...

# Cache and rate limiting
UPSTASH_REDIS_REST_URL=...
UPSTASH_REDIS_REST_TOKEN=...

# Payment
NEXT_PUBLIC_PAYPAL_CLIENT_ID=...
PAYPAL_CLIENT_SECRET=...
PAYPAL_ENV=sandbox   # or live; must match the client id and secret
```

### Run Development Server

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000)

### Run Tests

```bash
npm test
```

Jest unit tests cover the chat and image services (credit deduction, refunds on failure, provider fallback, prompt validation, the daily rate limit) and the PayPal purchase flow (server-side pricing, order ownership, repeated and pending captures, amount mismatch).

## Features

- **AI Chat**: conversations with Gemini, falling back to OpenAI, with cached replies
- **Image Generation**: text-to-image with Replicate's FLUX Schnell, stored in Cloudinary
- **Credit System**: new users get 20 free credits; each image or chat reply costs 1 credit and is refunded if generation fails
- **Credit Purchase**: packs of 10, 30 or 60 credits via PayPal; the server creates and captures the order and credits the account once
- **User Dashboard**: browse your generated images, download them, or share a public link
- **Admin API**: platform stats and user list, for the admin role only

## Roadmap

- Comic storyboard generation: one character kept consistent across a four-panel page
