# Module 6: Interview Strategy

Master the art of system design interviews with proven frameworks and techniques.

## Chapters

| # | Chapter | Focus |
|---|---------|-------|
| 1 | [Framework & Approach](./01-framework-approach.md) | Structured methodology for any design |
| 2 | [Requirements Gathering](./02-requirements-gathering.md) | Asking the right questions |
| 3 | [Estimation Techniques](./03-estimation-techniques.md) | Back-of-envelope calculations |
| 4 | [Communication Tips](./04-communication-tips.md) | Effective interviewer collaboration |
| 5 | [Common Mistakes](./05-common-mistakes.md) | Pitfalls to avoid |
| 6 | [Practice Problems](./06-practice-problems.md) | Self-assessment exercises |
| 7 | [Mock Interview Guide](./07-mock-interview-guide.md) | Simulating real interviews |

---

## The System Design Interview

```
┌─────────────────────────────────────────────────────────────┐
│              What Interviewers Evaluate                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Problem Exploration (20%)                          │ │
│  │     • Clarify ambiguous requirements                   │ │
│  │     • Ask relevant questions                           │ │
│  │     • Identify constraints                             │ │
│  │                                                         │ │
│  │  2. Technical Design (40%)                             │ │
│  │     • Propose reasonable architecture                  │ │
│  │     • Make sound trade-offs                            │ │
│  │     • Handle scale appropriately                       │ │
│  │                                                         │ │
│  │  3. Communication (20%)                                │ │
│  │     • Explain clearly                                  │ │
│  │     • Collaborate with interviewer                     │ │
│  │     • Structure thoughts                               │ │
│  │                                                         │ │
│  │  4. Depth & Breadth (20%)                              │ │
│  │     • Dive deep when asked                             │ │
│  │     • Show broad knowledge                             │ │
│  │     • Discuss alternatives                             │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Time Management

```
45-Minute Interview Breakdown:

┌────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  0        5       10       20       35       40       45 min       │
│  │────────│────────│────────│────────│────────│────────│          │
│  │ Require│ High   │        │        │ Deep   │ Q&A    │          │
│  │ ments  │ Level  │   Core Components        │ Dive   │          │
│  │        │ Design │                          │        │          │
│  │────────│────────│──────────────────────────│────────│          │
│                                                                     │
│  Phase 1: Requirements (5 min)                                     │
│  • Clarify scope and constraints                                   │
│  • Define functional requirements                                  │
│  • Identify non-functional requirements                            │
│                                                                     │
│  Phase 2: High-Level Design (5 min)                                │
│  • Draw main components                                            │
│  • Show data flow                                                  │
│  • Get interviewer buy-in                                          │
│                                                                     │
│  Phase 3: Core Components (15-20 min)                              │
│  • Design key services                                             │
│  • Define data models                                              │
│  • Address scale challenges                                        │
│                                                                     │
│  Phase 4: Deep Dive (5-10 min)                                     │
│  • Interviewer-directed exploration                                │
│  • Handle edge cases                                               │
│  • Discuss trade-offs                                              │
│                                                                     │
│  Phase 5: Wrap-up (5 min)                                          │
│  • Summarize design                                                │
│  • Mention improvements                                            │
│  • Answer questions                                                │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

---

## Difficulty Levels by Company

```
┌─────────────────────────────────────────────────────────────┐
│                 Company Expectations                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Junior (0-2 years):                                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  • Basic CRUD applications                             │ │
│  │  • Simple scaling concepts                             │ │
│  │  • URL shortener, Pastebin level                       │ │
│  │  • May not have system design round                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Mid-level (2-5 years):                                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  • Complete service design                             │ │
│  │  • Caching, database choices                           │ │
│  │  • Twitter, Instagram level                            │ │
│  │  • Expected to drive the discussion                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Senior (5+ years):                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  • Complex distributed systems                         │ │
│  │  • Deep trade-off discussions                          │ │
│  │  • Google Search, Payment Systems                      │ │
│  │  • Expected to lead and mentor                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Staff+ (8+ years):                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  • Organization-wide systems                           │ │
│  │  • Multi-team coordination                             │ │
│  │  • Technical strategy                                  │ │
│  │  • Novel problem solving                               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Principles

```
┌─────────────────────────────────────────────────────────────┐
│                    Success Principles                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Start Simple, Evolve                                    │
│     ┌──────────────────────────────────────────────────┐   │
│     │  MVP → Handle Scale → Add Features → Optimize    │   │
│     │                                                   │   │
│     │  Don't jump to complex solutions immediately     │   │
│     │  Show evolution of thought                       │   │
│     └──────────────────────────────────────────────────┘   │
│                                                              │
│  2. Trade-offs, Not Perfect Solutions                       │
│     ┌──────────────────────────────────────────────────┐   │
│     │  Every choice has pros and cons                  │   │
│     │  Explain WHY you chose an approach               │   │
│     │  "We could do X for consistency or Y for speed"  │   │
│     └──────────────────────────────────────────────────┘   │
│                                                              │
│  3. Numbers Matter                                          │
│     ┌──────────────────────────────────────────────────┐   │
│     │  Back up decisions with estimates                │   │
│     │  "100M users × 10 requests/day = 1B requests"    │   │
│     │  Use math to justify architecture                │   │
│     └──────────────────────────────────────────────────┘   │
│                                                              │
│  4. Think Out Loud                                          │
│     ┌──────────────────────────────────────────────────┐   │
│     │  Interviewer can only evaluate what you say      │   │
│     │  Verbalize your thought process                  │   │
│     │  "I'm considering X because..."                  │   │
│     └──────────────────────────────────────────────────┘   │
│                                                              │
│  5. Be Collaborative                                        │
│     ┌──────────────────────────────────────────────────┐   │
│     │  Interview is a conversation, not a test         │   │
│     │  Ask for feedback: "Does this make sense?"       │   │
│     │  Adapt to hints and guidance                     │   │
│     └──────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Study Roadmap

```
Week 1-2: Fundamentals
├── Scalability basics
├── Database fundamentals
├── Caching concepts
└── Practice: URL Shortener, Pastebin

Week 3-4: Core Components
├── Message queues
├── Load balancing
├── CDNs and storage
└── Practice: Twitter, Instagram

Week 5-6: Advanced Patterns
├── Distributed systems
├── Consistency models
├── Design patterns
└── Practice: YouTube, Uber

Week 7-8: Interview Prep
├── Mock interviews
├── Time management
├── Communication practice
└── Practice: Google Search, Payment System
```

---

[Next: Framework & Approach →](./01-framework-approach.md)
