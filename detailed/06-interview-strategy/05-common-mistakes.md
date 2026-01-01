# Common Mistakes

Pitfalls to avoid during your system design interview.

---

## Overview

```
┌─────────────────────────────────────────────────────────────┐
│                Categories of Mistakes                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Requirements Phase Mistakes                             │
│  2. Design Phase Mistakes                                   │
│  3. Communication Mistakes                                  │
│  4. Technical Mistakes                                      │
│  5. Time Management Mistakes                                │
│                                                              │
│  Each mistake includes:                                     │
│  • What goes wrong                                          │
│  • Why it's a problem                                       │
│  • How to fix it                                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 1. Requirements Phase Mistakes

```
┌─────────────────────────────────────────────────────────────┐
│               Requirements Mistakes                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  MISTAKE: Jumping straight into design                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ "Let me start with the database..."                │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • You might design the wrong thing                    │ │
│  │  • Wastes time backtracking                            │ │
│  │  • Shows lack of real-world experience                 │ │
│  │                                                         │ │
│  │  ✓ Fix: "Before I start, let me clarify a few          │ │
│  │     requirements..."                                    │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  MISTAKE: Assuming scale without asking                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ "I'll assume we have 1 billion users..."           │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • Over-engineering if scale is smaller                │ │
│  │  • Under-engineering if scale is larger                │ │
│  │  • Shows you don't gather requirements in real life    │ │
│  │                                                         │ │
│  │  ✓ Fix: "What scale should I design for? How many      │ │
│  │     users, requests per second?"                       │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  MISTAKE: Asking too many obvious questions               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ "Should users be able to view their profile?"      │ │
│  │  ❌ "Do we need a database?"                           │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • Wastes precious interview time                      │ │
│  │  • Makes you seem inexperienced                        │ │
│  │                                                         │ │
│  │  ✓ Fix: Ask about non-obvious decisions and trade-offs │ │
│  │     "Should the feed be chronological or ranked?"      │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  MISTAKE: Taking on too much scope                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ "I'll also add payments, recommendations,          │ │
│  │      notifications, and admin dashboard..."            │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • Can't go deep on anything                           │ │
│  │  • Spreads yourself too thin                           │ │
│  │                                                         │ │
│  │  ✓ Fix: "Let me focus on the core features and         │ │
│  │     mention others at the end if time permits."        │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Design Phase Mistakes

```
┌─────────────────────────────────────────────────────────────┐
│                  Design Mistakes                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  MISTAKE: Going too deep too early                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ Spending 20 minutes on database schema before      │ │
│  │     showing the overall architecture                   │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • May run out of time for other components            │ │
│  │  • Interviewer doesn't see big picture                 │ │
│  │  • May be optimizing wrong area                        │ │
│  │                                                         │ │
│  │  ✓ Fix: Start broad, then go deep                      │ │
│  │     "Let me first show the high-level architecture,    │ │
│  │      then we can dive into specific areas."            │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  MISTAKE: Not addressing obvious bottlenecks               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ Single database for 1M QPS                         │ │
│  │  ❌ No caching for read-heavy workload                 │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • Shows lack of scaling knowledge                     │ │
│  │  • Design won't work at stated scale                   │ │
│  │                                                         │ │
│  │  ✓ Fix: Proactively identify and address bottlenecks  │ │
│  │     "With 100K QPS, a single DB won't work. I'll add   │ │
│  │      caching and read replicas..."                     │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  MISTAKE: Over-engineering from the start                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ "We'll need Kubernetes, service mesh, event        │ │
│  │      sourcing, CQRS, and multi-region from day 1"      │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • Adds unnecessary complexity                         │ │
│  │  • May not be needed for the scale                     │ │
│  │  • Shows poor judgment                                 │ │
│  │                                                         │ │
│  │  ✓ Fix: Start simple, evolve based on requirements    │ │
│  │     "Let's start with a simpler architecture and      │ │
│  │      I'll show how it evolves as we scale."           │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  MISTAKE: Not considering failure modes                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ Designing only the happy path                      │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • Real systems fail                                   │ │
│  │  • Shows lack of production experience                 │ │
│  │                                                         │ │
│  │  ✓ Fix: Proactively mention failure handling          │ │
│  │     "If this service fails, we have retries and       │ │
│  │      fallback to cached data..."                      │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  MISTAKE: Choosing technologies you can't explain          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ "I'll use Cassandra" (but can't explain why)       │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • Interviewer will ask follow-up questions            │ │
│  │  • Shows superficial knowledge                         │ │
│  │                                                         │ │
│  │  ✓ Fix: Only use technologies you can justify         │ │
│  │     "I'll use Cassandra because we need high write    │ │
│  │      throughput and can tolerate eventual consistency" │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Communication Mistakes

```
┌─────────────────────────────────────────────────────────────┐
│               Communication Mistakes                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  MISTAKE: Silent thinking                                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ *Stares at whiteboard for 2 minutes*               │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • Interviewer can't evaluate your thinking            │ │
│  │  • Awkward silence                                     │ │
│  │  • Can't receive helpful hints                         │ │
│  │                                                         │ │
│  │  ✓ Fix: Think out loud                                 │ │
│  │     "Let me think about this... I'm considering        │ │
│  │      whether to use push or pull for the feed..."      │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  MISTAKE: Ignoring interviewer hints                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ Interviewer: "What about celebrity users?"         │ │
│  │     Candidate: "Anyway, so the database..."            │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • Missing important design considerations             │ │
│  │  • Appears inflexible                                  │ │
│  │  • Ignoring help being offered                         │ │
│  │                                                         │ │
│  │  ✓ Fix: Acknowledge and address hints                  │ │
│  │     "Great point! Celebrity users would need special   │ │
│  │      handling. Let me address that..."                 │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  MISTAKE: Being defensive when challenged                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ "No, my approach is correct!"                      │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • Shows poor collaboration skills                     │ │
│  │  • Closes off discussion                               │ │
│  │  • Interviewer may have valid point                    │ │
│  │                                                         │ │
│  │  ✓ Fix: Be open to feedback                           │ │
│  │     "That's a fair challenge. Let me reconsider...    │ │
│  │      You might be right that X would work better."    │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  MISTAKE: Not checking in with interviewer                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ *30 minutes of uninterrupted monologue*            │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • Might be going wrong direction                      │ │
│  │  • Not collaborative                                   │ │
│  │  • Interviewer disengaged                              │ │
│  │                                                         │ │
│  │  ✓ Fix: Check in every 5-7 minutes                    │ │
│  │     "Does this make sense? Any questions before I     │ │
│  │      continue to the next component?"                  │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Technical Mistakes

```
┌─────────────────────────────────────────────────────────────┐
│                 Technical Mistakes                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  MISTAKE: Single points of failure                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ Design:                                            │ │
│  │  Client → Server → Database (all single instances)     │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • Any failure takes down the system                   │ │
│  │  • Not production-ready                                │ │
│  │                                                         │ │
│  │  ✓ Fix: Add redundancy at every layer                 │ │
│  │     Load balancer, multiple servers, DB replicas      │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  MISTAKE: Wrong database for use case                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ Using MongoDB for financial transactions           │ │
│  │  ❌ Using MySQL for high-write IoT data                │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • Shows poor technical judgment                       │ │
│  │  • Design won't work in practice                       │ │
│  │                                                         │ │
│  │  ✓ Fix: Match database to requirements                │ │
│  │     SQL: ACID, complex queries, relationships         │ │
│  │     NoSQL: Scale, flexibility, specific access patterns│ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  MISTAKE: Ignoring network latency                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ "We'll call 5 services synchronously for each     │ │
│  │      request, it'll be fast"                          │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • 5 × 50ms = 250ms minimum                           │ │
│  │  • Real latency is often higher                       │ │
│  │                                                         │ │
│  │  ✓ Fix: Consider latency in your design               │ │
│  │     "I'll parallelize these calls to reduce latency   │ │
│  │      from 250ms to about 50ms + overhead"             │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  MISTAKE: Not considering data consistency                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ "We'll update the cache and database separately"   │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • Race conditions                                     │ │
│  │  • Data can get out of sync                            │ │
│  │                                                         │ │
│  │  ✓ Fix: Choose a consistency strategy                 │ │
│  │     "We'll use cache-aside with invalidation to       │ │
│  │      avoid the double-write race condition"           │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  MISTAKE: Forgetting about hotspots                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ "We'll shard by user ID" (for celebrity system)    │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • One shard handles celebrity's millions of followers │ │
│  │  • Uneven load distribution                            │ │
│  │                                                         │ │
│  │  ✓ Fix: Consider hot key handling                     │ │
│  │     "For hot users, we'll replicate their data        │ │
│  │      across multiple shards"                          │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Time Management Mistakes

```
┌─────────────────────────────────────────────────────────────┐
│               Time Management Mistakes                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  MISTAKE: Spending too long on requirements                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ 15 minutes asking questions                        │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • Less time for actual design                         │ │
│  │  • May seem like stalling                              │ │
│  │                                                         │ │
│  │  ✓ Fix: 5 minutes max for requirements                │ │
│  │     Focus on key clarifications, not exhaustive list  │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  MISTAKE: Getting stuck on one component                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ 25 minutes designing perfect database schema       │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • Other critical components not covered               │ │
│  │  • Looks like tunnel vision                            │ │
│  │                                                         │ │
│  │  ✓ Fix: Cover breadth first, then depth               │ │
│  │     "I could go deeper here, but let me first show    │ │
│  │      the full architecture..."                        │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  MISTAKE: No time for wrap-up                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ Interview ends mid-sentence                        │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • No chance to summarize strengths                    │ │
│  │  • Design feels incomplete                             │ │
│  │  • Missed opportunity to show improvements             │ │
│  │                                                         │ │
│  │  ✓ Fix: Watch the clock, leave 3-5 minutes            │ │
│  │     "We have 5 minutes left. Let me summarize and     │ │
│  │      mention potential improvements..."               │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  MISTAKE: Not adapting when behind schedule                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ❌ Continuing at same pace when running out of time   │ │
│  │                                                         │ │
│  │  Why it's bad:                                          │ │
│  │  • Won't cover essential components                    │ │
│  │  • Shows poor time management                          │ │
│  │                                                         │ │
│  │  ✓ Fix: Acknowledge and adjust                        │ │
│  │     "I notice we're running short on time. Let me     │ │
│  │      quickly cover the remaining parts at high level" │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Quick Checklist

```
┌─────────────────────────────────────────────────────────────┐
│              Pre-Interview Checklist                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Requirements:                                              │
│  □ Ask about scale and users                               │
│  □ Clarify scope (what's in/out)                           │
│  □ Understand latency/availability needs                   │
│  □ Keep to 5 minutes                                       │
│                                                              │
│  Design:                                                    │
│  □ Start with high-level architecture                      │
│  □ Address obvious bottlenecks                             │
│  □ Consider failure modes                                  │
│  □ Only use technologies you can explain                   │
│                                                              │
│  Communication:                                             │
│  □ Think out loud                                          │
│  □ Check in with interviewer                               │
│  □ Respond to hints positively                             │
│  □ Be open to feedback                                     │
│                                                              │
│  Technical:                                                 │
│  □ No single points of failure                             │
│  □ Appropriate database choices                            │
│  □ Consider latency and consistency                        │
│  □ Handle hotspots                                         │
│                                                              │
│  Time:                                                      │
│  □ Watch the clock                                         │
│  □ Cover breadth before depth                              │
│  □ Leave time for wrap-up                                  │
│  □ Adapt if running behind                                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

[← Back to Module](./README.md) | [Next: Practice Problems →](./06-practice-problems.md)
