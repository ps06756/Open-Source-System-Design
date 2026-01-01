# Design a Notification System

Design a scalable notification system that supports push notifications, SMS, and email.

## 1. Requirements

### Functional Requirements
- Send push notifications (iOS, Android, Web)
- Send SMS notifications
- Send email notifications
- Support for scheduled notifications
- User notification preferences
- Rate limiting per user
- Notification history/inbox

### Non-Functional Requirements
- High availability
- Soft real-time delivery (<30s for push, <1min for SMS)
- At-least-once delivery
- Scale: 10M notifications/day
- Pluggable providers (Twilio, SendGrid, FCM, APNs)

### Capacity Estimation

```
Notifications: 10M/day ≈ 115/sec
Peak: 1000/sec (assume 10x average)

Distribution:
- Push: 70% = 7M/day
- Email: 25% = 2.5M/day
- SMS: 5% = 500K/day

Storage:
- 10M × 365 = 3.65B notifications/year
- 3.65B × 200 bytes = 730 GB/year
```

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌────────────┐     ┌──────────────────────────────────────┐  │
│  │  Services  │────▶│         Notification Service          │  │
│  │ (Triggers) │     │                                        │  │
│  └────────────┘     │  ┌──────────────────────────────────┐ │  │
│                     │  │        Validation & Routing       │ │  │
│                     │  └──────────────────────────────────┘ │  │
│                     └─────────────────┬────────────────────┘  │
│                                       │                        │
│                            ┌──────────┴──────────┐            │
│                            │    Message Queue    │            │
│                            │      (Kafka)        │            │
│                            └──────────┬──────────┘            │
│                                       │                        │
│           ┌───────────────────────────┼───────────────────┐   │
│           │                           │                   │   │
│           ▼                           ▼                   ▼   │
│    ┌─────────────┐           ┌─────────────┐      ┌──────────┐│
│    │Push Workers │           │Email Workers│      │SMS Workers││
│    └──────┬──────┘           └──────┬──────┘      └─────┬────┘│
│           │                         │                    │     │
│           ▼                         ▼                    ▼     │
│    ┌─────────────┐           ┌─────────────┐      ┌──────────┐│
│    │ FCM / APNs  │           │  SendGrid   │      │  Twilio  ││
│    └─────────────┘           └─────────────┘      └──────────┘│
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. API Design

```
POST /api/notifications
{
    "user_id": "123",
    "template_id": "order_shipped",
    "channels": ["push", "email"],
    "data": {
        "order_id": "ORD-456",
        "tracking_url": "https://..."
    },
    "scheduled_at": "2024-01-15T10:00:00Z",  // optional
    "priority": "high"  // high, normal, low
}

Response:
{
    "notification_id": "notif_789",
    "status": "queued",
    "scheduled_at": "2024-01-15T10:00:00Z"
}

GET /api/notifications/{user_id}
Response:
{
    "notifications": [
        {
            "id": "notif_789",
            "title": "Order Shipped",
            "body": "Your order ORD-456 has shipped",
            "read": false,
            "created_at": "2024-01-15T10:00:00Z"
        }
    ]
}
```

## 4. Component Details

### Notification Service

```python
class NotificationService:
    def send(self, request):
        # 1. Validate request
        self.validate(request)

        # 2. Get user preferences
        preferences = self.get_preferences(request.user_id)

        # 3. Filter channels based on preferences
        channels = self.filter_channels(request.channels, preferences)

        # 4. Rate limit check
        if not self.rate_limiter.allow(request.user_id):
            raise RateLimitExceeded()

        # 5. Build notification for each channel
        for channel in channels:
            notification = self.build_notification(request, channel)

            # 6. Queue for delivery
            if request.scheduled_at:
                self.scheduler.schedule(notification, request.scheduled_at)
            else:
                self.queue.publish(channel, notification)

        # 7. Store in database
        self.store(request, channels)

        return notification_id
```

### Template System

```
┌────────────────────────────────────────────────────────┐
│                 Template System                        │
│                                                        │
│  Templates stored in database:                        │
│                                                        │
│  {                                                     │
│    "template_id": "order_shipped",                    │
│    "channels": {                                       │
│      "push": {                                         │
│        "title": "Order Shipped!",                     │
│        "body": "Your order {{order_id}} is on its way"│
│      },                                                │
│      "email": {                                        │
│        "subject": "Your order has shipped",           │
│        "html_template": "order_shipped.html"          │
│      },                                                │
│      "sms": {                                          │
│        "body": "Order {{order_id}} shipped. Track: {{url}}"│
│      }                                                 │
│    }                                                   │
│  }                                                     │
│                                                        │
│  Benefits:                                             │
│  - Consistent messaging across channels               │
│  - Non-engineers can update copy                     │
│  - A/B testing support                                │
│  - Localization                                        │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 5. Push Notification Worker

```python
class PushWorker:
    def __init__(self):
        self.fcm = FCMClient()
        self.apns = APNsClient()

    async def process(self, notification):
        user_id = notification['user_id']

        # Get user's device tokens
        devices = await self.get_devices(user_id)

        results = []
        for device in devices:
            try:
                if device['platform'] == 'ios':
                    result = await self.apns.send(
                        device['token'],
                        notification['title'],
                        notification['body'],
                        notification['data']
                    )
                elif device['platform'] == 'android':
                    result = await self.fcm.send(
                        device['token'],
                        notification['title'],
                        notification['body'],
                        notification['data']
                    )

                results.append({'device': device['id'], 'status': 'sent'})

            except InvalidTokenError:
                # Remove invalid token
                await self.remove_device(device['id'])

            except Exception as e:
                # Retry later
                results.append({'device': device['id'], 'status': 'failed'})

        # Update notification status
        await self.update_status(notification['id'], results)
```

### Device Token Management

```sql
CREATE TABLE user_devices (
    device_id VARCHAR(64) PRIMARY KEY,
    user_id BIGINT NOT NULL,
    platform VARCHAR(20),  -- ios, android, web
    token TEXT NOT NULL,
    app_version VARCHAR(20),
    created_at TIMESTAMP,
    last_active_at TIMESTAMP,
    INDEX idx_user_id (user_id)
);
```

## 6. Email Worker

```python
class EmailWorker:
    def __init__(self):
        self.providers = [
            SendGridProvider(weight=70),
            SESProvider(weight=30)
        ]

    async def process(self, notification):
        user = await self.get_user(notification['user_id'])
        template = await self.get_template(notification['template_id'])

        # Render email
        html = self.render(template['html'], notification['data'])

        # Select provider (weighted random for load balancing)
        provider = self.select_provider()

        try:
            await provider.send(
                to=user['email'],
                subject=template['subject'],
                html=html
            )
            await self.update_status(notification['id'], 'delivered')

        except ProviderError as e:
            # Fallback to another provider
            fallback = self.get_fallback_provider(provider)
            await fallback.send(to=user['email'], subject=template['subject'], html=html)
```

## 7. Priority Queue System

```
┌────────────────────────────────────────────────────────┐
│               Priority Queues                          │
│                                                        │
│  Kafka Topics by Priority:                            │
│                                                        │
│  notifications.push.high                              │
│  ├── OTP codes                                        │
│  ├── Security alerts                                  │
│  └── Payment confirmations                            │
│                                                        │
│  notifications.push.normal                            │
│  ├── Order updates                                    │
│  ├── Social interactions                             │
│  └── Content recommendations                          │
│                                                        │
│  notifications.push.low                               │
│  ├── Marketing                                        │
│  ├── Weekly digests                                   │
│  └── Feature announcements                            │
│                                                        │
│  Workers consume high priority first:                 │
│  - high: 50% of workers                              │
│  - normal: 35% of workers                            │
│  - low: 15% of workers                               │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 8. User Preferences

```sql
CREATE TABLE notification_preferences (
    user_id BIGINT PRIMARY KEY,
    email_enabled BOOLEAN DEFAULT true,
    push_enabled BOOLEAN DEFAULT true,
    sms_enabled BOOLEAN DEFAULT true,
    quiet_hours_start TIME,
    quiet_hours_end TIME,
    frequency_cap INT DEFAULT 50,  -- per day
    categories JSONB  -- {"marketing": false, "social": true}
);
```

```python
def filter_channels(user_id, requested_channels, category):
    prefs = get_preferences(user_id)

    # Check global toggles
    channels = []
    for channel in requested_channels:
        if channel == 'email' and prefs['email_enabled']:
            channels.append('email')
        elif channel == 'push' and prefs['push_enabled']:
            channels.append('push')
        elif channel == 'sms' and prefs['sms_enabled']:
            channels.append('sms')

    # Check category preference
    if category in prefs['categories']:
        if not prefs['categories'][category]:
            return []  # User disabled this category

    # Check quiet hours
    if is_quiet_hours(prefs):
        channels = [c for c in channels if c != 'push']

    return channels
```

## 9. Rate Limiting

```python
class NotificationRateLimiter:
    def __init__(self, redis):
        self.redis = redis
        self.limits = {
            'push': {'limit': 100, 'window': 86400},  # 100/day
            'sms': {'limit': 5, 'window': 86400},      # 5/day
            'email': {'limit': 50, 'window': 86400}    # 50/day
        }

    def allow(self, user_id, channel):
        key = f"rate:{user_id}:{channel}"
        config = self.limits[channel]

        current = self.redis.incr(key)
        if current == 1:
            self.redis.expire(key, config['window'])

        return current <= config['limit']
```

## 10. Analytics & Monitoring

```
┌────────────────────────────────────────────────────────┐
│               Metrics to Track                         │
│                                                        │
│  Delivery metrics:                                    │
│  - Sent count by channel                             │
│  - Delivery rate (delivered/sent)                    │
│  - Failure rate by provider                          │
│  - Latency (time from request to delivery)           │
│                                                        │
│  Engagement metrics:                                  │
│  - Open rate (email, push)                           │
│  - Click-through rate                                 │
│  - Unsubscribe rate                                   │
│                                                        │
│  System metrics:                                      │
│  - Queue depth                                        │
│  - Worker throughput                                  │
│  - Provider API latency                              │
│                                                        │
│  Alerting:                                            │
│  - Delivery rate < 95%                               │
│  - Queue depth > threshold                           │
│  - Provider failure rate spike                       │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 11. Key Takeaways

1. **Decouple with message queues** - async processing for reliability
2. **Multiple providers** - fallback prevents single point of failure
3. **Priority queues** - urgent notifications delivered first
4. **Template system** - consistent, maintainable messages
5. **User preferences** - respect user choices, reduce spam
6. **Rate limiting** - protect users from notification fatigue

---

Next: [News Feed](../05-news-feed/README.md)
