# Design an Ad Click Aggregator

Design a real-time ad click aggregation system for analytics and billing.

## 1. Requirements

### Functional Requirements
- Track ad impressions and clicks
- Real-time click counting
- Aggregate by ad, campaign, advertiser
- Fraud detection (click fraud)
- Generate billing reports

### Non-Functional Requirements
- Handle high volume (1M clicks/sec)
- Low latency queries (<1s for real-time)
- Accuracy (critical for billing)
- Fault tolerance (no data loss)
- Scale horizontally

### Capacity Estimation

```
Clicks: 1M/sec = 86B/day
Average click event: 200 bytes
Data rate: 200 MB/sec = 17 TB/day

Storage (30 days raw):
17 TB × 30 = 510 TB

Aggregated data (much smaller):
~10 TB/month
```

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌────────────────────────────────────────────────────────┐   │
│  │                   Click Tracking                        │   │
│  │  Ad Server → Click Event → Load Balancer               │   │
│  └───────────────────────┬────────────────────────────────┘   │
│                          │                                     │
│                          ▼                                     │
│  ┌────────────────────────────────────────────────────────┐   │
│  │                     Kafka                               │   │
│  │  (Partitioned by ad_id for ordering)                   │   │
│  └───────────────────────┬────────────────────────────────┘   │
│                          │                                     │
│           ┌──────────────┼──────────────┐                     │
│           │              │              │                     │
│           ▼              ▼              ▼                     │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐     │
│  │   Fraud     │ │   Stream    │ │   Raw Event         │     │
│  │  Detection  │ │ Aggregation │ │   Storage           │     │
│  │  (Flink)    │ │   (Flink)   │ │   (S3/HDFS)        │     │
│  └─────────────┘ └──────┬──────┘ └─────────────────────┘     │
│                         │                                     │
│                         ▼                                     │
│  ┌────────────────────────────────────────────────────────┐   │
│  │                 Aggregation Store                       │   │
│  │  ┌────────────────┐  ┌────────────────┐               │   │
│  │  │ Real-time      │  │ Historical     │               │   │
│  │  │ (Redis/Druid)  │  │ (ClickHouse)   │               │   │
│  │  └────────────────┘  └────────────────┘               │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. Click Event Schema

```json
{
  "event_id": "uuid",
  "event_type": "click",  // or "impression"
  "timestamp": 1640000000000,
  "ad_id": "ad_123",
  "campaign_id": "camp_456",
  "advertiser_id": "adv_789",
  "publisher_id": "pub_001",
  "user_id": "user_hash",
  "device_id": "device_hash",
  "ip_address": "192.168.1.1",
  "user_agent": "Mozilla/5.0...",
  "geo": {
    "country": "US",
    "region": "CA",
    "city": "San Francisco"
  },
  "referer": "https://publisher.com/page",
  "bid_price": 0.50,
  "click_position": 2
}
```

## 4. Stream Processing

```
┌────────────────────────────────────────────────────────────────┐
│                Stream Processing (Flink)                       │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Aggregation Windows:                                    │  │
│  │                                                          │  │
│  │  1. Tumbling Window (1 minute)                          │  │
│  │     - Count clicks per ad per minute                    │  │
│  │     - No overlap, exact counts                          │  │
│  │                                                          │  │
│  │  2. Sliding Window (5 min window, 1 min slide)         │  │
│  │     - Moving average for trend detection               │  │
│  │                                                          │  │
│  │  3. Session Window                                       │  │
│  │     - Group user activity sessions                      │  │
│  │     - For user behavior analysis                        │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Aggregation dimensions:                                       │
│  - ad_id                                                       │
│  - campaign_id                                                 │
│  - advertiser_id                                               │
│  - publisher_id                                                │
│  - geo (country, region)                                      │
│  - device_type                                                 │
│  - time bucket (minute, hour, day)                           │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Flink Job Implementation

```python
# Apache Flink pseudocode
class ClickAggregator:
    def process(self, stream):
        # Parse events
        parsed = stream.map(self.parse_event)

        # Filter fraud
        clean = parsed.filter(self.fraud_filter)

        # Aggregate by multiple dimensions
        by_ad_minute = clean \
            .key_by(lambda e: e['ad_id']) \
            .window(TumblingEventTimeWindows.of(Time.minutes(1))) \
            .aggregate(ClickCounter())

        by_campaign_minute = clean \
            .key_by(lambda e: e['campaign_id']) \
            .window(TumblingEventTimeWindows.of(Time.minutes(1))) \
            .aggregate(ClickCounter())

        by_advertiser_hour = clean \
            .key_by(lambda e: e['advertiser_id']) \
            .window(TumblingEventTimeWindows.of(Time.hours(1))) \
            .aggregate(ClickCounter())

        # Write to different sinks
        by_ad_minute.add_sink(RedisSink())
        by_campaign_minute.add_sink(RedisSink())
        by_advertiser_hour.add_sink(ClickHouseSink())

class ClickCounter(AggregateFunction):
    def create_accumulator(self):
        return {'clicks': 0, 'impressions': 0, 'spend': 0}

    def add(self, event, acc):
        if event['event_type'] == 'click':
            acc['clicks'] += 1
            acc['spend'] += event['bid_price']
        else:
            acc['impressions'] += 1
        return acc

    def get_result(self, acc):
        acc['ctr'] = acc['clicks'] / max(acc['impressions'], 1)
        return acc
```

## 5. Fraud Detection

```
┌────────────────────────────────────────────────────────────────┐
│                   Fraud Detection                              │
│                                                                │
│  Types of click fraud:                                        │
│  1. Bot clicks (automated)                                    │
│  2. Click farms                                               │
│  3. Competitor sabotage                                       │
│  4. Publisher fraud                                           │
│                                                                │
│  Detection signals:                                            │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  1. IP-based                                             │  │
│  │     - Too many clicks from same IP                      │  │
│  │     - Known bot/proxy IP ranges                         │  │
│  │                                                          │  │
│  │  2. Behavior-based                                       │  │
│  │     - Click without impression                          │  │
│  │     - Click too fast after impression                   │  │
│  │     - No mouse movement (bots)                          │  │
│  │     - Identical timing patterns                         │  │
│  │                                                          │  │
│  │  3. Device-based                                         │  │
│  │     - Too many clicks from same device                  │  │
│  │     - Suspicious device fingerprint                     │  │
│  │                                                          │  │
│  │  4. Geographic                                           │  │
│  │     - IP location vs timezone mismatch                  │  │
│  │     - Impossible travel speed                           │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Fraud Detection Implementation

```python
class FraudDetector:
    def __init__(self, redis):
        self.redis = redis
        self.thresholds = {
            'clicks_per_ip_hour': 10,
            'clicks_per_device_hour': 5,
            'min_time_since_impression': 0.5,  # seconds
        }

    async def is_fraudulent(self, event):
        """Check if click event is fraudulent"""
        signals = await self.gather_signals(event)

        # Rule-based checks
        if signals['clicks_per_ip_hour'] > self.thresholds['clicks_per_ip_hour']:
            return True, 'ip_velocity'

        if signals['clicks_per_device_hour'] > self.thresholds['clicks_per_device_hour']:
            return True, 'device_velocity'

        if signals['time_since_impression'] < self.thresholds['min_time_since_impression']:
            return True, 'too_fast'

        if self.is_known_bot_ip(event['ip_address']):
            return True, 'known_bot'

        # ML model for complex patterns
        fraud_score = self.ml_model.predict(signals)
        if fraud_score > 0.9:
            return True, 'ml_detected'

        return False, None

    async def gather_signals(self, event):
        ip = event['ip_address']
        device = event['device_id']
        hour_bucket = event['timestamp'] // 3600000

        return {
            'clicks_per_ip_hour': await self.redis.incr(f"ip:{ip}:{hour_bucket}"),
            'clicks_per_device_hour': await self.redis.incr(f"device:{device}:{hour_bucket}"),
            'time_since_impression': self.get_impression_delay(event),
            'ip_country': self.geoip.lookup(ip),
            'timezone_match': self.check_timezone(event)
        }
```

## 6. Data Storage

```sql
-- ClickHouse schema for historical data
CREATE TABLE click_aggregates (
    date Date,
    hour UInt8,
    minute UInt8,
    ad_id String,
    campaign_id String,
    advertiser_id String,
    publisher_id String,
    country String,
    device_type String,
    impressions UInt64,
    clicks UInt64,
    spend Decimal64(4),
    ctr Float64
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (advertiser_id, campaign_id, ad_id, date, hour);

-- Pre-aggregated tables for fast queries
CREATE MATERIALIZED VIEW daily_campaign_stats
ENGINE = SummingMergeTree()
ORDER BY (advertiser_id, campaign_id, date)
AS SELECT
    date,
    campaign_id,
    advertiser_id,
    sum(impressions) as impressions,
    sum(clicks) as clicks,
    sum(spend) as spend
FROM click_aggregates
GROUP BY date, campaign_id, advertiser_id;
```

## 7. Query Service

```python
class AnalyticsService:
    def __init__(self, redis, clickhouse):
        self.redis = redis  # Real-time data
        self.clickhouse = clickhouse  # Historical data

    async def get_realtime_stats(self, ad_id, window_minutes=60):
        """Get real-time stats from Redis"""
        now = int(time.time() * 1000)
        buckets = []

        for i in range(window_minutes):
            bucket_time = now - (i * 60000)
            bucket_key = f"stats:{ad_id}:{bucket_time // 60000}"
            buckets.append(bucket_key)

        # Multi-get from Redis
        results = await self.redis.mget(buckets)

        total = {'impressions': 0, 'clicks': 0, 'spend': 0}
        for r in results:
            if r:
                data = json.loads(r)
                total['impressions'] += data['impressions']
                total['clicks'] += data['clicks']
                total['spend'] += data['spend']

        return total

    async def get_historical_stats(self, advertiser_id, start_date, end_date):
        """Get historical stats from ClickHouse"""
        query = """
            SELECT
                date,
                sum(impressions) as impressions,
                sum(clicks) as clicks,
                sum(spend) as spend,
                clicks / impressions as ctr
            FROM click_aggregates
            WHERE advertiser_id = %(advertiser_id)s
              AND date BETWEEN %(start)s AND %(end)s
            GROUP BY date
            ORDER BY date
        """

        return await self.clickhouse.execute(query, {
            'advertiser_id': advertiser_id,
            'start': start_date,
            'end': end_date
        })
```

## 8. Exactly-Once Processing

```
┌────────────────────────────────────────────────────────────────┐
│              Exactly-Once Semantics                            │
│                                                                │
│  Critical for billing accuracy                                │
│                                                                │
│  Challenges:                                                   │
│  - Network failures cause retries                             │
│  - Stream processor crashes                                   │
│  - Duplicate events from clients                              │
│                                                                │
│  Solutions:                                                     │
│  1. Event deduplication                                       │
│     - Use event_id to detect duplicates                       │
│     - Store processed IDs in Redis (with TTL)                │
│                                                                │
│  2. Checkpointing (Flink)                                     │
│     - Periodically save state                                 │
│     - On failure, replay from checkpoint                     │
│                                                                │
│  3. Transactional sinks                                       │
│     - Kafka transactions                                       │
│     - Two-phase commit                                        │
│                                                                │
│  4. Idempotent writes                                         │
│     - Use upsert instead of insert                           │
│     - Include event_id in aggregate key                      │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 9. Key Takeaways

1. **Kafka for ingestion** - handles high volume, provides durability
2. **Stream processing** - Flink for real-time aggregation
3. **Lambda architecture** - real-time + batch for accuracy
4. **Multi-level aggregation** - minute/hour/day rollups
5. **Fraud detection** - rules + ML, real-time filtering
6. **Exactly-once** - critical for billing accuracy

---

Next: [Task Scheduler](../11-task-scheduler/README.md)
