# Atomic Order Orchestrator

> A high-performance, real-time logistics dashboard for managing 5,000+ orders with WebSocket synchronization and optimistic locking.

[![NestJS](https://img.shields.io/badge/NestJS-10.x-E0234E?logo=nestjs)](https://nestjs.com/)
[![Next.js](https://img.shields.io/badge/Next.js-16.1-000000?logo=next.js)](https://nextjs.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-3178C6?logo=typescript)](https://www.typescriptlang.org/)
[![Socket.IO](https://img.shields.io/badge/Socket.IO-4.x-010101?logo=socket.io)](https://socket.io/)

---

## 🌟 Features

- ✅ **Real-Time Synchronization** - WebSocket-based live updates across all clients
- ✅ **High Performance** - Virtualized rendering handles 5,000+ orders smoothly
- ✅ **Optimistic Locking** - Version-based conflict detection and resolution
- ✅ **Shopify Integration** - Webhook processing for order events
- ✅ **Scalable Architecture** - BullMQ queue processing with Redis
- ✅ **Type-Safe** - Full TypeScript implementation (Backend + Frontend)
- ✅ **Modern Stack** - NestJS 10, Next.js 16, React 19, TailwindCSS 4

---

## 📸 Screenshots

### Dashboard
![Dashboard](docs/screenshots/dashboard.png)
*Real-time order monitoring with live socket connection*

### Conflict Resolution
![Conflict Resolution](docs/screenshots/conflict-modal.png)
*Merge conflict detection with resolution options*

---

## 🚀 Quick Start

### Prerequisites

- Node.js 18+ or 20+
- Docker (for PostgreSQL & Redis)
- npm 9+

### Installation

```bash
# 1. Clone repository
git clone <repository-url>
cd fincart-monorepo

# 2. Install dependencies
npm install

# 3. Start Docker services (PostgreSQL + Redis)
cd fincart-backend
docker-compose up -d

# 4. Start development servers
cd ..
npm run dev
```

### Access

- **Dashboard**: http://localhost:3000/dashboard
- **Backend API**: http://localhost:9000
- **Test Webhook**: `node test-webhook.js`

---

## 📁 Project Structure

```
fincart-monorepo/
├── fincart-backend/          # NestJS Backend (Port 9000)
│   ├── src/
│   │   ├── orders/           # Orders module
│   │   └── sync/             # WebSocket gateway
│   └── docker-compose.yml    # PostgreSQL + Redis
│
├── fincart-frontend/         # Next.js Frontend (Port 3000)
│   ├── app/                  # Next.js App Router
│   ├── components/           # React components
│   ├── hooks/                # Custom hooks
│   └── types/                # TypeScript types
│
├── test-webhook.js           # Webhook testing script
└── package.json              # Monorepo scripts
```

---

## 🛠️ Technology Stack

### Backend
- **NestJS** - Progressive Node.js framework
- **TypeORM** - ORM for PostgreSQL
- **BullMQ** - Queue processing with Redis
- **Socket.IO** - WebSocket communication
- **PostgreSQL** - Primary database
- **Redis** - Queue storage & caching

### Frontend
- **Next.js 16** - React framework with App Router
- **React 19** - UI library
- **TanStack Query** - Data fetching & caching
- **TanStack Virtual** - Virtualized rendering
- **Socket.IO Client** - WebSocket client
- **TailwindCSS 4** - Utility-first CSS

---

## 📚 Documentation

| Document | Description |
|----------|-------------|
| [**TEST_SUITE_INDEX.md**](./TEST_SUITE_INDEX.md) | **📊 Complete Test Suite Overview** |
| [**SYSTEM_TEST_REPORT.md**](./SYSTEM_TEST_REPORT.md) | **✅ Comprehensive Test Verification Report** |
| [**CHAOS_TEST_EXECUTION_GUIDE.md**](./CHAOS_TEST_EXECUTION_GUIDE.md) | **🔥 Chaos Test Step-by-Step Guide** |
| [**ARCHITECTURE.md**](./ARCHITECTURE.md) | System architecture & design patterns |
| [**PROJECT_DOCUMENTATION.md**](./PROJECT_DOCUMENTATION.md) | Complete project documentation |
| [**QUICK_REFERENCE.md**](./QUICK_REFERENCE.md) | Quick reference guide |
| [**INTEGRATION_SUMMARY.md**](./fincart-frontend/INTEGRATION_SUMMARY.md) | Frontend-backend integration |
| [**TESTING_GUIDE.md**](./fincart-frontend/TESTING_GUIDE.md) | Testing procedures |

---

## 🔧 Development

### Start Development Servers

```bash
# Both servers simultaneously
npm run dev

# Or separately
npm run dev:backend   # Backend only
npm run dev:frontend  # Frontend only
```

### Test Real-Time Updates

```bash
# Send test webhook
node test-webhook.js

# Verify order appears in dashboard
# http://localhost:3000/dashboard
```

### Build for Production

```bash
# Build both projects
npm run build

# Or separately
cd fincart-backend && npm run build
cd fincart-frontend && npm run build
```

---

## ✅ Testing & Verification

### Chaos Test (100 Concurrent Updates)

Verify database consistency when 100 simultaneous conflicting updates are sent to the same order:

```bash
# Start backend on port 9000
cd fincart-backend
PORT=9000 npm run start:dev

# In another terminal, run chaos test
npx ts-node src/scripts/chaos-test-100.ts
```

**Expected Result:**
- ✅ All 100 webhooks processed
- ✅ Single row in database (no duplicates)
- ✅ Version number = 100 (or close)
- ✅ No data loss or corruption

📊 **Read Full Test Report:** [SYSTEM_TEST_REPORT.md](./SYSTEM_TEST_REPORT.md)

📖 **Read Execution Guide:** [CHAOS_TEST_EXECUTION_GUIDE.md](./CHAOS_TEST_EXECUTION_GUIDE.md)

### Unit Tests

```bash
cd fincart-backend
npm run test                    # Run unit tests
npm run test:watch             # Watch mode
npm run test:cov               # Coverage report
```

### E2E Tests

```bash
npm run test:e2e                # Run end-to-end tests
```

---

## 📡 API Reference

### GET /orders
Fetch all orders from database.

```bash
curl http://localhost:9000/orders
```

**Response**:
```json
[
  {
    "id": "uuid",
    "shopifyOrderId": "123456789",
    "status": "CONFIRMED",
    "shippingFee": "5.99",
    "version": 1,
    "createdAt": "2026-01-17T18:30:00.000Z",
    "updatedAt": "2026-01-17T18:30:00.000Z"
  }
]
```

### POST /webhooks/shopify
Receive Shopify webhook events.

```bash
curl -X POST http://localhost:9000/webhooks/shopify \
  -H "Content-Type: application/json" \
  -H "x-shopify-webhook-id: webhook-123" \
  -H "x-shopify-topic: orders/create" \
  -d '{"id": "123456789", "financial_status": "paid", "total_price": "150.00"}'
```

---

## 🔌 WebSocket Events

### ORDER_SYNCED
Emitted when an order is created or updated.

**Payload**:
```typescript
{
  id: string;
  shopifyOrderId: string;
  status: 'PENDING' | 'CONFIRMED' | 'SHIPPED' | 'CANCELLED';
  shippingFee: number;
  version: number;
  updatedAt: string;
}
```

**Frontend Usage**:
```typescript
const { isConnected } = useOrderSync();
// Automatically updates React Query cache
```

---

## 🧪 Testing

### Manual Testing

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

```bash
# Backend tests
cd fincart-backend
npm run test
npm run test:e2e

# Frontend type checking
cd fincart-frontend
npm run build
```

---

## 🚢 Deployment

### Backend

```bash
cd fincart-backend

# Build
npm run build

# Start production server
npm run start:prod
```

**Environment Variables**:
```env
DATABASE_URL=postgresql://user:pass@host:5432/db
REDIS_HOST=redis-host
PORT=9000
NODE_ENV=production
```

### Frontend

```bash
cd fincart-frontend

# Build
npm run build

# Start production server
npm run start
```

**Environment Variables**:
```env
NEXT_PUBLIC_API_URL=https://api.yourdomain.com
NEXT_PUBLIC_WS_URL=https://api.yourdomain.com
```

### Docker Deployment

```bash
# Build images
docker build -t fincart-backend ./fincart-backend
docker build -t fincart-frontend ./fincart-frontend

# Run containers
docker-compose up -d
```

---

## 🐛 Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| **Socket Disconnected** | Check backend is running, verify `.env.local` URLs |
| **Orders Not Loading** | Run `curl http://localhost:9000/orders` to test API |
| **Port Already in Use** | Kill node processes or use different port |
| **Database Connection Failed** | Check Docker is running: `docker ps` |

### Debug Commands

```bash
# Check backend health
curl http://localhost:9000/orders

# View Docker logs
cd fincart-backend
docker-compose logs -f

# Kill all node processes (Windows)
taskkill /F /IM node.exe

# Kill all node processes (Linux/Mac)
pkill -9 node
```

See [QUICK_REFERENCE.md](./QUICK_REFERENCE.md) for more troubleshooting tips.

---

## 📊 Performance

### Benchmarks

- **Table Rendering**: 5,000+ rows at 60 FPS
- **WebSocket Latency**: < 50ms
- **API Response Time**: < 100ms
- **Memory Usage**: < 200MB (frontend)

### Optimization Techniques

- **Virtualization**: Only renders visible table rows
- **React Query**: Intelligent caching and deduplication
- **Surgical Updates**: Updates only changed data in cache
- **BullMQ**: Async job processing for heavy operations

---

## 🔐 Security

- ✅ Environment variables for sensitive data
- ✅ CORS configuration
- ✅ Webhook signature validation
- ✅ SQL injection prevention (TypeORM)
- ✅ XSS prevention (React escaping)
- ✅ Rate limiting (recommended for production)

---

## 🤝 Contributing

1. Fork the repository
2. Create feature branch: `git checkout -b feature/my-feature`
3. Commit changes: `git commit -m 'feat: add my feature'`
4. Push to branch: `git push origin feature/my-feature`
5. Submit pull request

### Commit Convention

```
feat: add new feature
fix: fix bug
docs: update documentation
style: format code
refactor: refactor code
test: add tests
chore: update dependencies
```

---

## 📝 License

This project is private and proprietary.

---

## 👥 Team

- **Backend**: NestJS + TypeORM + BullMQ
- **Frontend**: Next.js + React Query + Socket.IO
- **DevOps**: Docker + Docker Compose

---

## 📞 Support

For questions or issues:

1. Check [PROJECT_DOCUMENTATION.md](./PROJECT_DOCUMENTATION.md)
2. Review [QUICK_REFERENCE.md](./QUICK_REFERENCE.md)
3. Check [TESTING_GUIDE.md](./fincart-frontend/TESTING_GUIDE.md)
4. Review browser console for errors
5. Check backend logs

---

## 🗺️ Roadmap

### Current Version (1.0.0)
- ✅ Real-time order synchronization
- ✅ Conflict resolution
- ✅ Virtualized table rendering
- ✅ Shopify webhook integration

### Future Enhancements
- [ ] Advanced filtering and search
- [ ] Bulk operations
- [ ] Export to CSV/Excel
- [ ] Email notifications
- [ ] Analytics dashboard
- [ ] Multi-user support with roles
- [ ] Order history tracking
- [ ] Customer data integration

---

## 🙏 Acknowledgments

- [NestJS](https://nestjs.com/) - Backend framework
- [Next.js](https://nextjs.org/) - Frontend framework
- [TanStack Query](https://tanstack.com/query) - Data fetching
- [Socket.IO](https://socket.io/) - WebSocket library
- [TailwindCSS](https://tailwindcss.com/) - Styling framework

---

**Built with ❤️ using TypeScript**

Last Updated: 2026-01-17 | Version: 1.0.0
