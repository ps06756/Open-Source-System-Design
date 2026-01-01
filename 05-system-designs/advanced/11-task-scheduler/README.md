# Design a Distributed Task Scheduler

Design a distributed task scheduler like Airflow, Celery, or cron at scale.

## 1. Requirements

### Functional Requirements
- Schedule tasks (one-time and recurring)
- Execute tasks reliably
- Support task dependencies (DAGs)
- Retry failed tasks
- Prioritization
- Monitoring and alerting

### Non-Functional Requirements
- High availability (no missed schedules)
- Scalability (millions of tasks/day)
- Exactly-once execution
- Low scheduling latency (<1s delay)
- Fault tolerance

### Capacity Estimation

```
Tasks: 10M executions/day ≈ 115 tasks/sec
Scheduled tasks: 1M active schedules
Task metadata: 1M × 1 KB = 1 GB
Execution history: 10M × 500 bytes × 30 days = 150 GB
```

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌────────────────────────────────────────────────────────┐   │
│  │                        Clients                          │   │
│  │  (API, CLI, Web UI)                                     │   │
│  └───────────────────────┬────────────────────────────────┘   │
│                          │                                     │
│                          ▼                                     │
│  ┌────────────────────────────────────────────────────────┐   │
│  │                   Scheduler Service                     │   │
│  │  - Manages schedules                                    │   │
│  │  - Triggers tasks at scheduled time                    │   │
│  │  - Leader election for HA                              │   │
│  └───────────────────────┬────────────────────────────────┘   │
│                          │                                     │
│                          ▼                                     │
│  ┌────────────────────────────────────────────────────────┐   │
│  │                    Task Queue                           │   │
│  │  (Redis/Kafka - prioritized, partitioned)              │   │
│  └───────────────────────┬────────────────────────────────┘   │
│                          │                                     │
│           ┌──────────────┼──────────────┐                     │
│           │              │              │                     │
│           ▼              ▼              ▼                     │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐             │
│  │  Worker 1   │ │  Worker 2   │ │  Worker N   │             │
│  │  (Executor) │ │  (Executor) │ │  (Executor) │             │
│  └─────────────┘ └─────────────┘ └─────────────┘             │
│                                                                │
│  ┌────────────────────────────────────────────────────────┐   │
│  │                   Storage Layer                         │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │   │
│  │  │ PostgreSQL  │  │    Redis    │  │   S3/Logs   │    │   │
│  │  │ (Metadata)  │  │  (State)    │  │   (Output)  │    │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘    │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. Task Definition

```python
# Task definition
class TaskDefinition:
    task_id: str
    name: str
    task_type: str  # 'http', 'python', 'shell', 'container'
    payload: dict   # Task-specific configuration
    schedule: str   # Cron expression or interval
    timeout: int    # Seconds
    retries: int
    retry_delay: int
    priority: int   # 1-10
    dependencies: List[str]  # Task IDs that must complete first
    concurrency_key: str  # For limiting concurrent executions

# Example task
{
    "task_id": "daily_report",
    "name": "Generate Daily Report",
    "task_type": "http",
    "payload": {
        "url": "https://api.internal/generate-report",
        "method": "POST",
        "body": {"date": "{{execution_date}}"}
    },
    "schedule": "0 6 * * *",  # 6 AM daily
    "timeout": 3600,
    "retries": 3,
    "retry_delay": 300,
    "priority": 5
}
```

## 4. Scheduler Service

```
┌────────────────────────────────────────────────────────────────┐
│                    Scheduler Service                           │
│                                                                │
│  Responsibilities:                                             │
│  1. Maintain schedule registry                                │
│  2. Calculate next execution times                            │
│  3. Trigger tasks at scheduled time                           │
│  4. Handle dependencies                                        │
│                                                                │
│  High Availability:                                            │
│  - Multiple scheduler instances                               │
│  - Leader election via distributed lock                       │
│  - Only leader schedules, followers on standby               │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Scheduler Algorithm:                                    │  │
│  │                                                          │  │
│  │  while True:                                             │  │
│  │      now = current_time()                               │  │
│  │                                                          │  │
│  │      # Get tasks due for execution                      │  │
│  │      due_tasks = db.query(                              │  │
│  │          "SELECT * FROM tasks                           │  │
│  │           WHERE next_run <= ? AND status = 'scheduled'" │  │
│  │          , now)                                         │  │
│  │                                                          │  │
│  │      for task in due_tasks:                             │  │
│  │          if check_dependencies(task):                   │  │
│  │              enqueue(task)                              │  │
│  │              update_next_run(task)                      │  │
│  │                                                          │  │
│  │      sleep(1)  # Check every second                     │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Scheduler Implementation

```python
class Scheduler:
    def __init__(self, db, queue, lock_service):
        self.db = db
        self.queue = queue
        self.lock = lock_service
        self.is_leader = False

    async def run(self):
        """Main scheduler loop"""
        while True:
            # Try to acquire leadership
            self.is_leader = await self.lock.acquire('scheduler-leader', ttl=30)

            if self.is_leader:
                await self.schedule_due_tasks()

            await asyncio.sleep(1)

    async def schedule_due_tasks(self):
        """Find and enqueue due tasks"""
        now = datetime.utcnow()

        # Get tasks that are due
        due_tasks = await self.db.query('''
            SELECT * FROM scheduled_tasks
            WHERE next_run_at <= $1
              AND status = 'scheduled'
            ORDER BY priority DESC, next_run_at ASC
            LIMIT 1000
            FOR UPDATE SKIP LOCKED
        ''', now)

        for task in due_tasks:
            # Check dependencies
            if not await self.dependencies_satisfied(task):
                continue

            # Create execution record
            execution = TaskExecution(
                id=generate_id(),
                task_id=task.id,
                scheduled_at=task.next_run_at,
                status='pending'
            )
            await self.db.insert('task_executions', execution)

            # Enqueue for worker
            await self.queue.enqueue(
                queue=f"tasks:priority:{task.priority}",
                message={
                    'execution_id': execution.id,
                    'task_id': task.id,
                    'payload': task.payload
                }
            )

            # Update next run time
            next_run = self.calculate_next_run(task.schedule)
            await self.db.update('scheduled_tasks', task.id, {
                'next_run_at': next_run,
                'last_run_at': now
            })

    def calculate_next_run(self, schedule):
        """Calculate next run time from cron expression"""
        cron = croniter(schedule, datetime.utcnow())
        return cron.get_next(datetime)
```

## 5. Task Queue

```
┌────────────────────────────────────────────────────────────────┐
│                      Task Queue                                │
│                                                                │
│  Priority queues:                                              │
│  - tasks:priority:10 (critical)                              │
│  - tasks:priority:5  (normal)                                 │
│  - tasks:priority:1  (low)                                    │
│                                                                │
│  Workers pull from highest priority first                     │
│                                                                │
│  Delayed/retry queue:                                          │
│  - Redis sorted set with execution time as score             │
│  - ZRANGEBYSCORE to get due tasks                            │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Redis implementation:                                   │  │
│  │                                                          │  │
│  │  # Enqueue immediate task                               │  │
│  │  LPUSH tasks:priority:5 {task_json}                     │  │
│  │                                                          │  │
│  │  # Enqueue delayed task                                  │  │
│  │  ZADD tasks:delayed {execute_at_timestamp} {task_json}  │  │
│  │                                                          │  │
│  │  # Worker pulls task                                     │  │
│  │  BRPOP tasks:priority:10 tasks:priority:5 tasks:priority:1│ │
│  │                                                          │  │
│  │  # Move delayed to ready                                 │  │
│  │  ZRANGEBYSCORE tasks:delayed 0 {now} LIMIT 100         │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 6. Worker (Executor)

```python
class Worker:
    def __init__(self, queue, db, task_handlers):
        self.queue = queue
        self.db = db
        self.handlers = task_handlers

    async def run(self):
        """Main worker loop"""
        while True:
            # Pull task from queue (blocking)
            task = await self.queue.dequeue(
                queues=['tasks:priority:10', 'tasks:priority:5', 'tasks:priority:1'],
                timeout=30
            )

            if task:
                await self.execute(task)

    async def execute(self, task):
        """Execute a task"""
        execution_id = task['execution_id']

        # Update status to running
        await self.db.update('task_executions', execution_id, {
            'status': 'running',
            'started_at': datetime.utcnow(),
            'worker_id': self.worker_id
        })

        try:
            # Get handler for task type
            handler = self.handlers[task['type']]

            # Execute with timeout
            result = await asyncio.wait_for(
                handler.execute(task['payload']),
                timeout=task.get('timeout', 3600)
            )

            # Mark success
            await self.db.update('task_executions', execution_id, {
                'status': 'completed',
                'completed_at': datetime.utcnow(),
                'result': result
            })

        except asyncio.TimeoutError:
            await self.handle_failure(execution_id, 'timeout', task)

        except Exception as e:
            await self.handle_failure(execution_id, str(e), task)

    async def handle_failure(self, execution_id, error, task):
        """Handle task failure with retry logic"""
        execution = await self.db.get('task_executions', execution_id)

        if execution['retry_count'] < task.get('retries', 3):
            # Schedule retry
            retry_delay = task.get('retry_delay', 60) * (2 ** execution['retry_count'])

            await self.queue.enqueue_delayed(
                task,
                delay_seconds=retry_delay
            )

            await self.db.update('task_executions', execution_id, {
                'status': 'retrying',
                'error': error,
                'retry_count': execution['retry_count'] + 1
            })
        else:
            # Max retries exceeded
            await self.db.update('task_executions', execution_id, {
                'status': 'failed',
                'error': error,
                'completed_at': datetime.utcnow()
            })
```

## 7. DAG (Dependency) Support

```
┌────────────────────────────────────────────────────────────────┐
│                  DAG Execution                                 │
│                                                                │
│  Task dependencies form a Directed Acyclic Graph              │
│                                                                │
│  Example:                                                      │
│         ┌───────┐                                             │
│         │ Task A│                                             │
│         └───┬───┘                                             │
│         ┌───┴───┐                                             │
│         │       │                                             │
│     ┌───▼───┐   │                                             │
│     │ Task B│   │                                             │
│     └───┬───┘   │                                             │
│         │   ┌───▼───┐                                         │
│         │   │ Task C│                                         │
│         │   └───┬───┘                                         │
│         └───┬───┘                                             │
│         ┌───▼───┐                                             │
│         │ Task D│                                             │
│         └───────┘                                             │
│                                                                │
│  D depends on B and C                                         │
│  B depends on A                                               │
│  C depends on A                                               │
│                                                                │
│  Execution:                                                    │
│  1. Start A                                                   │
│  2. When A completes → Start B and C (parallel)              │
│  3. When B and C complete → Start D                          │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### DAG Implementation

```python
class DAGScheduler:
    async def execute_dag(self, dag_id, dag_run_id):
        """Execute a DAG"""
        dag = await self.db.get('dags', dag_id)
        tasks = dag['tasks']

        # Build dependency graph
        graph = self.build_graph(tasks)

        # Find tasks with no dependencies (starting points)
        ready = [t for t in tasks if not t['dependencies']]

        # Track completed tasks
        completed = set()

        while len(completed) < len(tasks):
            # Execute ready tasks in parallel
            executing = []
            for task in ready:
                if task['id'] not in completed:
                    execution = self.create_execution(task, dag_run_id)
                    executing.append(execution)

            # Wait for any to complete
            done = await self.wait_any(executing)

            for execution in done:
                if execution.status == 'completed':
                    completed.add(execution.task_id)

                    # Find newly ready tasks
                    for task in tasks:
                        if task['id'] not in completed:
                            deps_met = all(
                                d in completed
                                for d in task['dependencies']
                            )
                            if deps_met and task not in ready:
                                ready.append(task)
                else:
                    # Handle failure
                    await self.handle_dag_failure(dag_run_id, execution)
                    return

        await self.mark_dag_complete(dag_run_id)
```

## 8. Database Schema

```sql
-- Task definitions
CREATE TABLE tasks (
    id UUID PRIMARY KEY,
    name VARCHAR(255),
    task_type VARCHAR(50),
    payload JSONB,
    schedule VARCHAR(100),  -- Cron expression
    timeout INT,
    retries INT DEFAULT 3,
    retry_delay INT DEFAULT 60,
    priority INT DEFAULT 5,
    concurrency_key VARCHAR(100),
    enabled BOOLEAN DEFAULT true,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- Scheduled instances
CREATE TABLE scheduled_tasks (
    id UUID PRIMARY KEY,
    task_id UUID REFERENCES tasks(id),
    next_run_at TIMESTAMP NOT NULL,
    last_run_at TIMESTAMP,
    status VARCHAR(20) DEFAULT 'scheduled',
    INDEX idx_next_run (next_run_at, status)
);

-- Execution history
CREATE TABLE task_executions (
    id UUID PRIMARY KEY,
    task_id UUID,
    scheduled_at TIMESTAMP,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    status VARCHAR(20),
    worker_id VARCHAR(100),
    retry_count INT DEFAULT 0,
    result JSONB,
    error TEXT,
    INDEX idx_task_status (task_id, status),
    INDEX idx_scheduled (scheduled_at)
);

-- DAGs
CREATE TABLE dags (
    id UUID PRIMARY KEY,
    name VARCHAR(255),
    tasks JSONB,  -- Array of task configs with dependencies
    schedule VARCHAR(100),
    enabled BOOLEAN DEFAULT true
);
```

## 9. Key Takeaways

1. **Leader election** - only one scheduler triggers tasks
2. **Priority queues** - critical tasks execute first
3. **At-least-once** - retry with idempotent tasks
4. **DAG support** - complex workflow dependencies
5. **Distributed workers** - scale execution horizontally
6. **Observability** - track every execution

---

Next: [Stock Exchange](../12-stock-exchange/README.md)
