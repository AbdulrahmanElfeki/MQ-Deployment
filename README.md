# IBM MQ StatefulSet Configuration Documentation

**Environment:** SIT (System Integration Testing)  
**Queue Manager Name:** `QM01`  
**IBM MQ Version:** 9.4.5.0-r2  
**Image:** `quay.io/aelfeki007/ibm-messaging/mq:9.4.5.0-r2`

---

## Overview

This document describes the IBM MQ authentication StatefulSet deployed in Kubernetes. The setup includes a single-replica queue manager with persistent storage, monitoring probes, and both MQ and web console access.

---

## Architecture & Components

### 1. **StatefulSet Configuration**

| Property | Value | Description |
|----------|-------|-------------|
| **Name** | `ibm-mq-auth` | Unique identifier for the StatefulSet |
| **Replicas** | 1 | Single instance deployment |
| **Pod Management** | OrderedReady | Pods created/terminated sequentially |
| **Revision History** | 10 | Last 10 revisions retained |

### 2. **Kubernetes Service**

```yaml
Service Name: ibm-mq-auth
Type: ClusterIP (default)

Exposed Ports:
  - Port 1414 (MQ Listener)
  - Port 9443 (Web Console - HTTPS)
```

**Service DNS Names:**
- Within cluster: `ibm-mq-auth.default.svc.cluster.local`
- Same namespace: `ibm-mq-auth`
- Full FQDN: `ibm-mq-auth.default.svc.cluster.local:1414`

### 3. **Persistent Storage**

| Property | Value | Purpose |
|----------|-------|---------|
| **Storage Size** | 20Gi | Queue manager data and message storage |
| **Access Mode** | ReadWriteOnce | Single pod read-write access |
| **Volume Mode** | Filesystem | Standard filesystem mode |
| **Mount Path** | `/mnt/mqm` | Queue manager data directory |
| **Retention Policy** | Retain | PVC retained when pod deleted/scaled |

---

## Environment Configuration

### License & Queue Manager Setup

```yaml
LICENSE: accept                    # Accepts IBM MQ license terms
MQ_QMGR_NAME: QM01                 # Queue Manager identifier
MQ_ENABLE_WEB: "true"              # Enable web console on port 9443
```

### Authentication Credentials

```yaml
MQ_ADMIN_PASSWORD:
  Secret: mq-web-credentials
  Key: password
  Purpose: Admin user authentication

MQ_APP_PASSWORD:
  Secret: mq-app-credentials
  Key: password
  Purpose: Application user authentication
```

**Default User Credentials:**
- **Admin User:** `admin` (password from `mq-web-credentials` secret)
- **App User:** `app` (password: `<app-password>`)

---

## Health Monitoring

### Liveness Probe

```yaml
Command: chkmqhealthy
Schedule: Every 10 seconds (periodSeconds)
Initial Delay: 60 seconds (initialDelaySeconds)
Failure Threshold: 3 failed checks
Timeout: 1 second
```

**Purpose:** Restarts pod if queue manager becomes unresponsive

### Readiness Probe

```yaml
Command: chkmqready
Schedule: Every 5 seconds (periodSeconds)
Initial Delay: 10 seconds (initialDelaySeconds)
Failure Threshold: 3 failed checks
Timeout: 1 second
```

**Purpose:** Removes pod from service endpoints if not ready to receive traffic

---

## Resource Management

### CPU & Memory Allocation

| Resource | Request | Limit | Purpose |
|----------|---------|-------|---------|
| **CPU** | 500m (0.5 core) | 2 cores | Guaranteed minimum; can burst up to 2 cores |
| **Memory** | 512Mi | 2Gi | Guaranteed minimum; max 2GB heap allocation |

**Recommendation:** For production, increase CPU request to 1-2 cores and memory request to 1Gi.

---

## Security Context

```yaml
fsGroup: 999
  - Sets GID ownership to 999 (mqm user group)
  
fsGroupChangePolicy: OnRootMismatch
  - Recursively changes ownership on mount if not matching
```

**Security Model:** Container runs as non-root (mqm user/group 999)

---

## Default Objects & Connectivity

### Available Queues

By default, the following queues are available for messaging:

| Queue Name | Type | Purpose |
|-----------|------|---------|
| `SYSTEM.DEFAULT.MODEL.QUEUE` | Model Queue | Default queue template |
| Custom queues | Local Queues | Application-specific queues (must be created) |

### Required Server Connection Channel

To connect applications, you must define/configure:

```
Channel Name: DEV.APP.SVRCONN
Type: Server Connection Channel
Authentication: app / <app-password>
```

---

## Connection Strings

### For Java Applications (JNDI/JMS)

```properties
host.name=ibm-mq-auth.default.svc.cluster.local
port=1414
queue.manager=QM01
channel=DEV.APP.SVRCONN
user.name=app
user.password=<app-password>
```

### For Node.js/Go Applications

```javascript
{
  host: 'ibm-mq-auth.default.svc.cluster.local',
  port: 1414,
  queueManager: 'QM01',
  channel: 'DEV.APP.SVRCONN',
  userId: 'app',
  password: '<app-password>'
}
```

### For Other MQ Clients

```
Connection String:
QM01@ibm-mq-auth.default.svc.cluster.local(1414)
```

---

## Web Console Access

| Property | Value |
|----------|-------|
| **URL** | `https://localhost:9443/ibmmq/console` |
| **Protocol** | HTTPS (Self-signed certificate) |
| **Username** | `admin` |
| **Password** | From `mq-web-credentials` secret |
| **Port Mapping** | Internal 9443 → External 9443 |

**Note:** Ignore self-signed certificate warnings in development

---

## Secrets Management

### Required Kubernetes Secrets

**Secret 1: mq-web-credentials**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mq-web-credentials
type: Opaque
stringData:
  password: <admin-password>  
```

**Secret 2: mq-app-credentials**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mq-app-credentials
type: Opaque
stringData:
  password: <app-password>
```

---

## Update & Rollback Strategy

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    partition: 0  # All pods participate in rolling update
```

**Process:** When updating the image/configuration:
1. Pod terminates gracefully (30-second grace period)
2. New pod created with updated spec
3. Persistent volume automatically re-attached

**Grace Period:** 30 seconds for clean shutdown

---

## Pod Lifecycle Management

| Phase | Behavior |
|-------|----------|
| **Creation** | Service DNS name registered after pod ready |
| **Running** | Continuous monitoring via liveness/readiness probes |
| **Termination** | 30-second grace period for graceful shutdown |
| **Deletion** | PVC retained per retention policy |

---

## Common Operations

### Check Queue Manager Status

```bash
kubectl exec -it ibm-mq-auth-0  -- dspmq
```

### View MQ Logs

```bash
kubectl logs -f ibm-mq-auth-0 
```

### Access Container Shell

```bash
kubectl exec -it ibm-mq-auth-0 -- /bin/bash
```

### Create Custom Queues

```bash
kubectl exec -it ibm-mq-auth-0  -- runmqsc QM01
# Then use MQSC commands:
# DEFINE QLOCAL(MY.QUEUE.NAME)
# DISPLAY Q(MY.QUEUE.NAME)
```

### Restart Queue Manager

```bash
kubectl delete pod ibm-mq-auth-0 -n efp-nprd
# StatefulSet will automatically recreate
```
---

## Troubleshooting Guide

| Issue | Cause | Solution |
|-------|-------|----------|
| Pod stuck in `Pending` | No persistent volume available | Check PV availability; ensure storage class exists |
| `CrashLoopBackOff` | License not accepted | Verify `LICENSE=accept` environment variable |
| Connection refused | Channel not configured | Create `DEV.APP.SVRCONN` channel via MQSC |
| Web console unreachable | Port not exposed | Verify service port mapping (9443) |
| Metrics unavailable | Metrics not enabled | Add `MQ_ENABLE_METRICS=true` environment variable |

---

## Performance Tuning Recommendations

### For Production Environment

```yaml
resources:
  requests:
    cpu: "2"          # Increase from 500m
    memory: 2Gi       # Increase from 512Mi
  limits:
    cpu: "4"
    memory: 4Gi
```

### For High Throughput Scenarios

```yaml
# Add to environment:
- name: MQ_QMGR_LOG_FILE_PAGES
  value: "16384"     # Larger log file size
```

---

## Disaster Recovery

### Backup Strategy

- **Volume Snapshots:** Use CSI snapshot controller for PVC snapshots
- **Frequency:** Daily snapshots recommended
- **Retention:** Keep last 7 snapshots

### Recovery Process

1. Create new PVC from snapshot
2. Delete existing StatefulSet
3. Update volumeClaimTemplate with snapshot reference
4. Recreate StatefulSet

---

## Monitoring & Alerting

### Key Metrics to Monitor

```
ibm_mq_queue_depth_current        # Messages in queue
ibm_mq_queue_open_input_count     # Active consumers
ibm_mq_put_rate                   # Message put rate
ibm_mq_get_rate                   # Message get rate
ibm_mq_inq_count                  # Inquiries
```

### Enable Prometheus Metrics

```yaml
env:
- name: MQ_ENABLE_METRICS
  value: "true"
```

Metrics available at: `https://localhost:9157/metrics`

---

## Related Resources

- **Queue Manager:** `QM01`
- **Image Repository:** quay.io/aelfeki007/ibm-messaging/mq

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-06-14 | Initial documentation | MQ Configuration |
| - | Configuration with deprecated vars | - |

---

## Support & References

- **Official Docs:** https://github.com/ibm-messaging/mq-container
- **IBM MQ Documentation:** https://www.ibm.com/docs/en/ibm-mq/9.4
- **Container Image:** quay.io/aelfeki007/ibm-messaging/mq:9.4.5.0-r2
