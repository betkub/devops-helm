# engage-worker-helm-chart

Helm chart สำหรับ deploy **background worker ที่ไม่มี HTTP listener** —
render เฉพาะ `Deployment` เท่านั้น (ไม่มี Service, ไม่มี Ingress, ไม่มี HPA)

Use case แรก: `spike-routing-dispatcher` (worker จาก image เดียวกับ service อื่น
รันด้วย env `ROLE=dispatcher`, health/heartbeat เขียนลง Postgres — ไม่มี endpoint ให้ probe)

## ทำไมต้องแยก chart จาก engage-helm-chart

- `engage-helm-chart` บังคับสร้าง Service + probe (http/grpc) + HPA
  ซึ่งใช้ไม่ได้กับ worker ที่ไม่เปิด port
- worker แบบ **single-writer** (Postgres advisory-lock leader election)
  ต้องรัน **1 replica เท่านั้น** และห้ามมี HPA มา scale เพิ่ม
- chart นี้ใช้ `strategy: Recreate` — pod เก่าถูกหยุดก่อนสร้าง pod ใหม่เสมอ
  กัน rolling update ทำให้ dispatcher รันซ้อนกัน 2 ตัวชั่วขณะ

## ⚠️ ข้อควรระวัง: PgBouncer

Advisory lock ของ Postgres ผูกกับ **session** — worker ที่ใช้ leader election แบบนี้
**ต้องต่อ Postgres โดยตรงหรือผ่าน pooler แบบ session mode เท่านั้น**
ห้ามชี้ `DATABASE_URL` ไปที่ PgBouncer transaction pooling เด็ดขาด
(lock จะหลุด/สลับ connection แล้ว single-writer guarantee พัง)

## Values

โครงเดียวกับ `engage-helm-chart` (name, image.name, image.tag, resources,
nodeSelector, tolerations, affinity, podAnnotations, configmap.enabled,
secret.enabled) แต่ตัด containerPort / grpcPort / healthCheck* / hpa / ingress ออก
และเพิ่ม:

| key | default | ความหมาย |
|---|---|---|
| `replicas` | `1` | จำนวน pod — single-writer ต้องคงไว้ที่ 1 |
| `terminationGracePeriodSeconds` | `60` | เวลารอ graceful shutdown หลัง SIGTERM |
| `livenessProbe` | `{}` | ไม่มี probe เป็น default; ใส่ spec เต็ม (exec/tcpSocket) ได้ |
| `readinessProbe` | `{}` | เช่นเดียวกับ livenessProbe |

env ถูก inject ผ่าน `envFrom` จาก ConfigMap และ Secret **ชื่อเดียวกับ `.Values.name`**
(เปิด/ปิดด้วย `configmap.enabled` / `secret.enabled`)

## ตัวอย่าง: spike-routing-dispatcher

```yaml
name: "spike-routing-dispatcher"
image:
  name: "716687806003.dkr.ecr.ap-southeast-1.amazonaws.com/spike-routing"
  tag: "develop-0000000"
replicas: 1
terminationGracePeriodSeconds: 60
nodeSelector:
  tier: "betkub"
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 150m
    memory: 192Mi
configmap:
  enabled: true   # ConfigMap "spike-routing-dispatcher" ต้องมี ROLE=dispatcher
secret:
  enabled: true   # Secret "spike-routing-dispatcher" เก็บ DATABASE_URL (session mode!), REDIS_URL
podAnnotations: {}
tolerations: []
affinity: {}
```
