# Common Mistakes

Avoid these pitfalls to perform your best in system design interviews.

## Mistake 1: Jumping to Solution

### The Problem
Starting to design before understanding requirements.

### What It Looks Like
```
Interviewer: "Design a URL shortener"
Candidate: "I'll use MD5 to hash the URL and store
it in Redis..."
```

### Better Approach
```
Interviewer: "Design a URL shortener"
Candidate: "Before I start, let me clarify a few things.
What's the expected scale? Should URLs expire? Do we
need analytics?"
```

### Why It Matters
- You might solve the wrong problem
- You miss constraints that affect the design
- You look like you don't think before acting

---

## Mistake 2: Not Drawing Diagrams

### The Problem
Talking without visual aids.

### What It Looks Like
"So there's a load balancer that connects to servers,
and those connect to a database, and there's also a
cache somewhere..."

### Better Approach
```
┌────────┐    ┌────────┐    ┌──────┐
│ Client │───▶│   LB   │───▶│Server│───▶ [Cache] ───▶ [DB]
└────────┘    └────────┘    └──────┘
```

### Why It Matters
- Visual aids help communication
- Shows organization skills
- Easier to reference later

---

## Mistake 3: Ignoring Scale

### The Problem
Designing for 1,000 users when the requirement is 1 billion.

### What It Looks Like
```
"I'll use a single PostgreSQL database."
(For a system expecting 100M daily users)
```

### Better Approach
```
"Given 100M users, a single database won't work.
I'll shard by user_id across multiple instances,
use read replicas, and cache heavily."
```

### Why It Matters
- Scale changes everything
- Single points of failure become critical
- Shows you understand distributed systems

---

## Mistake 4: Over-Engineering

### The Problem
Adding unnecessary complexity from the start.

### What It Looks Like
```
"We'll use Kubernetes with service mesh, event sourcing
with CQRS, and blockchain for audit trail..."
(For a simple CRUD application)
```

### Better Approach
```
"Let's start simple with a monolith and standard
relational database. As we scale, we can add
caching, then consider splitting services if needed."
```

### Why It Matters
- Simple systems are easier to understand and debug
- Complexity should be justified by requirements
- Shows practical engineering judgment

---

## Mistake 5: Single-Minded Focus

### The Problem
Spending all time on one component.

### What It Looks Like
```
(45-minute interview)
- 30 minutes: Discussing database schema in detail
- 0 minutes: APIs, caching, scaling, failure modes
```

### Better Approach
```
- 5 min: Requirements
- 5 min: High-level design (all components)
- 20 min: Deep dive (2-3 key areas)
- 10 min: Trade-offs and alternatives
- 5 min: Buffer and questions
```

### Why It Matters
- Interviewers want to see breadth AND depth
- Missing major components is a red flag
- Shows poor time management

---

## Mistake 6: Not Handling Failures

### The Problem
Assuming everything always works.

### What It Looks Like
```
"The server receives the request, writes to database,
and returns success."
(No mention of what happens if database is down)
```

### Better Approach
```
"If the database write fails, we retry with exponential
backoff. After 3 retries, we return an error to the client
and log for investigation. We also have a replica that
can be promoted if the primary fails."
```

### Why It Matters
- Real systems fail constantly
- Failure handling often reveals design issues
- Shows operational awareness

---

## Mistake 7: Not Taking Feedback

### The Problem
Ignoring hints or pushback from the interviewer.

### What It Looks Like
```
Interviewer: "What about the case where..."
Candidate: "Yeah, but as I was saying, we'll use..."
```

### Better Approach
```
Interviewer: "What about the case where..."
Candidate: "Good point! I hadn't considered that.
Let me think... We could handle it by..."
```

### Why It Matters
- Interview is collaborative
- Interviewer may be guiding you
- Shows you can work with others

---

## Mistake 8: Memorized Answers

### The Problem
Reciting a prepared answer without adaptation.

### What It Looks Like
```
Candidate: (Giving same Twitter design regardless of
different requirements mentioned by interviewer)
```

### Better Approach
Listen to the specific requirements and adapt your design.
"Since you mentioned we need strong consistency for this
feature, I'll use a different approach here..."

### Why It Matters
- Every interview has unique constraints
- Shows flexibility and understanding
- Generic answers are obvious

---

## Mistake 9: Poor Communication

### The Problem
Not explaining your thinking clearly.

### What It Looks Like
```
(Long pause)
"OK so... um... I guess we need... uh... a database."
```

### Better Approach
```
"Let me think about the storage layer. We have
relational data with transactions, so I'm considering
PostgreSQL. Let me draw it out..."
```

### Why It Matters
- Interviewers can't read minds
- Silent thinking is fine, but announce it
- Clear communication is part of the job

---

## Mistake 10: Not Knowing Basics

### The Problem
Missing fundamental knowledge.

### Common Gaps
- How DNS works
- Difference between SQL and NoSQL
- What a load balancer does
- HTTP vs WebSocket
- Basic Big O complexity

### Solution
Review the fundamentals before the interview. See [Module 1](../../01-fundamentals/README.md) and [Module 2](../../02-core-concepts/README.md).

---

## Quick Checklist

Before ending your interview, ask yourself:

- [ ] Did I clarify requirements?
- [ ] Did I estimate scale?
- [ ] Did I draw a diagram?
- [ ] Did I define APIs?
- [ ] Did I discuss database choice?
- [ ] Did I consider caching?
- [ ] Did I address scaling?
- [ ] Did I discuss failure handling?
- [ ] Did I explain trade-offs?
- [ ] Did I leave time for questions?

---

## Final Tips

1. **Practice out loud** - Explaining is different from thinking
2. **Time yourself** - 45 minutes goes fast
3. **Accept feedback gracefully** - It's collaborative
4. **Stay calm** - It's OK to think before speaking
5. **Be yourself** - Authenticity matters

---

[Back to Module 6](../README.md) | [Back to Course](../../README.md)
