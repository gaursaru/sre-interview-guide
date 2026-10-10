# SRE Interview Guide

Comprehensive interview guide for SRE professionals covering architecture, troubleshooting, Kubernetes, and FinOps.

---

## Table of Contents

1. [Part 1: Architecture & Infrastructure](#part-1-architecture--infrastructure)
2. [Part 2: Troubleshooting & Scenarios](#part-2-troubleshooting--scenarios)
3. [Part 3: Advanced Topics](#part-3-advanced-topics)
4. [Part 4: Python Coding Exercise](#part-4-python-coding-exercise)
5. [Part 5: Command Reference](#part-5-command-reference)

---

## Part 1: Architecture & Infrastructure

### 1. High Availability (HA) & Disaster Recovery (DR)

**Q: How do you architect a highly available, fault-tolerant system on AWS?**

**Headline:**  
_"Spread compute and data across at least two Availability Zones, keep compute strictly stateless, and build a cross-region DR path tuned to business RTO/RPO."_

#### Memory Steps

**Networking:**
- Multi-AZ VPC with public and private subnets
- Public subnets: Application Load Balancer (ALB) and NAT Gateways only
- Private subnets: Compute and databases

**Compute:**
- Stateless containers on EKS/OpenShift across private subnets
- Auto Scaling Groups (ASGs) for dynamic scaling

**Data:**
- Multi-AZ RDS (or DynamoDB Global Tables) in private subnets
- Cross-region read replicas for DR
- S3 with Cross-Region Replication (CRR) and KMS encryption

**Traffic & DR:**
- Route 53 failover routing with ALB health checks
- Health-check-driven failover between regions

#### Why This Answer?
Interviewers want to see:
- Stateless compute (no local state)
- Network segmentation (public vs. private)
- DR driven by business constraints (RTO/RPO), not over-engineering

#### Follow-Up Probes

**Why is the ALB in a public subnet?**
- Acts as the ingress bridge for external traffic
- Public subnets route `0.0.0.0/0` to an Internet Gateway (IGW)
- ALB terminates TLS and forwards requests to backends over private IPs
- Compute nodes never need public IPs

**Security Groups vs. NACLs:**

| Aspect | Security Group | NACL |
|--------|---|---|
| **Statefulness** | Stateful (return traffic auto-allowed) | Stateless (explicit allow/deny) |
| **Scope** | ENI/instance level | Subnet boundary |
| **Rules** | Allow-rules only | Allow and deny rules |
| **Use Case** | Least-privilege chaining (e.g., App-SG accepts from ALB-SG) | Broad CIDR/IP blocklists |

---

### 2. Containers: EKS vs. ECS vs. Serverless

**Q: Why choose EKS over ECS? Which is cheaper?**

**Headline:**  
_"Choose EKS for Kubernetes ecosystem tools and multi-cloud portability; choose ECS for lower overhead in pure AWS environments. ECS is cheaper for small/medium setups (no control-plane fee)."_

#### Memory Comparison

| Factor | ECS | EKS |
|--------|---|---|
| **Cluster Management** | Zero cluster fee | ~$0.10/hour (~$73/month) |
| **Best For** | Small/medium teams, straightforward microservices | Kubernetes ecosystem, GitOps, multi-cloud |
| **Tooling** | Basic task definitions | Karpenter, KEDA, ArgoCD, service meshes |
| **Portability** | AWS-specific | Standard Kubernetes manifests (cloud-agnostic) |

#### Why This Answer?
Demonstrates commercial maturity:
- Acknowledge EKS has cluster fees
- At scale, advanced autoscalers (Karpenter/KEDA + Spot) save more than $73/month
- ECS is cheaper upfront, but less flexible long-term

---

### 3. Hybrid Networking & VPC Connectivity

**Q: Why Transit Gateway over VPC Peering? How does cost factor in?**

**Headline:**  
_"VPC Peering is simple and cheap for a few VPCs; Transit Gateway scales to dozens of accounts and on-premise networks via a centralized hub-and-spoke model."_

#### Memory Comparison

| Aspect | VPC Peering | Transit Gateway |
|--------|---|---|
| **Topology** | Non-transitive point-to-point | Hub-and-spoke central router |
| **Scaling** | N VPCs = N(N-1)/2 links (15 accounts = 105 links!) | 1 attachment per VPC + central hub |
| **Hourly Cost** | None (data transfer only) | Per-attachment + per-GB processing fee |
| **Best For** | Simple 2–3 VPC setups | Enterprise with 10+ accounts/regions |

#### Why This Answer?
Shows understanding of:
- Architectural limits (routing table sprawl with peering)
- Financial trade-offs (processing overhead vs. operational complexity)

---

### 4. Infrastructure as Code: Terraform vs. CloudFormation

**Q: How do you structure Terraform for a multi-account enterprise?**

**Headline:**  
_"Use reusable, version-controlled modules combined with lightweight per-account root configs, where every environment maintains isolated remote state."_

#### Directory Pattern

```
infrastructure/
├── modules/
│   ├── vpc/
│   ├── eks/
│   ├── rds/
│   └── security_groups/
└── environments/
    ├── dev/
    │   ├── account_101/
    │   │   ├── main.tf
    │   │   ├── backend.tf
    │   │   └── terraform.tfvars
    └── prod/
        ├── account_202/
        │   ├── main.tf
        │   ├── backend.tf
        │   └── terraform.tfvars
```

#### Safety & Concurrency

**Remote State:**
- S3 backend, encrypted with customer-managed KMS keys
- Concurrency locking via DynamoDB (or S3 native locking in Terraform 1.10+)

**CI/CD Pipeline:**
- `terraform plan` runs on pull requests
- `terraform apply` runs only on merge to `main`
- GitHub Actions or similar orchestration

#### Why This Answer?
- Separate state files per account isolate blast radius
- If dev state corrupts, prod remains safe
- Prevents "terraform apply" mistakes across environments

---

### 5. Security & IAM Authorization Mechanics

**Q: How does IAM evaluate a request, and how do you secure cross-account access?**

**Headline:**  
_"IAM defaults to Deny. Any explicit Deny wins immediately over all Allows. Cross-account access requires dual-handshake authorization from both source and target."_

#### IAM Evaluation Order

```
Default Deny
    ↓
Explicit Deny? → YES → DENY (stops here)
    ↓ NO
Explicit Allow? → YES → ALLOW
    ↓ NO
DENY (final)
```

#### Cross-Account S3/KMS Access Pattern

**Source Account (Account A):**
- Workload role has `sts:AssumeRole` permission targeting Account B's Role ARN

**Target Account (Account B):**
- Role B's Trust Policy allows Account A's role to assume it

**Resource Policy (Account B):**
- S3 Bucket Policy explicitly permits Role B / Account A
- **KMS Key Policy must also explicitly permit Role B** ← Most common failure point

#### Why This Answer?
Demonstrates:
- Deep mechanics, not just UI knowledge
- Ability to troubleshoot complex security failures (missing KMS Key Policy is #1 silent failure)

---

## Part 2: Troubleshooting & Scenarios

### Universal Senior Answer Structure

When asked "How would you troubleshoot X?", use this proven 3-step framework:

1. **"First, I would verify..."** – Confirm impact using CloudWatch, metrics, or status codes
2. **"Then, I would isolate layer-by-layer..."** – DNS → Network → Security Groups → OS/Process → IAM
3. **"Finally, I would remediate..."** – Mitigate live impact first, then root-cause analysis

---

### 1. Linux & Systems Troubleshooting

**Q: The application reports "Disk Full", but `df -h` shows 50% free space. How do you troubleshoot?**

**Headline:**  
_"Check for inode exhaustion, mount point mismatches, user quotas, or deleted files held open by running processes."_

#### Diagnostic Commands

| Issue | Command | What It Shows |
|-------|---------|---|
| **Mount Mismatch** | `findmnt /path/to/app` | Confirm app isn't writing to a different, smaller mount or `/tmp` |
| **Inode Exhaustion** | `df -i` | If IUse% = 100%, no new files can be created (even with free blocks) |
| **Deleted Files Held Open** | `sudo lsof +L1` | Unlinked files still held by running process handles |

#### Remediate Unlinked Files

```bash
# Safely truncate file descriptor (freespace without dropping service)
: > /proc/<PID>/fd/<FD>

# Or reload the process gracefully
systemctl reload <app>
```

#### Why This Answer?
- Deleting files with `rm` while a process writes doesn't free blocks until process releases handle
- Shows live production experience in managing space safely

---

**Q: EC2 instance status checks pass (2/2), but SSH connection fails. How do you isolate it?**

**Headline:**  
_"Categorize failure by exact SSH error: Timeout = network routing, Refused = OS process, Denied = authentication."_

#### Isolation Matrix

| Error Type | Likely Cause | Diagnostics |
|---|---|---|
| **Connection Timeout** | Network layer | Check Security Group inbound rules (TCP 22), NACLs, route table routes to IGW/NAT, client egress |
| **Connection Refused** | Transport/OS layer | Port 22 reachable, but sshd daemon stopped or OS firewall (iptables/ufw) dropping traffic. Use AWS SSM Session Manager to access. |
| **Permission Denied (publickey)** | Authentication | Wrong SSH key, invalid permissions on `~/.ssh/authorized_keys` (must be 700 dir, 600 file), wrong username |

---

### 2. AWS Production Failure Scenarios

**Q: ALB returns HTTP 502 Bad Gateway, but EC2 targets show "Healthy". Why?**

**Headline:**  
_"A 502 means the ALB reached the backend, but received invalid/unparseable HTTP response or connection reset."_

#### Root Causes

**Keep-Alive Timeout Mismatch:**
- Target app's HTTP keep-alive timeout < ALB's idle timeout (60s default)
- Backend closes TCP connection as ALB forwards request
- **Fix:** Increase app keep-alive timeout higher than ALB default

**Shallow Health Checks:**
- `/health` endpoint returns 200 OK from basic thread
- Main application worker thread crashes or times out on actual requests
- **Fix:** Validate health check endpoint mirrors real application behavior

**Protocol/Port Misalignment:**
- ALB sends HTTP/2 or HTTPS, but app expects HTTP/1.1 or plain HTTP
- **Fix:** Verify target group protocol matches application listener

---

**Q: Private subnet EC2 instance connects to RDS fine, but cannot reach external internet APIs.**

**Headline:**  
_"Trace the outbound internet chain: Private Subnet Route Table → NAT Gateway → Internet Gateway."_

#### Outbound Path Verification

```
EC2 (Private Subnet)
    ↓
Route 0.0.0.0/0 → NAT Gateway
    ↓
NAT Gateway (Public Subnet)
    ↓
Route 0.0.0.0/0 → Internet Gateway (IGW)
    ↓
External Internet
```

**Checklist:**
- [ ] Private subnet route table has `0.0.0.0/0` → NAT Gateway
- [ ] NAT Gateway is in a public subnet
- [ ] Public subnet route table has `0.0.0.0/0` → Internet Gateway
- [ ] NAT Gateway is healthy (EIP attached, not deleted/starved)
- [ ] Outbound Security Group/NACL rules allow port 443 (HTTPS)

#### Why RDS Works But APIs Don't
- RDS traffic stays inside VPC (no NAT needed)
- External APIs require NAT Gateway path

---

### 3. Kubernetes Operational Scenarios

**Q: A Pod is stuck in CrashLoopBackOff, ImagePullBackOff, or Pending. What does each mean?**

**Headline:**  
_"Diagnose pod states by running `kubectl describe pod <pod>` and inspecting the event timeline at the bottom."_

#### Pod State Diagnosis

| State | Meaning | Diagnosis Command | Root Cause | Fix |
|-------|---------|---|---|---|
| **CrashLoopBackOff** | App starts, crashes, kubelet retries with exponential backoff | `kubectl logs <pod> --previous` | Check exit code and crash logs | Fix app bug or missing dependencies |
| **ImagePullBackOff** | Container runtime cannot pull image | `kubectl describe pod <pod>` → Events | Image name/tag typo, ECR lifecycle expiration, missing registry secret, missing IRSA permissions | Verify image, ECR permissions, registry credentials |
| **Pending** | Scheduler cannot place pod on any node | `kubectl describe pod <pod>` → Events | Insufficient CPU/Memory, unresolved NodeSelectors/Affinities, unsatisfied PersistentVolumeClaims | Add node capacity, fix affinity rules, check PVC status |

---

**Q: How do you upgrade an EKS cluster with zero downtime?**

**Headline:**  
_"Upgrade one minor version at a time: control plane first, node groups via rolling replacement with drain/cordon, add-ons last."_

#### Upgrade Sequence

**1. Audit & Preparation**
```bash
# Check for deprecated API versions
pluto detect-files -d .

# Alternative
kubent

# Take cluster backup
velero backup create pre-upgrade-backup
```

**2. Control Plane Upgrade**
```bash
# Via Terraform or AWS CLI
aws eks update-cluster-version --name <cluster> --kubernetes-version 1.29

# Note: Control plane cannot be downgraded
```

**3. Worker Node Rolling Replacement**

For each node in the node group:
```bash
# Mark node as unschedulable
kubectl cordon <node>

# Evict all workloads (respects PodDisruptionBudgets)
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

# Replace/update node (Managed Node Group or Karpenter)
# (Wait for node to come back up)

# Mark node as schedulable again
kubectl uncordon <node>
```

**4. Add-on Upgrade**
- Update VPC CNI
- Update CoreDNS
- Update kube-proxy

All to matching control plane version.

#### Why This Answer?
- Demonstrates understanding of Kubernetes upgrade mechanics
- Shows consideration for workload disruption (drain/cordon)
- Avoids rushed upgrades that cause outages

---

## Part 3: Advanced Topics

### 1. Reliability & Observability Mechanics

**Q: How do you define SLI, SLO, and SLA, and how do you manage an Error Budget?**

**Headline:**  
_"SLI is what you measure, SLO is the goal, SLA is the customer contract, and Error Budget = 100% - SLO."_

#### Definitions & Real-World Example

| Term | Definition | Example |
|------|-----------|---------|
| **SLI (Service Level Indicator)** | Metric you measure | Ratio of successful HTTP 200-499 requests over total requests (measured at ALB over 30 days) |
| **SLO (Service Level Objective)** | Target goal with product teams | 99.9% success rate over rolling 30-day window |
| **SLA (Service Level Agreement)** | Commercial contract with customers | 99.9% uptime; customer gets refund if breached |
| **Error Budget** | Allowed unreliability (100% - SLO) | 0.1% = ~43 minutes downtime/month |

#### Managing Burn Rate

**Alert Thresholds:**
- Error budget burn rate > 2x expected → Notify on-call SRE via PagerDuty
- Error budget 100% exhausted → Freeze non-critical feature deployments, divert engineering to reliability

#### Why This Answer?
Bridges technical telemetry directly to business and organizational policy—true Staff/Senior leadership thinking.

---

**Q: How do you handle a major live production outage as an SRE Incident Commander?**

**Headline:**  
_"Establish clear roles: stabilize user impact first, communicate transparently, defer deep root-cause analysis until system is healthy."_

#### Execution Sequence

**1. Declare & Assign Roles**
```
Severity: Sev-1
├── Incident Commander (drives response, owns timeline)
├── Operations Lead (executes fixes)
└── Communications Lead (updates stakeholders every 15–30 min)
```

**2. Mitigate Impact (Stabilization)**
Prefer in this order:
1. Rollback (fastest)
2. Traffic rerouting (Route 53 failover)
3. Feature flag disablement
4. (Last resort) Live debugging in production

**3. Post-Incident (Blameless Postmortem)**

Within 48 hours:
- Document full incident timeline
- Identify contributing factors (not "who failed")
- Highlight detection gaps
- Create actionable JIRA tickets for prevention
- **Focus on systemic improvements, never personal blame**

#### Why This Answer?
Shows maturity in handling chaos and preventing similar incidents.

---

### 2. Advanced Kubernetes Architecture

**Q: What is the difference between Liveness, Readiness, and Startup Probes?**

**Headline:**  
_"Liveness restarts unhealthy containers; Readiness removes unready pods from load balancer; Startup protects slow-booting apps."_

#### Probe Comparison

| Probe | Question | Action on Fail | Use Case |
|-------|----------|---|---|
| **Liveness** | "Is the process stuck/deadlocked?" | Kill and restart container | Catch hung processes; prevents zombie pods |
| **Readiness** | "Can this pod handle traffic NOW?" | Remove from Service Endpoints (no restart) | Database pool full, cache loading, slow startup |
| **Startup** | "Is the app still initializing?" | Disable Liveness/Readiness checks | Protect slow-booting apps (e.g., 2GB cache loading) |

#### Yaml Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example
spec:
  containers:
  - name: app
    image: myapp:latest
    
    startupProbe:
      httpGet:
        path: /startup
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
    
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
    
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

---

**Q: How do you secure Kubernetes Secrets?**

**Headline:**  
_"Base Kubernetes secrets are only Base64-encoded; securing them requires etcd encryption at rest, strict RBAC, and external secret injectors."_

#### Security Layers

**At Rest (Encryption):**
```bash
# Enable AWS KMS Envelope Encryption for etcd
# In EKS cluster settings → Control Plane Encryption
aws eks update-cluster-config --name <cluster> --encryption-config resources=secrets,provider=key-arn=<kms-key-arn>
```

**In Git/Pipeline:**
- Never commit secrets to version control
- Use External Secrets Operator (ESO) or Secrets Store CSI Driver
- Sync secrets dynamically from AWS Secrets Manager or Delinea into pod memory

**In Cluster (RBAC):**
```bash
# Restrict get/list on secrets
kubectl create role secret-reader --verb=get --resource=secrets
kubectl create rolebinding secret-reader-binding --role=secret-reader --serviceaccount=default:app

# Mount as temporary volume files (not env vars)
# Prevents secrets from appearing in `ps` or environment dumps
```

#### Why This Answer?
Shows understanding that "secret" is misleading—Base64 encoding is not encryption.

---

### 3. FinOps & Cost Engineering

**Q: How do you cut AWS infrastructure costs by 30% without sacrificing reliability?**

**Headline:**  
_"Target the big three cost drivers: Compute (~40%), Storage (~20%), and Data Transfer/Networking (~15%)."_

#### Actionable Plan

| Cost Driver | Strategy | Savings | Implementation |
|---|---|---|---|
| **Compute** | Move steady-state to Compute Savings Plans | ~40% | Buy 1–3 year commitments |
| | Migrate non-prod/batch to Spot (Karpenter) | ~70% | Karpenter auto-scales Spot with graceful fallback |
| **Storage** | EBS gp2 → gp3 (instant upgrade) | ~20% | Migrate in-place, better baseline IOPS |
| | S3 Lifecycle Rules (→ Intelligent-Tiering, Glacier) | ~30% | Auto-transition old logs/artifacts |
| **Networking** | Replace cross-AZ NAT Gateway with Gateway VPC Endpoints | ~100% for S3/DynamoDB traffic | No egress charges, traffic stays inside VPC |
| | | | |
| **Governance** | Mandatory resource tagging + auto-shutdown (Lambda/EventBridge) | ~15% | Non-prod environments shut down outside business hours |

#### Implementation Priority

1. **Week 1:** Enable S3 Lifecycle Rules + move to gp3 (quick wins)
2. **Week 2:** Deploy Gateway VPC Endpoints for S3/DynamoDB
3. **Week 3:** Consolidate non-prod on Spot (Karpenter)
4. **Month 2:** Purchase Compute Savings Plans for predictable prod workloads

#### Why This Answer?
Shows pragmatic, data-driven approach without sacrificing reliability.

---

## Part 4: Python Coding Exercise

In Senior SRE interviews, live coding focuses on **data manipulation, string parsing, and algorithm efficiency** rather than complex dynamic programming.

### Task: Reverse a List In-Place

**Input:** `[10, 20, 30, 40, 50]`  
**Output:** `[50, 40, 30, 20, 10]`  
**Constraints:** O(n) time, O(1) extra space

#### Solution

```python
def reverse_in_place(nums: list) -> list:
    """
    Reverses a list in-place using two pointers.
    
    Args:
        nums: List of integers to reverse
    
    Returns:
        The same list, reversed in-place
    
    Time Complexity: O(n) - iterates through half the list
    Space Complexity: O(1) - modifies memory in-place, no extra structures
    """
    left = 0
    right = len(nums) - 1
    
    while left < right:
        # Swap values using tuple unpacking
        nums[left], nums[right] = nums[right], nums[left]
        left += 1
        right -= 1
    
    return nums


# Test Case
numbers = [10, 20, 30, 40, 50]
reverse_in_place(numbers)
print(numbers)  # Output: [50, 40, 30, 20, 10]
```

#### Explanation

- **Two pointers:** Start at both ends of list
- **Swap in-place:** No temporary array created
- **Halt condition:** `left < right` (pointers meet in middle)
- **Time:** O(n) iterations through half the list
- **Space:** O(1) no extra data structures

---

### Follow-Up Questions to Expect

**Q: How does Python's native `.reverse()` work?**
- `nums.reverse()` → O(n) in-place (written in C)
- `nums[::-1]` → Creates entirely new shallow-copy list in memory (violates O(1) space)

**Q: What if the input is a string CSV: "10,20,30,40,50"?**

```python
def reverse_csv_string(csv_str: str) -> str:
    """Parse CSV string, reverse, and reformat."""
    # Parse string into integer list
    nums = [int(x.strip()) for x in csv_str.split(",")]
    
    # Reverse in-place
    reverse_in_place(nums)
    
    # Reformat back to CSV string
    return ",".join(map(str, nums))


# Test
result = reverse_csv_string("10,20,30,40,50")
print(result)  # Output: "50,40,30,20,10"
```

---

## Part 5: Command Reference

Quick command matrix for live speech during troubleshooting explanations. Drop these **exact commands** into your verbal explanation.

| Scenario | Primary Command | What It Shows |
|----------|---|---|
| **High CPU Usage** | `top` or `ps aux --sort=-%cpu \| head` | Exact PID consuming CPU cycles |
| **High RAM / OOM** | `free -m` / `dmesg \| grep -i oom` | Physical RAM vs Swap; confirms OOM-killer execution |
| **Port Binding** | `sudo ss -tulpn \| grep :8080` | Verifies if app bound to `0.0.0.0` or `127.0.0.1` |
| **Network Path** | `nc -vz <IP> <Port>` | Tests Layer-4 TCP handshake across SG/firewalls |
| **DNS Resolution** | `dig +short <hostname>` | Isolates DNS issues from network drops |
| **Process Logs** | `journalctl -u <service> --since "30 min ago"` | Reads systemd unit service logs |
| **Kubernetes Events** | `kubectl get events --sort-by=.lastTimestamp` | Recent cluster-level failures (OOM, eviction, nodes) |
| **ASG Failures** | `aws autoscaling describe-scaling-activities` | Exact API rejection reasons for launch failures |

---

## Quick Tips

✅ **Structure all troubleshooting answers:** Verify → Isolate → Remediate  
✅ **Know your layered model:** DNS → Network → Security → OS → Process → Application  
✅ **Ask "Why?" for every answer:** Shows deeper understanding  
✅ **Mention blast radius and isolation:** Separates senior from junior thinking  
✅ **Default-deny mindset for security:** IAM, firewalls, least-privilege everywhere  
✅ **Cost + reliability trade-offs:** Always mention business constraints  
✅ **Live production experience:** Use specific examples and gotchas you've encountered  

---

## Contributing

To contribute additional Q&A or improvements, please submit a pull request with clearly formatted sections following this guide's structure.
