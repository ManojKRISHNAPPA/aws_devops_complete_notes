# AWS EC2 Pricing Models: On-Demand vs Reserved vs Spot vs Dedicated

Imagine you are booking rooms in a hotel. Sometimes you need a room immediately, sometimes you book for a year, sometimes you want a cheap room that might become unavailable, and sometimes you want the entire space for yourself.

AWS EC2 pricing works in a similar way!

## 1. On-Demand Instances — Walk-in Customer 🏨

**Funny example:**

You enter a hotel and tell the receptionist:

"I need a room right now. I don't know whether I'll stay for one hour or three days!"

The receptionist gives you a room, and you pay for the time you use it.

**In AWS:**
- Pay for compute capacity as you use it.
- No long-term commitment.
- Flexible for changing workloads.
- Usually more expensive than discounted options for comparable usage.

**Real-time example:**

You are learning Kubernetes and want to launch an EC2 instance for testing. You use it for a few hours and stop it when you're done.

**Best for:** Development, testing, unpredictable workloads and short-term projects.

**Remember:** "I need it now, and I don't want a long-term commitment!"

---

## 2. Reserved Instances — Long-Term Hotel Membership 📅

**Funny example:**

You visit the same hotel every day. You tell the manager:

"I'll keep coming for the next one or three years. Give me a discount!"

The hotel offers you a better deal because you commit for the long term.

**In AWS:**
- Commit to a term of one or three years.
- Receive discounted pricing for eligible usage.
- Less flexible than simply using On-Demand pricing.
- Standard Reserved Instances have specific modification and exchange rules.

**Real-time example:**

Your company runs a production application 24/7, and you expect its compute requirements to remain relatively stable. A Reserved Instance may reduce costs for eligible usage.

**Best for:** Predictable, steady workloads.

**Remember:** "I'll commit for the long term if you give me a discount!"

**Important:** EC2 Reserved Instances are primarily a billing discount commitment. They are not necessarily a physical server reserved exclusively for you.

---

## 3. Spot Instances — Last-Minute Discounted Hotel Room 💸

**Funny example:**

A hotel has 100 rooms, but only 40 are occupied.

The manager announces:

"Empty rooms are available at a huge discount! But if a VIP arrives and we need the room back, your stay may be interrupted."

You accept the low price because you can manage without the room for a while.

**In AWS:**
- Use spare AWS EC2 capacity at potentially steep discounts.
- AWS can interrupt your instance when it needs the capacity back.
- Interruption notices are generally two minutes, when a notice is provided.
- You should design your workload to tolerate interruptions.

**Real-time example:**

Your Jenkins pipeline launches temporary EC2 build workers. If a worker is interrupted, Jenkins can retry the build on another worker.

**Best for:**
- Batch processing
- CI/CD build workers
- Data processing
- Fault-tolerant workloads
- Flexible background jobs

**Not ideal for:** Critical workloads that cannot tolerate interruptions.

**Remember:** "Give me the cheapest room. If I get kicked out, I'll manage!"

---

## 4. Dedicated Instances — Private Hotel Space 🔐

**Funny example:**

You enter a hotel and tell the manager:

"I don't want to share my physical space with other customers. I want my own dedicated space!"

The hotel arranges exclusive physical accommodation for you.

**In AWS:**
- Dedicated Instances run on hardware dedicated to a single customer.
- Other customers' instances do not share that physical server.
- They may suit certain isolation, compliance or licensing requirements.
- They generally cost more than comparable shared-tenancy options.

**Real-time example:**

A company has a workload that requires dedicated physical hardware because of its isolation or licensing requirements.

**Best for:** Workloads requiring dedicated hardware tenancy.

**Remember:** "I want my own physical space. No sharing!"

**Important:** Dedicated Instances do not necessarily mean you control the entire physical server. For greater host-level control, consider Dedicated Hosts.

---

## 5. Quick Comparison Table

| Feature | On-Demand | Reserved | Spot | Dedicated |
|---|---|---|---|---|
| Payment model | Pay as you use | Commit for 1 or 3 years for pricing benefits | Discounted spare capacity | Dedicated hardware tenancy |
| Cost | Standard pricing | Often lower for eligible usage | Often the cheapest | Usually higher |
| Long-term commitment | No | Yes | No long-term commitment required | Depends on option |
| Interruption risk | No normal Spot-style reclamation | No normal Spot-style reclamation | Yes | No normal Spot-style reclamation |
| Flexibility | High | Lower | High, but interruptions matter | Depends on configuration |
| Best use case | Testing and variable workloads | Stable workloads | Retryable jobs | Isolation and compliance |

*Note: Dedicated is a tenancy choice, unlike On-Demand, Reserved and Spot purchasing models. Dedicated Instances can use On-Demand pricing, and Dedicated Hosts have their own billing options.*

---

## 6. Real-Time DevOps Example: Running 20 Microservices

Imagine your company runs a GameApp with 20 microservices on AWS.

### Production API
Use On-Demand capacity initially, or a suitable commitment-based discount when usage becomes predictable.

### Stable production compute
Consider Reserved Instances or Savings Plans for eligible, steady usage.

### Jenkins build workers
Use Spot Instances for retryable builds to reduce costs, with fallback capacity if needed.

### Workload requiring physical isolation
Consider Dedicated Instances or Dedicated Hosts, depending on your requirements.

---

## 7. Interview Question 🎯

**Question:** Your Jenkins pipeline launches temporary EC2 workers to build Docker images. The workers do not need to run continuously, and a failed build can be retried. Which EC2 pricing option would you consider?

**Answer:** Spot Instances.

**Why?** They can offer substantial discounts because they use spare AWS capacity. Since the builds can be retried, the pipeline can tolerate interruptions.

## 8. Easy Way to Remember

- **On-Demand:** "I need it now!"
- **Reserved:** "I'll keep coming for years!"
- **Spot:** "Give me the cheap room; I'll manage if it becomes unavailable!"
- **Dedicated:** "I want dedicated physical hardware!"

**One-line summary:**

On-Demand provides flexibility, Reserved provides commitment-based savings, Spot provides low-cost spare capacity with interruption risk, and Dedicated provides physical hardware isolation.
