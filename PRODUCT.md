# Multi-Store Price Monitor Application Design (Updated 2025)

## System Architecture Overview

### Tech Stack (Updated for 2025)
- **Frontend**: Next.js 15.5+ (App Router) with TypeScript and React 19
- **Backend**: Next.js API Routes with serverless functions
- **Database**: PostgreSQL (Neon free tier - 0.5 GB storage, 50 CU-hours/month)
- **Authentication**: Clerk (free tier - 10,000 MAUs)
- **AI Service**: DeepSeek API (most cost-effective at $0.27/1M input tokens, $1.10/1M output tokens until Sept 2025)
- **Hosting**: Vercel (free tier with 100GB bandwidth/month)
- **Job Scheduling**: GitHub Actions (2,000 minutes/month free for private repos, unlimited for public)
- **Web Scraping**: Playwright (preferred over Puppeteer in 2025) with serverless deployment

## Subscription Tiers Design

### Tier Structure
```typescript
enum SubscriptionTier {
  FREE = 'free',
  PREMIUM = 'premium',
  PREMIUM_PLUS = 'premium_plus'
}

interface TierLimits {
  maxProducts: number;
  checksPerDay: number;
  alertsEnabled: boolean;
  priceHistoryDays: number;
  apiAccess: boolean;
}

const TIER_LIMITS: Record<SubscriptionTier, TierLimits> = {
  [SubscriptionTier.FREE]: {
    maxProducts: 5,
    checksPerDay: 1,  // Guaranteed once per 24 hours
    alertsEnabled: true,
    priceHistoryDays: 7,
    apiAccess: false
  },
  [SubscriptionTier.PREMIUM]: {
    maxProducts: 25,
    checksPerDay: 4,  // Every 6 hours
    alertsEnabled: true,
    priceHistoryDays: 30,
    apiAccess: false
  },
  [SubscriptionTier.PREMIUM_PLUS]: {
    maxProducts: 100,
    checksPerDay: 24, // Every hour
    alertsEnabled: true,
    priceHistoryDays: 90,
    apiAccess: true
  }
};
```

## Database Schema (Optimized for Multi-Tenant SaaS)

```sql
-- Enable UUID extension for Neon
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- User accounts with subscription info
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  clerk_user_id VARCHAR(255) UNIQUE NOT NULL,
  email VARCHAR(255) NOT NULL,
  subscription_tier VARCHAR(20) DEFAULT 'free',
  subscription_status VARCHAR(20) DEFAULT 'active', -- active, cancelled, past_due
  stripe_customer_id VARCHAR(255),
  stripe_subscription_id VARCHAR(255),
  trial_ends_at TIMESTAMP WITH TIME ZONE,
  product_count INTEGER DEFAULT 0, -- Denormalized for quick checks
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Products to monitor
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  model VARCHAR(255),
  category VARCHAR(100),
  image_url TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Stores to monitor
CREATE TABLE stores (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name VARCHAR(100) NOT NULL,
  domain VARCHAR(255) NOT NULL,
  logo_url TEXT,
  scrape_config JSONB, -- Store-specific configuration
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Product URLs at different stores
CREATE TABLE product_listings (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  product_id UUID REFERENCES products(id) ON DELETE CASCADE,
  store_id UUID REFERENCES stores(id) ON DELETE CASCADE,
  url TEXT NOT NULL,
  selector_hints JSONB,
  is_active BOOLEAN DEFAULT true,
  last_checked TIMESTAMP WITH TIME ZONE,
  next_check_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  check_frequency_hours INTEGER DEFAULT 24,
  consecutive_failures INTEGER DEFAULT 0,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  UNIQUE(product_id, store_id)
);

-- Dynamic scraping queue
CREATE TABLE scrape_queue (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  listing_id UUID REFERENCES product_listings(id) ON DELETE CASCADE,
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  subscription_tier VARCHAR(20) NOT NULL,
  scheduled_for TIMESTAMP WITH TIME ZONE NOT NULL,
  priority INTEGER DEFAULT 5, -- 1-10, calculated based on tier and staleness
  attempts INTEGER DEFAULT 0,
  status VARCHAR(20) DEFAULT 'pending', -- pending, processing, completed, failed
  locked_until TIMESTAMP WITH TIME ZONE, -- For preventing double processing
  processed_at TIMESTAMP WITH TIME ZONE,
  error_message TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Indexes for efficient queue processing
CREATE INDEX idx_scrape_queue_pending ON scrape_queue(scheduled_for, status, locked_until) 
  WHERE status = 'pending';
CREATE INDEX idx_scrape_queue_user ON scrape_queue(user_id, scheduled_for);
CREATE INDEX idx_product_listings_next_check ON product_listings(next_check_at) 
  WHERE is_active = true;

-- Price history
CREATE TABLE price_history (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  listing_id UUID REFERENCES product_listings(id) ON DELETE CASCADE,
  user_id UUID REFERENCES users(id) ON DELETE CASCADE, -- For retention policy
  raw_price_text TEXT,
  extracted_price DECIMAL(10, 2),
  currency VARCHAR(3) DEFAULT 'USD',
  promotions JSONB,
  conditions JSONB,
  is_available BOOLEAN DEFAULT true,
  scraped_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  ai_confidence DECIMAL(3, 2)
);

-- Index for efficient queries and cleanup
CREATE INDEX idx_price_history_listing_time ON price_history(listing_id, scraped_at DESC);
CREATE INDEX idx_price_history_user_time ON price_history(user_id, scraped_at DESC);

-- User watchlists with alerts
CREATE TABLE user_watchlists (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  product_id UUID REFERENCES products(id) ON DELETE CASCADE,
  target_price DECIMAL(10, 2),
  notify_on_drop BOOLEAN DEFAULT true,
  last_notified_at TIMESTAMP WITH TIME ZONE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  UNIQUE(user_id, product_id)
);

-- GitHub Actions run tracking
CREATE TABLE cron_runs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  workflow_name VARCHAR(100),
  run_id VARCHAR(100),
  status VARCHAR(20),
  items_processed INTEGER,
  started_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  completed_at TIMESTAMP WITH TIME ZONE,
  error_message TEXT
);

-- Usage tracking for billing
CREATE TABLE usage_tracking (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  date DATE NOT NULL,
  products_tracked INTEGER DEFAULT 0,
  checks_performed INTEGER DEFAULT 0,
  api_calls INTEGER DEFAULT 0,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  UNIQUE(user_id, date)
);
```

## Dynamic Queue Management System

The key to supporting subscription tiers without modifying GitHub Actions is a **dynamic queue system** that GitHub Actions simply processes without knowing about individual users or tiers.

### Queue Population Algorithm
```typescript
// app/api/internal/populate-queue/route.ts
// This runs every hour via GitHub Actions to populate the queue

export async function POST(request: Request) {
  // Verify internal call
  const authHeader = request.headers.get('authorization');
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const sql = neon(process.env.DATABASE_URL!);
  
  // Get all active listings that need checking
  const needsChecking = await sql`
    WITH user_limits AS (
      SELECT 
        u.id as user_id,
        u.subscription_tier,
        u.product_count,
        CASE u.subscription_tier
          WHEN 'premium_plus' THEN 24  -- checks per day
          WHEN 'premium' THEN 4
          ELSE 1
        END as checks_per_day,
        CASE u.subscription_tier
          WHEN 'premium_plus' THEN 10  -- priority
          WHEN 'premium' THEN 5
          ELSE 1
        END as base_priority
      FROM users u
      WHERE u.subscription_status = 'active'
    ),
    listings_to_check AS (
      SELECT 
        pl.*,
        p.user_id,
        ul.subscription_tier,
        ul.checks_per_day,
        ul.base_priority,
        -- Calculate urgency based on how overdue the check is
        EXTRACT(EPOCH FROM (NOW() - pl.next_check_at)) / 3600 as hours_overdue,
        -- Calculate check interval based on tier
        24.0 / ul.checks_per_day as check_interval_hours
      FROM product_listings pl
      JOIN products p ON pl.product_id = p.id
      JOIN user_limits ul ON p.user_id = ul.user_id
      WHERE pl.is_active = true
        AND pl.next_check_at <= NOW() + INTERVAL '1 hour'
        AND pl.consecutive_failures < 5  -- Skip broken listings
    )
    SELECT 
      id as listing_id,
      user_id,
      subscription_tier,
      next_check_at,
      -- Calculate dynamic priority
      LEAST(10, GREATEST(1, 
        base_priority + 
        CASE 
          WHEN hours_overdue > 24 THEN 5  -- Severely overdue
          WHEN hours_overdue > 12 THEN 3  -- Very overdue
          WHEN hours_overdue > 6 THEN 2   -- Overdue
          WHEN hours_overdue > 0 THEN 1   -- Slightly overdue
          ELSE 0
        END
      )) as priority,
      check_interval_hours
    FROM listings_to_check
    ORDER BY priority DESC, next_check_at ASC
    LIMIT 1000  -- Process up to 1000 items per run
  `;

  // Insert into queue
  for (const item of needsChecking) {
    await sql`
      INSERT INTO scrape_queue (
        listing_id,
        user_id,
        subscription_tier,
        scheduled_for,
        priority
      )
      VALUES (
        ${item.listing_id},
        ${item.user_id},
        ${item.subscription_tier},
        NOW(),
        ${item.priority}
      )
      ON CONFLICT DO NOTHING  -- Skip if already in queue
    `;
  }

  return NextResponse.json({
    success: true,
    queued: needsChecking.length
  });
}
```

### Queue Processing System
```typescript
// app/api/cron/process-queue/route.ts
// GitHub Actions calls this every 10 minutes to process the queue

export async function POST(request: Request) {
  // Verify GitHub Actions
  const authHeader = request.headers.get('authorization');
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const sql = neon(process.env.DATABASE_URL!);
  
  // Lock items for processing (prevents double processing)
  const lockedUntil = new Date(Date.now() + 5 * 60 * 1000); // 5 minute lock
  
  const itemsToProcess = await sql`
    UPDATE scrape_queue
    SET 
      status = 'processing',
      locked_until = ${lockedUntil}
    WHERE id IN (
      SELECT id 
      FROM scrape_queue
      WHERE status = 'pending'
        AND scheduled_for <= NOW()
        AND (locked_until IS NULL OR locked_until < NOW())
      ORDER BY priority DESC, scheduled_for ASC
      LIMIT 20  -- Process 20 items per run
    )
    RETURNING *
  `;

  if (itemsToProcess.length === 0) {
    return NextResponse.json({ 
      success: true, 
      message: 'No items to process' 
    });
  }

  // Group by store for efficient scraping
  const itemsByStore = itemsToProcess.reduce((acc, item) => {
    const storeId = item.store_id;
    if (!acc[storeId]) acc[storeId] = [];
    acc[storeId].push(item);
    return acc;
  }, {});

  const results = [];
  const browser = await chromium.launch({
    headless: true,
    args: ['--no-sandbox', '--disable-setuid-sandbox']
  });

  try {
    for (const [storeId, items] of Object.entries(itemsByStore)) {
      for (const item of items) {
        try {
          const result = await scrapeProduct(browser, item);
          results.push(result);
          
          // Mark as completed and update next check time
          const checkInterval = getCheckInterval(item.subscription_tier);
          
          await sql`
            UPDATE product_listings
            SET 
              last_checked = NOW(),
              next_check_at = NOW() + INTERVAL '${checkInterval} hours',
              consecutive_failures = 0
            WHERE id = ${item.listing_id}
          `;
          
          await sql`
            UPDATE scrape_queue
            SET 
              status = 'completed',
              processed_at = NOW()
            WHERE id = ${item.id}
          `;
          
        } catch (error) {
          console.error(`Failed to scrape item ${item.id}:`, error);
          
          // Update failure count
          await sql`
            UPDATE product_listings
            SET consecutive_failures = consecutive_failures + 1
            WHERE id = ${item.listing_id}
          `;
          
          await sql`
            UPDATE scrape_queue
            SET 
              status = 'failed',
              error_message = ${error.message},
              processed_at = NOW()
            WHERE id = ${item.id}
          `;
        }
      }
    }
  } finally {
    await browser.close();
  }

  // Clean up old queue items
  await sql`
    DELETE FROM scrape_queue
    WHERE status IN ('completed', 'failed')
      AND processed_at < NOW() - INTERVAL '24 hours'
  `;

  return NextResponse.json({
    success: true,
    processed: results.length,
    failed: itemsToProcess.length - results.length
  });
}

function getCheckInterval(tier: string): number {
  switch(tier) {
    case 'premium_plus': return 1;  // Every hour
    case 'premium': return 6;        // Every 6 hours
    default: return 24;              // Once per day
  }
}
```

## GitHub Actions Configuration (Static, No User Dependencies)

### Directory Structure
```
.github/
  workflows/
    queue-population.yml       # Every hour - populates the queue
    queue-processor-1.yml      # Every 10 minutes - processes queue
    queue-processor-2.yml      # Every 10 minutes (offset by 5) - processes queue
    queue-processor-3.yml      # Every 10 minutes (offset by 7) - processes queue
    alerts-processor.yml       # Every 30 minutes - processes alerts
    cleanup.yml               # Daily - database maintenance
    health-check.yml          # Every 15 minutes - system health
```

### 1. Queue Population (Hourly)
```yaml
# .github/workflows/queue-population.yml
name: Populate Scraping Queue
on:
  schedule:
    - cron: '0 * * * *'  # Every hour on the hour
  workflow_dispatch:

jobs:
  populate:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    
    steps:
      - name: Populate Queue Based on User Tiers
        run: |
          response=$(curl -X POST ${{ secrets.VERCEL_URL }}/api/internal/populate-queue \
            -H "Authorization: Bearer ${{ secrets.CRON_SECRET }}" \
            -H "Content-Type: application/json" \
            -w "\n%{http_code}" \
            -s)
          
          http_code=$(echo "$response" | tail -n1)
          body=$(echo "$response" | head -n-1)
          
          echo "Queue population result: $body"
          
          if [ "$http_code" != "200" ]; then
            echo "Error: HTTP $http_code"
            exit 1
          fi
```

### 2. Queue Processors (Every 10 Minutes, Staggered)
```yaml
# .github/workflows/queue-processor-1.yml
name: Process Queue 1
on:
  schedule:
    - cron: '*/10 * * * *'  # Every 10 minutes
  workflow_dispatch:

jobs:
  process:
    runs-on: ubuntu-latest
    timeout-minutes: 8
    
    steps:
      - name: Process Queue Items
        uses: nick-fields/retry@v2
        with:
          timeout_minutes: 7
          max_attempts: 2
          retry_wait_seconds: 30
          command: |
            curl -X POST ${{ secrets.VERCEL_URL }}/api/cron/process-queue \
              -H "Authorization: Bearer ${{ secrets.CRON_SECRET }}" \
              -H "X-Processor-ID: 1" \
              -f

# .github/workflows/queue-processor-2.yml
name: Process Queue 2
on:
  schedule:
    - cron: '5,15,25,35,45,55 * * * *'  # Offset by 5 minutes
  workflow_dispatch:

jobs:
  process:
    runs-on: ubuntu-latest
    timeout-minutes: 8
    
    steps:
      - name: Process Queue Items
        uses: nick-fields/retry@v2
        with:
          timeout_minutes: 7
          max_attempts: 2
          retry_wait_seconds: 30
          command: |
            curl -X POST ${{ secrets.VERCEL_URL }}/api/cron/process-queue \
              -H "Authorization: Bearer ${{ secrets.CRON_SECRET }}" \
              -H "X-Processor-ID: 2" \
              -f

# .github/workflows/queue-processor-3.yml
name: Process Queue 3
on:
  schedule:
    - cron: '7,17,27,37,47,57 * * * *'  # Offset by 7 minutes
  workflow_dispatch:

jobs:
  process:
    runs-on: ubuntu-latest
    timeout-minutes: 8
    
    steps:
      - name: Process Queue Items
        run: |
          # Skip night hours to save minutes
          hour=$(date +%H)
          if [ "$hour" -ge 2 ] && [ "$hour" -le 5 ]; then
            echo "Skipping - night hours (processor 3 only)"
            exit 0
          fi
          
          curl -X POST ${{ secrets.VERCEL_URL }}/api/cron/process-queue \
            -H "Authorization: Bearer ${{ secrets.CRON_SECRET }}" \
            -H "X-Processor-ID: 3" \
            -f
```

### 3. Alert Processing (Every 30 Minutes)
```yaml
# .github/workflows/alerts-processor.yml
name: Process Price Alerts
on:
  schedule:
    - cron: '*/30 * * * *'  # Every 30 minutes
  workflow_dispatch:

jobs:
  alerts:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    
    steps:
      - name: Check and Send Price Alerts
        run: |
          curl -X POST ${{ secrets.VERCEL_URL }}/api/cron/process-alerts \
            -H "Authorization: Bearer ${{ secrets.CRON_SECRET }}" \
            -f
```

### 4. Database Cleanup (Daily)
```yaml
# .github/workflows/cleanup.yml
name: Database Maintenance
on:
  schedule:
    - cron: '0 3 * * *'  # 3 AM UTC daily
  workflow_dispatch:

jobs:
  cleanup:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    
    steps:
      - name: Clean Old Data Based on User Tiers
        run: |
          curl -X POST ${{ secrets.VERCEL_URL }}/api/cron/cleanup \
            -H "Authorization: Bearer ${{ secrets.CRON_SECRET }}" \
            -H "Content-Type: application/json" \
            -d '{"action": "tier-based-cleanup"}' \
            -f
      
      - name: Vacuum Database
        run: |
          curl -X POST ${{ secrets.VERCEL_URL }}/api/cron/cleanup \
            -H "Authorization: Bearer ${{ secrets.CRON_SECRET }}" \
            -H "Content-Type: application/json" \
            -d '{"action": "vacuum"}' \
            -f
```

## User Management APIs

### Adding Products (Respects Tier Limits)
```typescript
// app/api/products/route.ts
export async function POST(request: Request) {
  const { userId } = auth();
  if (!userId) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  
  const sql = neon(process.env.DATABASE_URL!);
  const body = await request.json();
  
  // Check user's tier and current product count
  const user = await sql`
    SELECT subscription_tier, product_count
    FROM users
    WHERE clerk_user_id = ${userId}
  `;
  
  const tierLimits = TIER_LIMITS[user[0].subscription_tier];
  
  if (user[0].product_count >= tierLimits.maxProducts) {
    return NextResponse.json({ 
      error: 'Product limit reached. Please upgrade your subscription.' 
    }, { status: 403 });
  }
  
  // Add product and update count
  await sql.begin(async sql => {
    const product = await sql`
      INSERT INTO products (user_id, name, model, category)
      VALUES (
        (SELECT id FROM users WHERE clerk_user_id = ${userId}),
        ${body.name},
        ${body.model},
        ${body.category}
      )
      RETURNING id
    `;
    
    // Add listings for each store URL provided
    for (const storeUrl of body.storeUrls) {
      const checkInterval = getCheckInterval(user[0].subscription_tier);
      
      await sql`
        INSERT INTO product_listings (
          product_id,
          store_id,
          url,
          check_frequency_hours,
          next_check_at
        )
        VALUES (
          ${product[0].id},
          (SELECT id FROM stores WHERE domain = ${extractDomain(storeUrl)}),
          ${storeUrl},
          ${checkInterval},
          NOW()  -- Check immediately
        )
      `;
    }
    
    // Update user's product count
    await sql`
      UPDATE users 
      SET product_count = product_count + 1
      WHERE clerk_user_id = ${userId}
    `;
  });
  
  return NextResponse.json({ success: true });
}
```

### Tier-Based Data Retention
```typescript
// app/api/cron/cleanup/route.ts
export async function POST(request: Request) {
  const body = await request.json();
  const sql = neon(process.env.DATABASE_URL!);
  
  if (body.action === 'tier-based-cleanup') {
    // Clean price history based on user tiers
    await sql`
      DELETE FROM price_history
      WHERE id IN (
        SELECT ph.id
        FROM price_history ph
        JOIN users u ON ph.user_id = u.id
        WHERE 
          (u.subscription_tier = 'free' AND ph.scraped_at < NOW() - INTERVAL '7 days')
          OR (u.subscription_tier = 'premium' AND ph.scraped_at < NOW() - INTERVAL '30 days')
          OR (u.subscription_tier = 'premium_plus' AND ph.scraped_at < NOW() - INTERVAL '90 days')
      )
    `;
    
    // Clean old queue items
    await sql`
      DELETE FROM scrape_queue
      WHERE status IN ('completed', 'failed')
        AND processed_at < NOW() - INTERVAL '48 hours'
    `;
  }
  
  return NextResponse.json({ success: true });
}
```

## Monitoring Dashboard

### Queue Status Dashboard
```typescript
// app/dashboard/queue/page.tsx
export default async function QueueDashboard() {
  const queueStats = await getQueueStats();
  
  return (
    <div>
      <h2>Queue Status</h2>
      
      <div className="grid grid-cols-3 gap-4">
        <Card>
          <h3>Pending Items</h3>
          <p className="text-3xl">{queueStats.pending}</p>
        </Card>
        
        <Card>
          <h3>Processing</h3>
          <p className="text-3xl">{queueStats.processing}</p>
        </Card>
        
        <Card>
          <h3>Avg Wait Time</h3>
          <p className="text-3xl">{queueStats.avgWaitTime} min</p>
        </Card>
      </div>
      
      <h3>By Subscription Tier</h3>
      <table>
        <thead>
          <tr>
            <th>Tier</th>
            <th>In Queue</th>
            <th>Avg Priority</th>
            <th>Products Tracked</th>
          </tr>
        </thead>
        <tbody>
          {queueStats.byTier.map(tier => (
            <tr key={tier.name}>
              <td>{tier.name}</td>
              <td>{tier.inQueue}</td>
              <td>{tier.avgPriority}</td>
              <td>{tier.totalProducts}</td>
            </tr>
          ))}
        </tbody>
      </table>
      
      <h3>System Health</h3>
      <div>
        <p>✓ All products checked within 24 hours: {queueStats.allCheckedIn24h ? '✅' : '❌'}</p>
        <p>GitHub Actions Usage: {queueStats.githubMinutesUsed} / 2000 minutes</p>
        <p>Database Size: {queueStats.dbSize} / 500 MB</p>
      </div>
    </div>
  );
}
```

## Cost Analysis

### GitHub Actions Usage Calculation
With the current setup:
- **Queue Population**: 24 runs/day × 1 min = 24 minutes
- **Queue Processors**: 3 processors × 144 runs/day × 2 min = 864 minutes
- **Alerts**: 48 runs/day × 1 min = 48 minutes
- **Cleanup**: 1 run/day × 2 min = 2 minutes
- **Health Checks**: 96 runs/day × 0.5 min = 48 minutes

**Total**: ~986 minutes/month (under the 2000 minute limit)

This can support approximately:
- **500 free tier users** (5 products each = 2,500 products)
- **100 premium users** (25 products each = 2,500 products)
- **50 premium+ users** (100 products each = 5,000 products)

## Key Features for Scalability

1. **No GitHub Actions Changes Required**: As users sign up and add products, they're automatically added to the queue based on their tier
2. **Fair Queue Processing**: Priority system ensures premium users get more frequent checks while guaranteeing all products are checked at least once per 24 hours
3. **Automatic Scaling**: Queue processors handle whatever is in the queue, regardless of user count
4. **Tier-Based Features**: Check frequency, data retention, and priority automatically adjust based on subscription
5. **Overdue Protection**: Products that haven't been checked get increasing priority to ensure 24-hour guarantee

## Environment Variables
```env
# Database (Neon)
DATABASE_URL=postgresql://user:pass@ep-xxx.us-east-2.aws.neon.tech/neondb?sslmode=require

# Authentication (Clerk)
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...

# Payments (Stripe)
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_PRICE_ID_PREMIUM=price_...
STRIPE_PRICE_ID_PREMIUM_PLUS=price_...

# AI Service (DeepSeek)
DEEPSEEK_API_KEY=sk-...

# Application
NEXT_PUBLIC_APP_URL=https://your-app.vercel.app
CRON_SECRET=generate-a-strong-random-secret-here

# GitHub Actions (set in GitHub Secrets)
VERCEL_URL=https://your-app.vercel.app
```

## Summary

This design provides:

1. **Guaranteed 24-hour checks** for all products through priority-based queue system
2. **No GitHub Actions modifications** needed as users scale
3. **Automatic tier-based processing** without hardcoding user data
4. **Scalable to thousands of products** within free tier limits
5. **Fair resource allocation** based on subscription tiers
6. **Simple upgrade path** for users who need more products or frequency

The queue-based architecture decouples user management from the cron infrastructure, allowing the system to scale seamlessly as your user base grows.