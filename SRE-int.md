# SRE Interview Questions & Answers
### Site Reliability Engineering | 8+ Years Experience | Easy English

---

## Table of Contents

1. [Core SRE Concepts](#1-core-sre-concepts)
2. [SLI, SLO, SLA & Error Budgets](#2-sli-slo-sla--error-budgets)
3. [Incident Management](#3-incident-management)
4. [Monitoring & Observability](#4-monitoring--observability)
5. [On-Call & Alerting](#5-on-call--alerting)
6. [Reliability & Fault Tolerance](#6-reliability--fault-tolerance)
7. [CI/CD & Change Management](#7-cicd--change-management)
8. [Capacity Planning & Performance](#8-capacity-planning--performance)
9. [Linux & Systems](#9-linux--systems)
10. [Kubernetes & Cloud](#10-kubernetes--cloud)
11. [Toil & Automation](#11-toil--automation)
12. [Scenario-Based Questions](#12-scenario-based-questions)

---

## 1. Core SRE Concepts

---

**Q1. What is SRE and how is it different from DevOps?**

**A:**
SRE (Site Reliability Engineering) was started by Google. It applies **software engineering practices to operations work**.

| SRE | DevOps |
|---|---|
| Specific role with clear rules (error budgets, SLOs) | A culture and mindset |
| Uses code to solve ops problems | Encourages dev and ops to work together |
| Focuses on reliability and reducing toil | Focuses on speed of delivery |
| Born at Google | Born from the community |

In simple words: **SRE is one way to implement DevOps**. DevOps is the idea, SRE is the practice.

---

**Q2. What are the main responsibilities of an SRE?**

**A:**
An SRE is responsible for:

1. **Availability** — keep services running and healthy
2. **Latency** — make sure services respond fast
3. **Performance** — systems work well under load
4. **Efficiency** — use resources wisely (cost)
5. **Change management** — deploy safely without breaking things
6. **Monitoring** — know when something is wrong before users do
7. **Capacity planning** — make sure we have enough resources for future growth
8. **Incident response** — fix things fast when they break
9. **Reducing toil** — automate repetitive manual work

---

**Q3. What is the 50% rule in SRE?**

**A:**
Google's SRE rule: SREs should spend **maximum 50% of their time on operations work** (on-call, incidents, manual tasks).

The other **50% must be on engineering work** (automation, improving reliability, building tools).

If ops work goes above 50%, the team stops being SRE and becomes a traditional ops team. The solution is to automate or push work back to the development team.

---

**Q4. What is toil in SRE?**

**A:**
Toil is work that is:
- **Manual** — a human has to do it
- **Repetitive** — same task over and over
- **No lasting value** — doing it doesn't improve anything permanently
- **Grows with traffic** — as the system gets bigger, more toil is needed

Examples of toil:
- Manually restarting a service every night
- Copying data between systems by hand
- Manually approving deployments one by one

SREs aim to **automate toil away**. The goal is to keep toil below 50% of total work.

---

**Q5. What is the SRE book and why is it important?**

**A:**
The **SRE Book** (full name: "Site Reliability Engineering: How Google Runs Production Systems") was published by Google in 2016. It is freely available online.

It is important because:
- It defines the SRE role and practices
- It introduced concepts like error budgets, SLOs, postmortems
- It became the industry standard reference for reliability engineering
- Most companies building SRE teams use it as their foundation

There is also the **SRE Workbook** (2018) which has more practical examples.

---

## 2. SLI, SLO, SLA & Error Budgets

---

**Q6. What is the difference between SLI, SLO, and SLA?**

**A:**

| Term | Full Name | What it means | Example |
|---|---|---|---|
| **SLI** | Service Level Indicator | A metric that measures how the service is doing | 99.5% of requests succeed |
| **SLO** | Service Level Objective | The target you set for your SLI | We want 99.9% success rate |
| **SLA** | Service Level Agreement | Legal contract with customers, with penalties | We guarantee 99.9% or you get a refund |

Simple way to remember:
- **SLI** = what you measure
- **SLO** = what you aim for
- **SLA** = what you promise (with money at stake)

---

**Q7. What is an Error Budget and how do you use it?**

**A:**
Error budget = **how much downtime/errors you are allowed** based on your SLO.

Formula:
```
Error Budget = 100% - SLO target

Example:
SLO = 99.9% availability
Error Budget = 100% - 99.9% = 0.1%

In a 30-day month:
0.1% of 43,200 minutes = 43.2 minutes of allowed downtime
```

How SREs use it:
- If budget is **healthy** → team can move fast, deploy more often
- If budget is **running low** → slow down deployments, focus on reliability
- If budget is **exhausted** → freeze deployments until the month resets

Error budget is a **negotiation tool** between SRE (reliability) and Dev (features).

---

**Q8. What are good examples of SLIs for a web application?**

**A:**
Common SLIs by category:

| Category | SLI Example |
|---|---|
| **Availability** | % of HTTP requests that return 2xx or 3xx |
| **Latency** | % of requests completed in under 200ms |
| **Error rate** | % of requests that do NOT return 5xx errors |
| **Throughput** | Requests per second the system handles |
| **Durability** | % of data written that can be read back correctly |

Good SLIs measure what the **user actually experiences**, not just if the server is up.

---

**Q9. What is a good SLO target? Should we always aim for 100%?**

**A:**
No. **100% availability is the wrong target** because:
- It is impossible to achieve
- It costs too much to even try
- It leaves no error budget for deployments and changes
- Users often cannot tell the difference between 99.9% and 100%

Good targets depend on the service:
- **Internal tools:** 99% (87.6 hours downtime/year is acceptable)
- **Customer-facing web app:** 99.9% (8.76 hours/year)
- **Payment service:** 99.99% (52 minutes/year)
- **Life-critical systems:** 99.999% (5 minutes/year)

The key question: **what reliability do users actually need?**

---

**Q10. How do you set an SLO for a new service with no historical data?**

**A:**
For a new service I use this approach:

1. **Talk to users / stakeholders** — what reliability do they expect?
2. **Look at similar services** — what do competitors offer?
3. **Start conservative** — set a lower SLO first (e.g., 99.5%)
4. **Measure for 1-2 months** — see what the service actually achieves
5. **Adjust** — tighten or loosen the SLO based on data
6. **Never set SLO above what you can actually deliver**

Starting with a looser SLO is better than setting a tight one and immediately breaching it.

---

## 3. Incident Management

---

**Q11. Walk me through your incident response process.**

**A:**
My incident response follows these steps:

**1. Detect** — alert fires, user reports, or monitoring catches the issue

**2. Acknowledge** — person on-call claims the incident (stops alert spam)

**3. Triage** — how bad is it? Set severity (P1/P2/P3)

**4. Communicate** — notify stakeholders, open incident channel in Slack

**5. Investigate** — find the cause using logs, metrics, traces

**6. Mitigate** — stop the bleeding fast (rollback, restart, redirect traffic)
> Note: Mitigation ≠ fix. You restore service first, then find root cause.

**7. Resolve** — confirm service is fully back to normal

**8. Postmortem** — within 48-72 hours, write a blameless postmortem

---

**Q12. What is a blameless postmortem and why is it important?**

**A:**
A postmortem is a document written **after an incident** to understand what happened and prevent it from happening again.

**Blameless** means: we do not blame individuals. We understand that:
- People make mistakes under pressure
- The system should prevent single person mistakes from causing outages
- Fear of blame stops people from reporting issues early

A good postmortem includes:
- **Timeline** — what happened, when
- **Root cause** — the real reason, not just symptoms
- **Impact** — how many users affected, for how long
- **What went well** — what helped us recover
- **What went wrong** — what slowed us down
- **Action items** — concrete tasks to prevent recurrence (with owners and due dates)

---

**Q13. What are incident severity levels and how do you decide them?**

**A:**
Typical severity definitions:

| Level | Name | Meaning | Example |
|---|---|---|---|
| **P1 / SEV1** | Critical | Full outage, all users affected | Website completely down |
| **P2 / SEV2** | Major | Partial outage or major feature broken | Payments failing for 30% of users |
| **P3 / SEV3** | Minor | Small impact, workaround exists | Report generation is slow |
| **P4 / SEV4** | Low | No user impact, background issue | Log pipeline delayed by 10 minutes |

How I decide: I ask — **how many users are affected and what can they not do?**

---

**Q14. What is MTTD, MTTR, MTTF, and MTBF?**

**A:**

| Metric | Full Name | What it measures |
|---|---|---|
| **MTTD** | Mean Time To Detect | How long to notice an incident |
| **MTTR** | Mean Time To Recover | How long to fix and restore service |
| **MTTF** | Mean Time To Failure | Average time a system runs before failing |
| **MTBF** | Mean Time Between Failures | Average time between two failures |

As an SRE I focus most on:
- **Lowering MTTD** — better alerting, dashboards
- **Lowering MTTR** — runbooks, automation, clear on-call process

---

**Q15. How do you handle an incident where you don't know what is wrong?**

**A:**
When I have no idea what's wrong, I follow this structure:

1. **Look at recent changes** — what deployed in the last 2 hours? (Most outages are caused by recent changes)
2. **Check the obvious** — is it the entire service or one component? Is it one region or all regions?
3. **Divide and conquer** — isolate the problem layer by layer (load balancer → app → database)
4. **Check metrics** — CPU, memory, latency, error rate — find where the spike is
5. **Check logs** — search for ERROR or FATAL around the time the issue started
6. **Ask for help** — bring in another engineer if stuck for more than 15-20 minutes
7. **Rollback first** — if a recent deploy is suspected, rollback first and investigate later

The most important rule: **restore service first, then find root cause**.

---

## 4. Monitoring & Observability

---

**Q16. What is Observability and how is it different from Monitoring?**

**A:**

| Monitoring | Observability |
|---|---|
| Watching known metrics | Ability to understand any internal state from outside |
| Tells you WHAT is broken | Helps you understand WHY it is broken |
| You define dashboards upfront | You can ask any question about the system |
| Good for known failure modes | Good for unknown / new failures |

Simple way to think about it:
- **Monitoring** = "Is the service up? Is CPU high?"
- **Observability** = "Why is this one user's request taking 10 seconds?"

Observability is built on **3 pillars**:
1. **Metrics** — numbers over time (CPU, RPS, error rate)
2. **Logs** — text records of events
3. **Traces** — path of a single request through the system

---

**Q17. What are the three pillars of observability?**

**A:**

**1. Metrics**
- Numbers collected over time
- Good for: dashboards, alerts, trends
- Tools: Prometheus, CloudWatch, Datadog
- Example: `http_requests_total{status="500"} = 42`

**2. Logs**
- Text records of what happened
- Good for: debugging, finding exact error messages
- Tools: ELK Stack, CloudWatch Logs, Loki
- Example: `ERROR 2024-01-15 14:32:01 - Database connection timeout`

**3. Traces**
- Track one request through all services
- Good for: finding slow parts in microservices
- Tools: Jaeger, Zipkin, AWS X-Ray, Datadog APM
- Example: Request took 2s — 1.8s was in the database query

---

**Q18. What is the RED method and USE method for monitoring?**

**A:**

**RED Method** (for services / APIs):
- **R**ate — how many requests per second?
- **E**rrors — how many requests are failing?
- **D**uration — how long do requests take?

**USE Method** (for infrastructure / resources):
- **U**tilization — what % of the resource is being used? (CPU 80%)
- **S**aturation — is there a queue building up? (disk I/O wait)
- **E**rrors — are there hardware or system errors?

I use **RED for application-level** monitoring and **USE for server/infra-level** monitoring.

---

**Q19. How do you set up a good alert? What makes an alert bad?**

**A:**
A **good alert**:
- Is actionable — tells you exactly what to do
- Has clear severity
- Points to a runbook
- Fires only when a human needs to act
- Has enough context (which service, which region, impact)

A **bad alert**:
- Fires too often (alert fatigue)
- Is not actionable — no clear next step
- Fires for things that fix themselves
- No runbook attached
- Wakes people up at 3am for a non-critical issue

Golden rules for alerting:
1. **Alert on symptoms, not causes** — "users can't log in" not "CPU is at 80%"
2. **Every alert should have a runbook**
3. **If an alert fires and you do nothing, delete it or fix it**
4. **Reduce noise aggressively** — alert fatigue kills on-call effectiveness

---

**Q20. What is the difference between a dashboard and an alert?**

**A:**
- **Dashboard** — for humans to look at proactively, tells the story of system health
- **Alert** — pushes a notification to a human when immediate action is needed

You should not rely on dashboards for incident detection — that requires someone to be watching. Alerts detect issues automatically.

A good rule: **dashboards for trends and reviews, alerts for immediate action needed**.

---

## 5. On-Call & Alerting

---

**Q21. How do you make on-call less painful?**

**A:**
My approach to improving on-call:

1. **Fix recurring alerts** — if the same alert fires every week, fix the root cause or delete the alert
2. **Write runbooks** — document how to handle every alert so anyone can respond
3. **Reduce false positives** — tune alert thresholds to reduce noise
4. **Automate common fixes** — auto-restart, auto-scale, auto-rollback
5. **Set up proper escalation paths** — so one person is not handling everything alone
6. **Do on-call reviews weekly** — look at every alert that fired, categorize and fix
7. **Fair rotation** — spread on-call across the team, no single person always on-call
8. **Compensate on-call properly** — time off, pay, or recognition

The goal: **on-call should be boring**. If it is exciting, you have reliability problems.

---

**Q22. What is alert fatigue and how do you fix it?**

**A:**
Alert fatigue happens when **too many alerts fire** and engineers start ignoring them — even the real ones.

Signs of alert fatigue:
- Engineers acknowledge alerts without looking at them
- Same alerts fire repeatedly for weeks
- People dread being on-call
- Real incidents are missed because alerts are noise

How to fix it:
1. **Audit every alert** — does it need human action right now?
2. **Delete or silence** alerts that auto-resolve
3. **Raise thresholds** — not every 5xx spike is an emergency
4. **Group related alerts** — one ticket, not 50 notifications
5. **Use multi-window alerting** — alert only if issue lasts more than X minutes
6. **Track alert volume** — set a target (e.g., max 5 pages per on-call shift)

---

## 6. Reliability & Fault Tolerance

---

**Q23. What design patterns do you use to make services more reliable?**

**A:**
Key reliability patterns I use:

| Pattern | What it does |
|---|---|
| **Circuit Breaker** | Stop calling a failing service to prevent cascade failures |
| **Retry with backoff** | Retry failed requests but wait longer each time |
| **Bulkhead** | Isolate failures — one bad service doesn't kill everything |
| **Timeout** | Never wait forever — set limits on all external calls |
| **Rate Limiting** | Protect services from being overwhelmed |
| **Health Checks** | Regularly check if service is alive and ready |
| **Graceful degradation** | Serve a reduced experience instead of total failure |
| **Redundancy** | Run multiple instances so one failure doesn't matter |

---

**Q24. What is a circuit breaker pattern? Give a real example.**

**A:**
A circuit breaker **stops calling a service that is failing**, to prevent making things worse.

States:
- **Closed** — normal, all requests go through
- **Open** — service is failing, stop sending requests, return error immediately
- **Half-open** — after a timeout, try one request to see if service recovered

Real example:
Your web app calls a recommendation service. If recommendations service is down, instead of waiting 30 seconds for each request to timeout (making users wait), the circuit breaker:
1. Detects failures (5 failures in 10 seconds)
2. Opens the circuit — stops calling recommendations service
3. Returns cached recommendations or empty list immediately
4. After 60 seconds, tries one request — if it works, closes the circuit

Tools: Hystrix (Java), resilience4j, AWS App Mesh, Istio

---

**Q25. What is chaos engineering and have you practiced it?**

**A:**
Chaos engineering is the practice of **intentionally injecting failures** into a system to find weaknesses before they cause real outages.

Famous example: Netflix's **Chaos Monkey** — randomly kills production servers to make sure the system survives node failures.

Types of chaos experiments:
- Kill a random pod/node
- Add 500ms latency to service calls
- Consume all CPU or memory on a node
- Drop network packets between services
- Fail a database replica

How I practice it:
1. Start in staging/pre-production
2. Define a steady state (what "normal" looks like)
3. Hypothesize what will happen when failure is injected
4. Run the experiment
5. Compare to hypothesis — if wrong, you found a weakness
6. Fix the weakness, document the experiment

Tools: Chaos Monkey, Gremlin, LitmusChaos (for Kubernetes), AWS Fault Injection Simulator

---

**Q26. What is the difference between High Availability (HA) and Disaster Recovery (DR)?**

**A:**

| | High Availability (HA) | Disaster Recovery (DR) |
|---|---|---|
| **Goal** | Prevent downtime | Recover from a major disaster |
| **Scope** | Single region / AZ failures | Full region failure, data loss |
| **How** | Redundancy, load balancing, auto-scaling | Backup, replication, failover to another region |
| **RTO** | Seconds to minutes | Minutes to hours |
| **RPO** | Near zero | Minutes to hours of data loss |
| **Example** | Run 3 replicas across 3 AZs | Restore from S3 backup in another region |

**RTO** = Recovery Time Objective — how fast must we recover?
**RPO** = Recovery Point Objective — how much data can we afford to lose?

---

## 7. CI/CD & Change Management

---

**Q27. How does SRE approach change management and deployments?**

**A:**
Most outages are caused by changes. SRE's goal is to make changes **safe and reversible**.

Key practices:

1. **Canary deployments** — release to 1-5% of users first, check metrics, then expand
2. **Blue-Green deployments** — run new version alongside old, switch traffic instantly
3. **Feature flags** — deploy code but keep the feature off until ready
4. **Automatic rollback** — if error rate spikes after deploy, auto-rollback
5. **Deployment windows** — avoid deploying on Fridays or during peak traffic
6. **Small, frequent deployments** — easier to rollback than a large change
7. **Change freeze during high-risk periods** — holiday season, major events

---

**Q28. What is a canary deployment and how do you monitor it?**

**A:**
A canary deployment sends a **small percentage of traffic to the new version** while most traffic goes to the old version.

```
100% traffic → Old version (v1)
       ↓
5% traffic → New version (v2)   ← watch this closely
95% traffic → Old version (v1)
       ↓
If v2 is healthy → expand to 50% → 100%
If v2 has issues → rollback to 0%
```

What to monitor during canary:
- Error rate (must not be higher than v1)
- Latency (must not be slower than v1)
- Business metrics (conversion rate, login success)

Tools: Argo Rollouts, Flagger, AWS CodeDeploy, Spinnaker

---

**Q29. How do you handle a bad deployment that is causing errors in production?**

**A:**
My priority is: **restore service first, investigate later**.

Steps:
1. **Confirm** — is the error rate actually higher than before the deploy?
2. **Rollback** — immediately rollback to the previous version
   ```bash
   kubectl rollout undo deployment/my-app
   # or
   git revert + redeploy
   ```
3. **Confirm recovery** — check error rate is back to normal
4. **Notify stakeholders** — what happened, what was done
5. **Investigate in staging** — reproduce the issue without production impact
6. **Fix forward only if rollback is not possible** — (database migrations, etc.)
7. **Write postmortem**

---

## 8. Capacity Planning & Performance

---

**Q30. How do you do capacity planning for a service?**

**A:**
Capacity planning = making sure you have enough resources before you need them.

My process:

1. **Measure current usage** — CPU, memory, storage, network, requests per second
2. **Find the bottleneck** — which resource will run out first?
3. **Predict growth** — use historical trends (e.g., 20% growth per quarter)
4. **Calculate when you hit the limit** — "at current growth we hit 80% CPU in 3 months"
5. **Plan ahead** — add capacity before you need it (80% utilization is the warning level)
6. **Load test** — verify the system handles 2x current peak traffic
7. **Set auto-scaling policies** — so the system handles spikes automatically

Review capacity quarterly or whenever traffic patterns change significantly.

---

**Q31. What is the difference between vertical scaling and horizontal scaling?**

**A:**

| | Vertical Scaling (Scale Up) | Horizontal Scaling (Scale Out) |
|---|---|---|
| **What** | Make one machine bigger | Add more machines |
| **Example** | Upgrade from 4 CPU to 16 CPU | Add 3 more servers |
| **Limit** | Has a hardware limit | Almost unlimited |
| **Downtime** | Usually requires restart | No downtime |
| **Cost** | Expensive at large scale | More cost-efficient |
| **Best for** | Databases, stateful systems | Web servers, microservices |

SRE preference: **horizontal scaling** whenever possible — it gives better availability and no single point of failure.

---

**Q32. What is a load test and how do you run one?**

**A:**
A load test sends **simulated user traffic** to your service to see how it behaves under stress.

Types:
- **Load test** — normal expected traffic × 2
- **Stress test** — push until the system breaks, find the limit
- **Soak test** — run normal load for a long time (hours/days) to find memory leaks
- **Spike test** — sudden huge increase in traffic

Tools: k6, JMeter, Locust, Gatling, AWS Artillery

Simple k6 example:
```javascript
import http from 'k6/http';
export let options = {
  vus: 100,        // 100 virtual users
  duration: '5m',  // run for 5 minutes
};
export default function () {
  http.get('https://my-service.com/api/health');
}
```

What to watch during load test: response time, error rate, CPU, memory, DB connections.

---

## 9. Linux & Systems

---

**Q33. A Linux server is running slow. How do you troubleshoot it?**

**A:**
I use this systematic approach:

```bash
# 1. Check overall system health
top                    # CPU and memory overview
htop                   # better visual version

# 2. Check CPU
uptime                 # load average (should be < number of CPUs)
mpstat -P ALL 1        # per-CPU usage

# 3. Check Memory
free -h                # RAM and swap usage
vmstat 1               # virtual memory stats

# 4. Check Disk I/O
iostat -xz 1           # disk read/write stats
df -h                  # disk space
iotop                  # which process is using most disk I/O

# 5. Check Network
netstat -tunp          # open connections
ss -s                  # socket stats
iftop                  # network traffic by connection

# 6. Check processes
ps aux --sort=-%cpu    # top CPU consumers
ps aux --sort=-%mem    # top memory consumers

# 7. Check logs
tail -f /var/log/syslog
journalctl -xe
```

I start wide and narrow down — usually the issue is CPU, memory, disk I/O, or network.

---

**Q34. What is the difference between a process and a thread?**

**A:**
- **Process** — an independent running program with its own memory space
  - Example: Chrome browser is a process
  - If it crashes, it does not affect other processes

- **Thread** — a lightweight unit of execution inside a process, shares memory
  - Example: each Chrome tab runs in a thread (or sub-process)
  - Threads in the same process share memory — faster but risky (race conditions)

For SRE: understanding this helps when a process crashes vs when it hangs (likely a thread deadlock), and helps with debugging using tools like `strace` and `gdb`.

---

**Q35. What is OOM Killer in Linux and how do you handle it?**

**A:**
OOM Killer = Out Of Memory Killer. When Linux runs out of RAM, it picks a process and **kills it** to free memory.

How to detect it:
```bash
dmesg | grep -i "oom"
grep -i "oom" /var/log/syslog
```

You will see something like:
```
Out of memory: Kill process 1234 (java) score 800 or sacrifice child
```

How to handle:
1. **Find why memory ran out** — memory leak? Unexpected traffic spike?
2. **Increase memory** — vertical scale or tune JVM/app heap settings
3. **Set memory limits in Kubernetes** — so one pod doesn't consume all node memory
4. **Fix the memory leak** — use heap dumps, profilers (JProfiler, VisualVM)
5. **Set up memory alerts** — alert at 80% before OOM happens

---

## 10. Kubernetes & Cloud

---

**Q36. How do you ensure high availability for an application on Kubernetes?**

**A:**
I use these Kubernetes features for HA:

```yaml
# 1. Multiple replicas
spec:
  replicas: 3

# 2. Pod Anti-Affinity — spread pods across nodes
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: my-app
      topologyKey: kubernetes.io/hostname

# 3. PodDisruptionBudget — always keep 2 pods running
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app

# 4. Resource limits — prevent one pod from starving others
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

# 5. Liveness and Readiness probes
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
```

---

**Q37. What is the difference between liveness probe and readiness probe in Kubernetes?**

**A:**

| | Liveness Probe | Readiness Probe |
|---|---|---|
| **Question it asks** | Is the container alive? | Is the container ready to receive traffic? |
| **If fails** | Kubernetes **restarts** the container | Kubernetes **removes** it from the load balancer |
| **Use case** | Detect deadlocks, crashes | Detect when app is still starting up or temporarily overloaded |
| **Example** | App is running but stuck in infinite loop | App is starting, needs 30 seconds to warm up cache |

Simple rule:
- Liveness = **restart me if I'm broken**
- Readiness = **don't send me traffic if I'm not ready**

---

**Q38. How do you debug a pod that is stuck in CrashLoopBackOff?**

**A:**
```bash
# 1. Check pod status and events
kubectl describe pod <pod-name> -n <namespace>

# 2. Check current logs
kubectl logs <pod-name> -n <namespace>

# 3. Check previous container logs (before it crashed)
kubectl logs <pod-name> -n <namespace> --previous

# 4. Check resource limits — is it being OOMKilled?
kubectl describe pod <pod-name> | grep -A 5 "Last State"

# 5. Try running the container manually to reproduce
kubectl run debug --image=<same-image> --rm -it -- /bin/sh

# 6. Check if ConfigMap or Secret it depends on exists
kubectl get configmap <name> -n <namespace>
kubectl get secret <name> -n <namespace>
```

Common causes:
- Application error on startup (check logs)
- Missing environment variable or config
- Wrong image or entrypoint command
- OOMKilled — memory limit too low
- Failing liveness probe that restarts the pod

---

## 11. Toil & Automation

---

**Q39. Give an example of toil you have automated in your career.**

**A:**
A strong answer structure:

**Situation:** We had a manual process where on-call engineers had to restart a specific microservice every night at 2am because of a memory leak. It had been happening for 3 months.

**What I did:**
1. First, I wrote a script to detect the leak and auto-restart:
   ```bash
   #!/bin/bash
   MEMORY=$(kubectl top pod my-service --no-headers | awk '{print $3}' | sed 's/Mi//')
   if [ "$MEMORY" -gt 1500 ]; then
     kubectl rollout restart deployment/my-service
     echo "Restarted due to high memory: ${MEMORY}Mi"
   fi
   ```
2. Set it up as a Kubernetes CronJob running every hour
3. Added alerting so we knew when it triggered
4. Then worked with the dev team to find and fix the actual memory leak
5. Removed the CronJob once the leak was fixed

**Result:** Removed the 2am manual restart. Improved on-call sleep. Also found and fixed the root cause.

---

**Q40. How do you decide what to automate first?**

**A:**
I use two criteria to prioritize automation:

1. **Frequency × Time** — how often does it happen × how long it takes each time
2. **Risk** — can a human error here cause an outage?

I make a simple spreadsheet:

| Task | Frequency | Time per task | Risk | Priority |
|---|---|---|---|---|
| Restart service | Daily | 10 min | Medium | HIGH |
| Add new user | Weekly | 5 min | Low | MEDIUM |
| Provision new server | Monthly | 2 hours | High | HIGH |
| Generate monthly report | Monthly | 30 min | Low | LOW |

Automate high-priority items first. Tasks that are rare and low-risk can stay manual.

---

## 12. Scenario-Based Questions

---

**Q41. Your service's error rate jumped from 0.1% to 15% suddenly. What do you do?**

**A:**
1. **Acknowledge** — take ownership of the incident immediately
2. **Check recency** — what changed in the last 30-60 minutes? (deploy, config change, infra change)
3. **Check scope** — is it all users or specific users/regions?
4. **Look at metrics** — where exactly is the error happening? (app, DB, external API?)
5. **Check logs** — what error message is appearing?
6. **Rollback if recent deploy** — don't wait, rollback first
7. **Notify stakeholders** — even if you don't know the cause yet
8. **Escalate** — if not resolved in 15 minutes, bring in more people
9. **Restore** — find the fastest path to bring error rate down
10. **Postmortem** — document within 48 hours

---

**Q42. Your on-call alert fires at 3am. You look at the dashboard and everything looks normal. What do you do?**

**A:**
1. **Confirm the alert is real** — check the data the alert is based on, not just dashboards
2. **Look at alert history** — is this a flapping alert that fires and auto-resolves?
3. **Check if there was a brief spike** — maybe a 1-minute blip triggered the alert
4. **Look at logs** — was there actually an error even if it is resolved now?
5. **Check if alert threshold is too sensitive** — should it require 5 minutes of errors, not 1 minute?

If truly a false alarm:
- Acknowledge and close the alert
- Create a ticket to tune the alert threshold
- If this alert fires more than once a week with no action needed, fix or delete it

This is alert fatigue — I would make it a priority to fix this alert in the next working day.

---

**Q43. You need to do a database migration during a live production deployment. How do you approach it?**

**A:**
Database migrations are risky because they are hard to rollback. I use this approach:

**Rule: Never do a migration that is not backwards compatible in one step.**

**Safe migration pattern (expand-contract):**

Step 1 — **Expand**: Add the new column/table without removing the old one. Deploy app code that writes to BOTH old and new.

Step 2 — **Migrate**: Run a background job to copy data from old to new.

Step 3 — **Contract**: Once all data is migrated and verified, remove the old column. Deploy app code that only uses the new column.

This way:
- You can rollback the app code at any step
- No downtime required
- No big-bang migration that locks the database

---

**Q44. Your team is getting paged 20+ times a day. How do you fix this?**

**A:**
20+ pages/day is a crisis. Here is my plan:

**Week 1: Stop the bleeding**
- Identify the top 3 most frequent alerts
- Silence or raise thresholds on alerts that auto-resolve without action
- Temporarily reduce noise to manageable level

**Week 2-4: Fix root causes**
- For each frequent alert, find and fix the root cause
- If the root cause is a known bug, work with devs to prioritize it
- Write or improve runbooks for remaining alerts

**Ongoing process:**
- Weekly on-call review — look at every page, mark as "actionable" or "needs fixing"
- Set a team goal: max 5 actionable pages per on-call shift
- Track alert volume in a dashboard over time

The goal is to make on-call **boring** — if the pager fires, it should always mean something real.

---

**Q45. How would you design monitoring for a new microservice from scratch?**

**A:**
I follow this order:

**1. Define SLIs first** — what matters to users?
- Success rate of API calls
- P99 latency
- Throughput (requests/sec)

**2. Set SLOs** — what targets are acceptable?
- 99.9% success rate, P99 < 300ms

**3. Build dashboards**
- Golden signals: Rate, Errors, Duration, Saturation (USE + RED)
- One page for service health, one for infrastructure health

**4. Set alerts**
- Alert on SLO burn rate — "you will exhaust error budget in 1 hour if this continues"
- Not on raw metrics (not "CPU > 80%", but "error rate > SLO threshold")

**5. Add structured logging**
- Every request logged with: timestamp, request ID, user ID, status, duration
- Easy to query in a log tool

**6. Add distributed tracing**
- Instrument with OpenTelemetry
- Connect traces across microservice boundaries

**7. Runbooks**
- Write a runbook for every alert before it goes to production

---

## Quick Reference Summary

```
SLI  = what you measure
SLO  = target you set
SLA  = promise to customers (with penalties)

Error Budget = 100% - SLO
Toil = manual, repetitive, no lasting value

MTTD = time to detect
MTTR = time to recover

3 Pillars = Metrics + Logs + Traces
RED = Rate, Errors, Duration (for services)
USE = Utilization, Saturation, Errors (for resources)

Incident order = Detect → Triage → Mitigate → Resolve → Postmortem
Deployment safety = Canary → Monitor → Expand or Rollback

HA = survive node/AZ failure (same region)
DR = survive region failure (cross-region backup)
```

---

*45 Questions | SRE | 8+ Years Experience | Easy English*
*Generated: 2026-06-29*
