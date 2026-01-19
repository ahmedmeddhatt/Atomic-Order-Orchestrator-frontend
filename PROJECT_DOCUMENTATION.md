# Atomic Order Orchestrator - Complete Project Documentation

## 📋 Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Technology Stack](#technology-stack)
4. [Project Structure](#project-structure)
5. [Getting Started](#getting-started)
6. [Backend Documentation](#backend-documentation)
7. [Frontend Documentation](#frontend-documentation)
8. [API Reference](#api-reference)
9. [WebSocket Events](#websocket-events)
10. [Database Schema](#database-schema)
11. [Development Workflow](#development-workflow)
12. [Testing](#testing)
13. [Deployment](#deployment)
14. [Troubleshooting](#troubleshooting)

---

## 📖 Project Overview

**Atomic Order Orchestrator** is a high-performance, real-time logistics dashboard designed to handle 5,000+ active shipments without lag. The system integrates with Shopify webhooks to process order updates and provides a real-time dashboard for monitoring and managing orders.

### Key Features

- ✅ **Real-Time Updates**: WebSocket-based live synchronization
- ✅ **High Performance**: Virtualized table rendering for 5,000+ orders
- ✅ **Optimistic Locking**: Version-based conflict resolution
- ✅ **Shopify Integration**: Webhook processing for order events
- ✅ **Scalable Architecture**: BullMQ queue processing with Redis
- ✅ **Type-Safe**: Full TypeScript implementation
- ✅ **Modern UI**: Next.js 16 with TailwindCSS 4

### Use Cases

1. **E-commerce Order Management**: Track and manage Shopify orders in real-time
2. **Logistics Monitoring**: Monitor shipment statuses across multiple orders
3. **Conflict Resolution**: Handle concurrent order updates with merge conflict detection
4. **Performance Testing**: Stress-test with 5,000+ concurrent orders

---

## 🏗️ Architecture

### System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         SHOPIFY PLATFORM                        │
│                    (External Order Source)                      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ Webhooks (orders/create, orders/update)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    BACKEND (Port 9000)                          │
│                    NestJS + TypeScript                          │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Orders Controller                           │  │
│  │  - GET /orders (Fetch all orders)                        │  │
│  │  - POST /webhooks/shopify (Receive webhooks)             │  │
│  └────────────────┬─────────────────────────────────────────┘  │
│                   │                                             │
│  ┌────────────────▼─────────────────────────────────────────┐  │
│  │              Orders Service                              │  │
│  │  - Business Logic                                        │  │
│  │  - Shipping Fee Calculation                              │  │
│  │  - Order Status Mapping                                  │  │
│  └────────────────┬─────────────────────────────────────────┘  │
│                   │                                             │
│  ┌────────────────▼─────────────────────────────────────────┐  │
│  │              BullMQ Queue (Redis)                        │  │
│  │  - Job: process-order                                    │  │
│  │  - Async Processing                                      │  │
│  │  - Retry Logic                                           │  │
│  └────────────────┬─────────────────────────────────────────┘  │
│                   │                                             │
│  ┌────────────────▼─────────────────────────────────────────┐  │
│  │              Orders Processor                            │  │
│  │  - Save to PostgreSQL                                    │  │
│  │  - Emit WebSocket Events                                 │  │
│  └────────────────┬─────────────────────────────────────────┘  │
│                   │                                             │
│  ┌────────────────▼─────────────────────────────────────────┐  │
│  │              PostgreSQL Database                         │  │
│  │  - Orders Table                                          │  │
│  │  - Optimistic Locking (version column)                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Sync Gateway (Socket.IO)                    │  │
│  │  - Namespace: /sync                                      │  │
│  │  - Event: ORDER_SYNCED                                   │  │
│  └────────────────┬─────────────────────────────────────────┘  │
└───────────────────┼─────────────────────────────────────────────┘
                    │
                    │ WebSocket Connection
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    FRONTEND (Port 3000/5000)                    │
│                    Next.js 16 + React 19                        │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              React Query Cache                           │  │
│  │  - Query Key: ['orders']                                 │  │
│  │  - Automatic Refetching                                  │  │
│  │  - Optimistic Updates                                    │  │
│  └────────────────┬─────────────────────────────────────────┘  │
│                   │                                             │
│  ┌────────────────▼─────────────────────────────────────────┐  │
│  │              useOrders Hook                              │  │
│  │  - Fetches: GET /orders                                  │  │
│  │  - Maps backend data to frontend format                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              useOrderSync Hook                           │  │
│  │  - Connects to: /sync namespace                          │  │
│  │  - Listens: ORDER_SYNCED events                          │  │
│  │  - Updates React Query cache surgically                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Dashboard Page                              │  │
│  │  - KPI Cards (Total Orders, Active Shipments)            │  │
│  │  - Socket Connection Status Badge                        │  │
│  │  - Virtualized Orders Table (5,000+ rows)                │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Edit Order Modal                            │  │
│  │  - Form with Status, Tracking, Notes                     │  │
│  │  - Conflict Detection (version-based)                    │  │
│  │  - Merge Conflict Resolution UI                          │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow

1. **Shopify Webhook** → Backend receives order event
2. **Backend Processing** → Validates, calculates shipping, queues job
3. **BullMQ Queue** → Processes job asynchronously
4. **Database** → Saves order with version number
5. **WebSocket** → Emits ORDER_SYNCED event
6. **Frontend** → Receives event, updates React Query cache
7. **UI** → Re-renders affected components

---

## 🛠️ Technology Stack

### Backend

| Technology | Version | Purpose |
|------------|---------|---------|
| **NestJS** | 10.x | Backend framework |
| **TypeScript** | 5.x | Type-safe development |
| **PostgreSQL** | Latest | Primary database |
| **TypeORM** | Latest | ORM for database operations |
| **BullMQ** | Latest | Job queue processing |
| **Redis** | Latest | Queue storage & caching |
| **Socket.IO** | 4.x | WebSocket communication |
| **Class Validator** | Latest | DTO validation |
| **Class Transformer** | Latest | Data transformation |

### Frontend

| Technology | Version | Purpose |
|------------|---------|---------|
| **Next.js** | 16.1.2 | React framework |
| **React** | 19.2.3 | UI library |
| **TypeScript** | 5.x | Type-safe development |
| **TanStack Query** | 5.90.17 | Data fetching & caching |
| **TanStack Virtual** | 3.13.18 | Virtualized rendering |
| **Socket.IO Client** | 4.8.3 | WebSocket client |
| **TailwindCSS** | 4.x | Styling framework |
| **Lucide React** | 0.562.0 | Icon library |
| **CVA** | 0.7.1 | Component variants |

### DevOps

| Technology | Purpose |
|------------|---------|
| **Docker** | Containerization |
| **Docker Compose** | Multi-container orchestration |
| **npm workspaces** | Monorepo management |
| **Concurrently** | Run multiple dev servers |

---

## 📁 Project Structure

```
fincart-monorepo/
├── fincart-backend/              # NestJS Backend
│   ├── src/
│   │   ├── orders/               # Orders Module
│   │   │   ├── dto/              # Data Transfer Objects
│   │   │   ├── entities/         # TypeORM Entities
│   │   │   ├── orders.controller.ts
│   │   │   ├── orders.service.ts
│   │   │   ├── orders.processor.ts
│   │   │   └── orders.module.ts
│   │   ├── sync/                 # WebSocket Module
│   │   │   ├── sync.gateway.ts
│   │   │   └── sync.module.ts
│   │   ├── app.module.ts
│   │   └── main.ts
│   ├── .env                      # Environment variables
│   ├── docker-compose.yml        # PostgreSQL + Redis
│   ├── package.json
│   └── tsconfig.json
│
├── fincart-frontend/             # Next.js Frontend
│   ├── app/                      # Next.js App Router
│   │   ├── dashboard/
│   │   │   └── page.tsx          # Main dashboard page
│   │   ├── api/
│   │   │   └── orders/
│   │   │       └── route.ts      # Mock API (for testing)
│   │   ├── layout.tsx            # Root layout
│   │   ├── page.tsx              # Home page
│   │   └── globals.css           # Global styles
│   ├── components/               # React Components
│   │   ├── orders-table.tsx      # Virtualized table
│   │   ├── edit-order-modal.tsx  # Edit modal with conflict resolution
│   │   └── ui/                   # UI Components
│   │       ├── status-badge.tsx
│   │       └── skeleton.tsx
│   ├── hooks/                    # Custom React Hooks
│   │   ├── useOrders.ts          # Data fetching
│   │   ├── useOrderSync.ts       # WebSocket sync
│   │   └── useSocket.ts          # Socket.IO wrapper
│   ├── providers/                # React Context Providers
│   │   └── query-provider.tsx    # React Query setup
│   ├── types/                    # TypeScript Types
│   │   └── order.ts              # Order interfaces
│   ├── lib/                      # Utilities
│   │   └── utils.ts              # Helper functions
│   ├── .env.local                # Environment variables
│   ├── package.json
│   ├── INTEGRATION_SUMMARY.md    # Integration docs
│   └── TESTING_GUIDE.md          # Testing procedures
│
├── test-webhook.js               # Webhook testing script
├── package.json                  # Monorepo scripts
└── PROJECT_DOCUMENTATION.md      # This file
```

---

## 🚀 Getting Started

### Prerequisites

- **Node.js**: v18+ or v20+
- **npm**: v9+ or v10+
- **Docker**: Latest (for PostgreSQL & Redis)
- **Git**: Latest

### Installation

#### 1. Clone the Repository

```bash
git clone <repository-url>
cd fincart-monorepo
```

#### 2. Install Dependencies

```bash
# Install all dependencies (backend + frontend)
npm install
```

#### 3. Setup Backend

```bash
cd fincart-backend

# Copy environment variables
cp .env.example .env

# Edit .env with your configuration
# DATABASE_URL, REDIS_HOST, etc.

# Start PostgreSQL and Redis
docker-compose up -d

# Run database migrations (if any)
npm run migration:run
```

#### 4. Setup Frontend

```bash
cd ../fincart-frontend

# Environment variables are already configured
# NEXT_PUBLIC_API_URL=http://localhost:9000
# NEXT_PUBLIC_WS_URL=http://localhost:9000
```

#### 5. Start Development Servers

**Option A: Run Both Servers Simultaneously**
```bash
# From root directory
npm run dev
```

**Option B: Run Separately**
```bash
# Terminal 1 - Backend
cd fincart-backend
npm run start:dev

# Terminal 2 - Frontend
cd fincart-frontend
npm run dev
```

#### 6. Access the Application

- **Frontend Dashboard**: http://localhost:3000/dashboard
- **Backend API**: http://localhost:9000
- **API Documentation**: http://localhost:9000/api (if Swagger is configured)

---

## 🔧 Backend Documentation

### Environment Variables

```env
# Database
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USER=postgres
DATABASE_PASSWORD=postgres
DATABASE_NAME=fincart_db

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# Application
PORT=9000
NODE_ENV=development

# Shopify (optional)
SHOPIFY_WEBHOOK_SECRET=your_secret_here
```

### Key Modules

#### Orders Module

**Purpose**: Handle order creation, updates, and retrieval

**Files**:
- `orders.controller.ts` - HTTP endpoints
- `orders.service.ts` - Business logic
- `orders.processor.ts` - Queue job processing
- `orders.entity.ts` - Database entity

**Endpoints**:
- `GET /orders` - Fetch all orders
- `POST /webhooks/shopify` - Receive Shopify webhooks

#### Sync Module

**Purpose**: Real-time WebSocket communication

**Files**:
- `sync.gateway.ts` - Socket.IO gateway

**Events**:
- `ORDER_SYNCED` - Emitted when order is created/updated

### Database Schema

```sql
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  shopifyOrderId VARCHAR(255) NOT NULL UNIQUE,
  status VARCHAR(50) NOT NULL,
  shippingFee DECIMAL(10, 2) NOT NULL,
  lastExternalUpdatedAt TIMESTAMP NOT NULL,
  version INTEGER NOT NULL DEFAULT 1,
  createdAt TIMESTAMP DEFAULT NOW(),
  updatedAt TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_orders_shopify_id ON orders(shopifyOrderId);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_updated_at ON orders(updatedAt DESC);
```

### Shipping Fee Calculation

```typescript
calculateShippingFee(totalPrice: number): number {
  if (totalPrice < 50) return 5.99;
  if (totalPrice < 100) return 9.99;
  if (totalPrice < 200) return 14.99;
  return 19.99;
}
```

### Status Mapping

| Shopify Status | Backend Status |
|---------------|----------------|
| `financial_status: 'paid'` | `CONFIRMED` |
| `financial_status: 'pending'` | `PENDING` |
| `fulfillment_status: 'fulfilled'` | `SHIPPED` |
| `financial_status: 'refunded'` | `CANCELLED` |

---

## 💻 Frontend Documentation

### Environment Variables

```env
NEXT_PUBLIC_API_URL=http://localhost:9000
NEXT_PUBLIC_WS_URL=http://localhost:9000
```

### Key Components

#### Dashboard Page (`app/dashboard/page.tsx`)

**Features**:
- KPI cards (Total Orders, Active Shipments)
- Socket connection status indicator
- Real-time orders table
- Responsive layout

#### Orders Table (`components/orders-table.tsx`)

**Features**:
- Virtualized rendering (handles 5,000+ rows)
- Sortable columns
- Row striping for readability
- Edit button per row

**Performance**:
- Uses `@tanstack/react-virtual`
- Only renders visible rows
- Smooth scrolling at 60 FPS

#### Edit Order Modal (`components/edit-order-modal.tsx`)

**Features**:
- Edit status, tracking number, notes
- Real-time conflict detection
- Merge conflict resolution UI
- Version-based optimistic locking

**Conflict Resolution**:
1. User opens modal and makes changes
2. Server updates the same order
3. Modal detects version mismatch
4. Shows conflict alert with options:
   - **Refresh**: Accept server changes
   - **Force Overwrite**: Keep user changes

### Custom Hooks

#### useOrders

**Purpose**: Fetch and manage orders data

```typescript
const { orders, isLoading, isError, updateOrder } = useOrders();
```

**Features**:
- React Query integration
- Automatic refetching
- Optimistic updates
- Error handling

#### useOrderSync

**Purpose**: Real-time WebSocket synchronization

```typescript
const { isConnected } = useOrderSync();
```

**Features**:
- Connects to `/sync` namespace
- Listens for `ORDER_SYNCED` events
- Updates React Query cache surgically
- Version-based conflict prevention

#### useSocket

**Purpose**: Socket.IO wrapper

```typescript
const { socket, isConnected, on, emit } = useSocket(url);
```

**Features**:
- Auto-reconnection
- Type-safe event handlers
- Cleanup on unmount

### Type Definitions

```typescript
export enum OrderStatus {
  PENDING = 'PENDING',
  CONFIRMED = 'CONFIRMED',
  SHIPPED = 'SHIPPED',
  CANCELLED = 'CANCELLED',
}

export interface Order {
  id: string;
  shopifyOrderId: string;
  version: number;
  status: OrderStatus;
  shippingFee: number;
  createdAt: string;
  updatedAt: string;
  
  // Optional display fields
  customer?: Customer;
  items?: OrderItem[];
  total?: number;
  trackingNumber?: string;
  notes?: string;
}
```

---

## 📡 API Reference

### GET /orders

Fetch all orders from the database.

**Request**:
```http
GET http://localhost:9000/orders
```

**Response**: `200 OK`
```json
[
  {
    "id": "ecd6cfa5-b123-4567-8901-234567890abc",
    "shopifyOrderId": "123457343",
    "status": "CONFIRMED",
    "shippingFee": "5.99",
    "lastExternalUpdatedAt": "2026-01-15T18:30:00.000Z",
    "version": 1,
    "createdAt": "2026-01-15T18:25:00.000Z",
    "updatedAt": "2026-01-15T18:30:00.000Z"
  }
]
```

### POST /webhooks/shopify

Receive Shopify webhook events.

**Request**:
```http
POST http://localhost:9000/webhooks/shopify
Content-Type: application/json
x-shopify-webhook-id: webhook-1234567890
x-shopify-topic: orders/create

{
  "id": "123456789",
  "updated_at": "2026-01-17T18:30:00Z",
  "financial_status": "paid",
  "fulfillment_status": null,
  "total_price": "150.00"
}
```

**Response**: `200 OK`
```json
{
  "status": "accepted",
  "webhookId": "webhook-1234567890"
}
```

---

## 🔌 WebSocket Events

### Connection

**URL**: `http://localhost:9000/sync`

**Configuration**:
```typescript
const socket = io('http://localhost:9000/sync', {
  transports: ['websocket', 'polling'],
  reconnection: true,
  reconnectionAttempts: 5,
  reconnectionDelay: 1000,
});
```

### ORDER_SYNCED Event

**Direction**: Server → Client

**Payload**:
```typescript
{
  id: string;
  shopifyOrderId: string;
  status: OrderStatus;
  shippingFee: number;
  updatedAt: string;
  version: number;
}
```

**Example**:
```json
{
  "id": "ecd6cfa5-b123-4567-8901-234567890abc",
  "shopifyOrderId": "123457343",
  "status": "CONFIRMED",
  "shippingFee": 5.99,
  "updatedAt": "2026-01-17T18:30:00.000Z",
  "version": 2
}
```

---

## 🗄️ Database Schema

### Orders Table

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | Internal order ID |
| `shopifyOrderId` | VARCHAR(255) | UNIQUE, NOT NULL | Shopify order ID |
| `status` | VARCHAR(50) | NOT NULL | Order status |
| `shippingFee` | DECIMAL(10,2) | NOT NULL | Calculated shipping fee |
| `lastExternalUpdatedAt` | TIMESTAMP | NOT NULL | Last Shopify update |
| `version` | INTEGER | NOT NULL, DEFAULT 1 | Optimistic locking version |
| `createdAt` | TIMESTAMP | DEFAULT NOW() | Record creation time |
| `updatedAt` | TIMESTAMP | DEFAULT NOW() | Last modification time |

### Indexes

```sql
CREATE INDEX idx_orders_shopify_id ON orders(shopifyOrderId);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_updated_at ON orders(updatedAt DESC);
```

---

## 🔄 Development Workflow

### Daily Development

1. **Start Docker Services**
   ```bash
   cd fincart-backend
   docker-compose up -d
   ```

2. **Start Dev Servers**
   ```bash
   # From root
   npm run dev
   ```

3. **Make Changes**
   - Backend: Changes auto-reload with NestJS watch mode
   - Frontend: Changes auto-reload with Next.js Fast Refresh

4. **Test Changes**
   ```bash
   # Test webhook
   node test-webhook.js
   
   # Check dashboard
   # Open http://localhost:3000/dashboard
   ```

### Adding New Features

#### Backend

1. Create new module: `nest g module feature-name`
2. Create controller: `nest g controller feature-name`
3. Create service: `nest g service feature-name`
4. Add to `app.module.ts`

#### Frontend

1. Create component in `components/`
2. Create hook in `hooks/` if needed
3. Add types in `types/`
4. Import and use in pages

### Code Quality

```bash
# Backend
cd fincart-backend
npm run lint
npm run format
npm run test

# Frontend
cd fincart-frontend
npm run lint
npm run build  # Type checking
```

---

## 🧪 Testing

### Manual Testing

See `TESTING_GUIDE.md` for detailed testing procedures.

**Quick Test**:
```bash
# 1. Start servers
npm run dev

# 2. Open dashboard
# http://localhost:3000/dashboard

# 3. Send test webhook
node test-webhook.js

# 4. Verify order appears in table
```

### Automated Testing

#### Backend Tests

```bash
cd fincart-backend

# Unit tests
npm run test

# E2E tests
npm run test:e2e

# Coverage
npm run test:cov
```

#### Frontend Tests

```bash
cd fincart-frontend

# Type checking
npm run build

# Linting
npm run lint
```

### Performance Testing

**Test with 5,000+ Orders**:

1. Modify `test-webhook.js` to send multiple webhooks
2. Monitor table scrolling performance
3. Check memory usage in DevTools
4. Verify no lag or stuttering

---

## 🚢 Deployment

### Backend Deployment

#### Docker

```bash
cd fincart-backend

# Build image
docker build -t fincart-backend .

# Run container
docker run -p 9000:9000 \
  -e DATABASE_URL=... \
  -e REDIS_HOST=... \
  fincart-backend
```

#### Production Environment

```bash
# Build
npm run build

# Start
npm run start:prod
```

### Frontend Deployment

#### Vercel (Recommended)

```bash
cd fincart-frontend

# Install Vercel CLI
npm i -g vercel

# Deploy
vercel
```

#### Docker

```bash
# Build
npm run build

# Start
npm run start
```

### Environment Variables

**Backend Production**:
- Set `NODE_ENV=production`
- Use production database credentials
- Configure Redis connection
- Set CORS origins

**Frontend Production**:
- Update `NEXT_PUBLIC_API_URL` to production backend URL
- Update `NEXT_PUBLIC_WS_URL` to production WebSocket URL

---

## 🔍 Troubleshooting

### Common Issues

#### Socket Won't Connect

**Symptoms**: Red "Socket Disconnected" badge

**Solutions**:
1. Verify backend is running: `curl http://localhost:9000/orders`
2. Check `.env.local` has correct URLs
3. Check backend Socket.IO namespace is `/sync`
4. Clear browser cache

#### Orders Not Displaying

**Symptoms**: Empty table or loading forever

**Solutions**:
1. Check backend has orders: `curl http://localhost:9000/orders`
2. Check browser console for errors
3. Verify CORS is enabled on backend
4. Check `NEXT_PUBLIC_API_URL` in `.env.local`

#### Real-Time Updates Not Working

**Symptoms**: Webhook sent but table doesn't update

**Solutions**:
1. Verify socket is connected (green badge)
2. Check browser console for `ORDER_SYNCED` events
3. Run `test-webhook.js` and watch console
4. Verify backend is emitting events

#### Port Already in Use

**Symptoms**: `Port 3000 is in use`

**Solutions**:
```bash
# Kill process on port 3000
# Windows
netstat -ano | findstr :3000
taskkill /PID <PID> /F

# Linux/Mac
lsof -ti:3000 | xargs kill -9

# Or use different port
npm run dev -- --port 5000
```

#### Database Connection Failed

**Symptoms**: Backend crashes on startup

**Solutions**:
1. Verify Docker is running: `docker ps`
2. Check PostgreSQL is up: `docker-compose ps`
3. Verify `.env` database credentials
4. Restart Docker services: `docker-compose restart`

---

## 📚 Additional Resources

### Documentation

- **Integration Guide**: `fincart-frontend/INTEGRATION_SUMMARY.md`
- **Testing Guide**: `fincart-frontend/TESTING_GUIDE.md`
- **Backend Guide**: `fincart-backend/FRONTEND_INTEGRATION_GUIDE.md`

### External Links

- [NestJS Documentation](https://docs.nestjs.com)
- [Next.js Documentation](https://nextjs.org/docs)
- [React Query Documentation](https://tanstack.com/query/latest)
- [Socket.IO Documentation](https://socket.io/docs/v4/)
- [TypeORM Documentation](https://typeorm.io)

---

## 📝 License

This project is private and proprietary.

---

## 👥 Support

For questions or issues:
1. Check this documentation
2. Review `TESTING_GUIDE.md`
3. Check browser console for errors
4. Review backend logs

---

**Last Updated**: 2026-01-17
**Version**: 1.0.0
