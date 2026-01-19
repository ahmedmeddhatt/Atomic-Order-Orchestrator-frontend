# Quick Reference Guide

## 🚀 Quick Start Commands

### First Time Setup
```bash
# Clone and install
git clone <repo-url>
cd fincart-monorepo
npm install

# Start Docker services
cd fincart-backend
docker-compose up -d

# Start development servers
cd ..
npm run dev
```

### Daily Development
```bash
# Start everything
npm run dev

# Or separately:
npm run dev:backend   # Backend only
npm run dev:frontend  # Frontend only
```

### Access Points
- **Dashboard**: http://localhost:3000/dashboard
- **Backend API**: http://localhost:9000
- **Test Webhook**: `node test-webhook.js`

---

## 📋 Common Commands

### Backend
```bash
cd fincart-backend

# Development
npm run start:dev

# Production
npm run build
npm run start:prod

# Testing
npm run test
npm run test:e2e
npm run test:cov

# Database
docker-compose up -d        # Start PostgreSQL + Redis
docker-compose down         # Stop services
docker-compose logs -f      # View logs
```

### Frontend
```bash
cd fincart-frontend

# Development
npm run dev
npm run dev -- --port 5000  # Custom port

# Production
npm run build
npm run start

# Type checking
npm run build  # Also checks types
```

---

## 🔧 Environment Variables

### Backend (`.env`)
```env
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USER=postgres
DATABASE_PASSWORD=postgres
DATABASE_NAME=fincart_db
REDIS_HOST=localhost
REDIS_PORT=6379
PORT=9000
```

### Frontend (`.env.local`)
```env
NEXT_PUBLIC_API_URL=http://localhost:9000
NEXT_PUBLIC_WS_URL=http://localhost:9000
```

---

## 🎯 Key Files

### Backend
```
src/
├── orders/
│   ├── orders.controller.ts    # HTTP endpoints
│   ├── orders.service.ts       # Business logic
│   ├── orders.processor.ts     # Queue processing
│   └── entities/order.entity.ts # Database model
└── sync/
    └── sync.gateway.ts         # WebSocket gateway
```

### Frontend
```
app/
├── dashboard/page.tsx          # Main dashboard
components/
├── orders-table.tsx            # Virtualized table
└── edit-order-modal.tsx        # Edit modal
hooks/
├── useOrders.ts                # Data fetching
├── useOrderSync.ts             # WebSocket sync
└── useSocket.ts                # Socket wrapper
types/
└── order.ts                    # Type definitions
```

---

## 📡 API Endpoints

### GET /orders
```bash
curl http://localhost:9000/orders
```

### POST /webhooks/shopify
```bash
curl -X POST http://localhost:9000/webhooks/shopify \
  -H "Content-Type: application/json" \
  -H "x-shopify-webhook-id: webhook-123" \
  -H "x-shopify-topic: orders/create" \
  -d '{
    "id": "123456789",
    "updated_at": "2026-01-17T18:30:00Z",
    "financial_status": "paid",
    "fulfillment_status": null,
    "total_price": "150.00"
  }'
```

---

## 🔌 WebSocket Events

### Connect
```typescript
import { io } from 'socket.io-client';

const socket = io('http://localhost:9000/sync', {
  transports: ['websocket', 'polling'],
  reconnection: true,
});
```

### Listen for Events
```typescript
socket.on('ORDER_SYNCED', (event) => {
  console.log('Order updated:', event);
  // { id, shopifyOrderId, status, shippingFee, version, updatedAt }
});
```

---

## 🐛 Debugging

### Check Backend Health
```bash
# API is responding
curl http://localhost:9000/orders

# Database is running
docker ps | grep postgres

# Redis is running
docker ps | grep redis
```

### Check Frontend
```bash
# Open browser console (F12)
# Should see: "Socket connected: <id>"

# Check network tab
# Should see: WebSocket connection to /sync
```

### View Logs
```bash
# Backend logs
cd fincart-backend
npm run start:dev  # Watch console

# Docker logs
docker-compose logs -f postgres
docker-compose logs -f redis
```

---

## 🧪 Testing Workflow

### 1. Start Servers
```bash
npm run dev
```

### 2. Open Dashboard
```
http://localhost:3000/dashboard
```

### 3. Send Test Webhook
```bash
node test-webhook.js
```

### 4. Verify
- ✅ Order appears in table
- ✅ Socket badge is green
- ✅ No console errors

---

## 🔄 Git Workflow

### Feature Development
```bash
# Create feature branch
git checkout -b feature/my-feature

# Make changes
# ...

# Commit
git add .
git commit -m "feat: add my feature"

# Push
git push origin feature/my-feature

# Create PR
```

### Commit Message Format
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

## 📦 Dependencies

### Add New Dependency

**Backend**:
```bash
cd fincart-backend
npm install <package-name>
```

**Frontend**:
```bash
cd fincart-frontend
npm install <package-name>
```

### Update Dependencies
```bash
# Check outdated
npm outdated

# Update all
npm update

# Update specific package
npm install <package-name>@latest
```

---

## 🚨 Emergency Fixes

### Kill All Node Processes
```bash
# Windows
taskkill /F /IM node.exe

# Linux/Mac
pkill -9 node
```

### Reset Database
```bash
cd fincart-backend
docker-compose down -v  # Delete volumes
docker-compose up -d    # Recreate
```

### Clear Frontend Cache
```bash
cd fincart-frontend
rm -rf .next
npm run build
```

### Reset Everything
```bash
# Stop all services
docker-compose down -v

# Clean install
rm -rf node_modules package-lock.json
npm install

# Restart
docker-compose up -d
npm run dev
```

---

## 📊 Performance Tips

### Backend
- Use database indexes for queries
- Enable Redis caching
- Use BullMQ for async processing
- Monitor memory usage

### Frontend
- Use React Query for caching
- Implement virtualization for large lists
- Lazy load components
- Optimize images

---

## 🔐 Security Checklist

- [ ] Environment variables not committed
- [ ] CORS configured properly
- [ ] Webhook signature validation
- [ ] SQL injection prevention (TypeORM)
- [ ] XSS prevention (React escaping)
- [ ] Rate limiting on API endpoints
- [ ] HTTPS in production

---

## 📝 Code Snippets

### Create New Order (Backend)
```typescript
const order = await this.ordersService.create({
  shopifyOrderId: '123456',
  status: OrderStatus.CONFIRMED,
  shippingFee: 9.99,
  lastExternalUpdatedAt: new Date(),
});
```

### Fetch Orders (Frontend)
```typescript
const { orders, isLoading } = useOrders();
```

### Listen for Updates (Frontend)
```typescript
const { isConnected } = useOrderSync();
```

### Emit WebSocket Event (Backend)
```typescript
this.syncGateway.server
  .to('sync')
  .emit('ORDER_SYNCED', orderData);
```

---

## 🎓 Learning Resources

### NestJS
- [Official Docs](https://docs.nestjs.com)
- [NestJS Fundamentals Course](https://courses.nestjs.com)

### Next.js
- [Official Docs](https://nextjs.org/docs)
- [Learn Next.js](https://nextjs.org/learn)

### React Query
- [Official Docs](https://tanstack.com/query/latest)
- [React Query Tutorial](https://tanstack.com/query/latest/docs/react/overview)

### Socket.IO
- [Official Docs](https://socket.io/docs/v4/)
- [Socket.IO Tutorial](https://socket.io/get-started/chat)

---

## 📞 Quick Help

| Issue | Solution |
|-------|----------|
| Port in use | `taskkill /F /IM node.exe` or use different port |
| Socket disconnected | Check backend is running, verify `.env.local` |
| Orders not loading | Check `curl http://localhost:9000/orders` |
| Database error | `docker-compose restart postgres` |
| Redis error | `docker-compose restart redis` |
| Build fails | `rm -rf .next && npm run build` |
| Type errors | Check `types/order.ts` matches backend |

---

**Quick Links**:
- [Full Documentation](./PROJECT_DOCUMENTATION.md)
- [Integration Guide](./fincart-frontend/INTEGRATION_SUMMARY.md)
- [Testing Guide](./fincart-frontend/TESTING_GUIDE.md)
