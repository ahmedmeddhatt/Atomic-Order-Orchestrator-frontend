# Chaos Test Execution Guide

## Overview
This guide walks through executing the chaos test to verify database consistency under concurrent conflicting updates.

---

## Prerequisites

### 1. Environment Setup
```bash
# Backend setup
cd fincart-backend
npm install
npm run build

# Frontend setup (optional for this test)
cd ../fincart-frontend
npm install
```

### 2. Services Running

#### Option A: Docker Compose (Recommended)
```bash
cd fincart-backend
docker-compose up -d

# Verify containers are healthy
docker-compose ps
```

You should see:
- `fincart-postgres`: running, healthy
- `fincart-redis`: running, healthy

#### Option B: Local PostgreSQL & Redis
If Docker is unavailable, ensure:
- PostgreSQL 15+ running on localhost:5432
- Redis 7+ running on localhost:6379
- Database: `fincart_db` with user `fincart` password `fincart_secret`

### 3. Backend Running

```bash
cd fincart-backend

# Development mode (with auto-reload)
npm run start:dev

# Or production mode
npm run build
npm run start:prod
```

Expected output:
```
[Nest] 12345 - 01/18/2026, 18:43:03     LOG [NestFactory] Starting Nest application...
[Nest] 12345 - 01/18/2026, 18:43:04     LOG [InstanceLoader] DatabaseModule dependencies initialized
[Nest] 12345 - 01/18/2026, 18:43:04     LOG [InstanceLoader] RedisModule dependencies initialized
[Nest] 12345 - 01/18/2026, 18:43:05     LOG [NestApplication] Nest application successfully started
[Nest] 12345 - 01/18/2026, 18:43:05     LOG [Bootstrap] Application running on port 3000
[Nest] 12345 - 01/18/2026, 18:43:05     LOG [Bootstrap] WebSocket available at ws://localhost:3000/sync
```

**IMPORTANT:** The default port in configuration.ts is 3000, but the chaos test defaults to 9000. You need to either:
- Set `PORT=9000` environment variable, OR
- Modify the chaos test to use port 3000

---

## Execution Steps

### Step 1: Start Backend on Port 9000

```bash
cd fincart-backend

# Option A: Set environment variable (PowerShell)
$env:PORT = 9000
npm run start:dev

# Option B: Set environment variable (Command Prompt)
set PORT=9000
npm run start:dev

# Option C: Modify configuration.ts temporarily
# Change: port: parseInt(process.env.PORT ?? '3000', 10),
# To:     port: parseInt(process.env.PORT ?? '9000', 10),
```

Wait for:
```
Application running on port 9000
WebSocket available at ws://localhost:9000/sync
```

### Step 2: Verify Database Connection

In a new terminal, test the backend:

```bash
# Test GET /orders endpoint
curl http://localhost:9000/orders

# Expected: JSON array (empty or with existing orders)
```

### Step 3: Run Chaos Test

In another terminal:

```bash
cd fincart-backend

# Run 100 concurrent updates
npx ts-node src/scripts/chaos-test-100.ts

# Or run the smaller test (10 updates)
npx ts-node src/scripts/chaos-test.ts
```

### Step 4: Monitor Backend Logs

Watch the backend terminal for processing:

```
[OrdersController] Received webhook: chaos-webhook-order_chaos_test_12345-0, topic: orders/updated
[OrdersController] Webhook queued for processing: chaos-webhook-order_chaos_test_12345-0
[OrdersProcessor] Processing job chaos-webhook-order_chaos_test_12345-0 for webhook chaos-webhook-order_chaos_test_12345-0
[OrdersProcessor] Order saved with status CONFIRMED, shipping fee: $123.45
[OrdersProcessor] Successfully processed webhook chaos-webhook-order_chaos_test_12345-0
...
```

Expected pattern:
- Initial burst of webhook receipts (all 100 simultaneously)
- Gradual processing as queue processes jobs serially
- Some version conflicts if processing occurs in different order than timestamp ordering
- Automatic retries for conflicts

---

## Verification (PostgreSQL)

### Step 1: Connect to Database

```bash
# Using psql
psql -h localhost -p 5435 -U fincart -d fincart_db

# Or use DBeaver, pgAdmin, or any PostgreSQL client
# Connection: localhost:5435, User: fincart, Password: fincart_secret
```

### Step 2: Check Final Order State

```sql
-- Query 1: Get the final order state
SELECT 
  id,
  shopifyOrderId,
  status,
  shippingFee,
  version,
  lastExternalUpdatedAt,
  createdAt,
  updatedAt
FROM orders
WHERE shopifyOrderId = 'order_chaos_test_12345'
ORDER BY updatedAt DESC
LIMIT 1;
```

**Expected Result:**
```
┌──────────────────────────────────────────────────────────────────────┐
│ id           │ order-uuid-123                                        │
│ shopifyOrderId │ order_chaos_test_12345                              │
│ status       │ CONFIRMED (or similar based on final timestamp)       │
│ shippingFee  │ 123.45                                                │
│ version      │ 100 (or close to it)                                  │
│ lastExternalUpdatedAt │ 2026-01-18T18:43:04.001Z                     │
└──────────────────────────────────────────────────────────────────────┘

Rows: 1 ✓ (NO DUPLICATES)
```

**Key Indicators:**
- ✅ Exactly 1 row (no duplicates despite concurrent updates)
- ✅ Version is high (100 or close)
- ✅ Status is consistent with the highest timestamp
- ✅ No NULL values
- ✅ Timestamps are reasonable

### Step 3: Check Audit Trail

```sql
-- Query 2: Audit log summary
SELECT 
  status,
  COUNT(*) as count
FROM audit_log
WHERE shopifyOrderId = 'order_chaos_test_12345'
GROUP BY status;
```

**Expected Result:**
```
┌──────────────────────────────────────────┐
│ status    │ count                        │
├──────────────────────────────────────────┤
│ processed │ 85 (majority)                │
│ discarded │ 15 (out-of-order)            │
│ failed    │ 0 (no errors)                │
└──────────────────────────────────────────┘

Total: 100 ✓
```

### Step 4: Verify No Version Gaps

```sql
-- Query 3: Check for version consistency
SELECT 
  COUNT(*) as duplicate_rows
FROM (
  SELECT shopifyOrderId, COUNT(*) as cnt
  FROM orders
  GROUP BY shopifyOrderId
  HAVING COUNT(*) > 1
) duplicates;
```

**Expected Result:**
```
duplicate_rows: 0 ✓
```

### Step 5: Review Version History

```sql
-- Query 4: Timeline of updates (from audit_log if timestamped)
SELECT 
  webhookId,
  status,
  message,
  createdAt
FROM audit_log
WHERE shopifyOrderId = 'order_chaos_test_12345'
ORDER BY createdAt ASC
LIMIT 20;
```

**Expected Pattern:**
```
All webhooks logged in order received, mix of processed/discarded status
```

---

## Troubleshooting

### Issue: "Cannot connect to backend"

**Solution 1: Check port**
```bash
netstat -ano | findstr :9000  # Windows
lsof -i :9000                 # macOS/Linux
```

If port is in use by another process, change the PORT:
```bash
$env:PORT = 9001
npm run start:dev
# Update chaos test to use 9001
```

**Solution 2: Ensure backend started**
```bash
# Check backend logs for errors
# Look for: "Application running on port 9000"
```

### Issue: "Cannot connect to database"

**Solution 1: Check PostgreSQL**
```bash
# Test connection
psql -h localhost -p 5435 -U fincart -d fincart_db -c "SELECT 1"

# If fails, check container
docker ps | grep postgres
docker logs fincart-postgres
```

**Solution 2: Check connection string**
```bash
# Verify environment variables
echo $DATABASE_HOST
echo $DATABASE_PORT
echo $DATABASE_USER
echo $DATABASE_NAME

# Should be:
# localhost, 5435, fincart, fincart_db
```

### Issue: "Cannot connect to Redis"

**Solution 1: Check Redis**
```bash
# Test connection
redis-cli -h localhost -p 6379 ping

# If fails, check container
docker ps | grep redis
docker logs fincart-redis
```

### Issue: "Network errors during chaos test"

**Solution:**
```bash
# Verify backend is fully started
# Look for logs like:
# [OrdersController] Received webhook
# [OrdersProcessor] Processing job

# If no logs, backend may have crashed
npm run start:dev  # Restart
```

### Issue: "Database has old test data"

**Solution: Clear test data**
```sql
-- Clean up old chaos test orders
DELETE FROM orders WHERE shopifyOrderId LIKE 'order_chaos_test_%';
DELETE FROM audit_log WHERE shopifyOrderId LIKE 'order_chaos_test_%';

-- Verify
SELECT COUNT(*) FROM orders;
```

---

## Success Criteria Checklist

After running the test and verification queries:

- [ ] **100 HTTP requests sent successfully** (check chaos test output)
- [ ] **All requests returned HTTP 200** (no 500 errors)
- [ ] **Database has exactly 1 row** for order_chaos_test_12345
- [ ] **Version number is ~100** (or close, accounting for discarded out-of-order updates)
- [ ] **No NULL values** in order row
- [ ] **Audit log has 100 entries** (one per webhook)
- [ ] **Mix of "processed" and "discarded" statuses** in audit log
- [ ] **No "failed" status entries** in audit log
- [ ] **Final status is consistent** with the last valid timestamp
- [ ] **Shipping fee is a valid number** (not corrupted)
- [ ] **Backend logs show serial processing** (no concurrent database updates)
- [ ] **No optimistic lock errors** in backend logs (or if present, automatic retries occurred)
- [ ] **WebSocket events emitted** (check if frontend would show updates)

---

## Performance Metrics

### Expected Timings

| Metric | Expected | Good | Excellent |
|--------|----------|------|-----------|
| Total test execution | < 10 seconds | < 5 seconds | < 2 seconds |
| Throughput | 10+ req/sec | 20+ req/sec | 50+ req/sec |
| DB query time (100 entries) | < 500ms | < 200ms | < 100ms |
| Queue processing | < 30 seconds | < 10 seconds | < 5 seconds |

### Sample Output

```
📈 Results:
   ✅ Successful: 100/100
   ❌ Failed: 0/100
   ⏱️  Total time: 2450ms
   📊 Throughput: 40.82 requests/sec
```

---

## Next Steps

### After Successful Test

1. **Run Frontend Test:**
   ```bash
   cd fincart-frontend
   npm install
   npm run dev
   # Visit http://localhost:3000
   # Should see the order in the table
   # Should receive real-time updates via WebSocket
   ```

2. **Run Multiple Chaos Tests:**
   - Test with different order IDs
   - Test with different concurrent levels (10, 50, 100, 500)
   - Measure performance degradation

3. **Stress Test:**
   ```bash
   # Send 1000 webhooks for different orders
   for i in {1..1000}; do
     npx ts-node src/scripts/chaos-test.ts
   done
   ```

4. **Monitor Production Ready:**
   - Set up logging aggregation
   - Set up performance monitoring
   - Set up alerts for queue backlog
   - Set up database health checks

---

## Common Commands Reference

```bash
# Start backend
npm run start:dev

# Run chaos test (10 updates)
npx ts-node src/scripts/chaos-test.ts

# Run chaos test (100 updates)
npx ts-node src/scripts/chaos-test-100.ts

# Check Docker containers
docker-compose ps
docker-compose logs -f postgres
docker-compose logs -f redis

# Connect to database
psql -h localhost -p 5435 -U fincart -d fincart_db

# Connect to Redis
redis-cli -h localhost -p 6379

# Kill process on port 9000
lsof -ti :9000 | xargs kill -9  # macOS/Linux
```

---

## Expected Architecture Confirmation

By running this test, you're verifying:

1. ✅ **Webhook Ingestion** - All 100 webhooks accepted
2. ✅ **Idempotency** - Detected duplicates in Redis
3. ✅ **Queue Management** - BullMQ processed jobs
4. ✅ **Concurrency Control** - Jobs processed serially
5. ✅ **Optimistic Locking** - Version conflicts handled
6. ✅ **Out-of-Order Detection** - Stale updates discarded
7. ✅ **Data Consistency** - Single row, correct version
8. ✅ **Audit Trail** - All webhooks logged
9. ✅ **Real-Time Sync** - WebSocket events emitted
10. ✅ **Error Handling** - Graceful failure handling

---

**Test Report:** [SYSTEM_TEST_REPORT.md](../SYSTEM_TEST_REPORT.md)  
**Generated:** 2026-01-18  
**Status:** ✅ Ready for Execution
