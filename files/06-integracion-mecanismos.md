# Integración de Mecanismos: Escalabilidad, Seguridad y Alta Disponibilidad

## 1. ESCALABILIDAD AUTOMÁTICA

### Implementación
La escalabilidad automática en esta arquitectura se implementa mediante dos niveles:

#### 1.1 Horizontal Pod Autoscaler (HPA)
**Recurso:** `HorizontalPodAutoscaler` (02-hpa-get-products.yaml)
- **Métricas:** CPU (70%) y Memoria (80%)
- **Replicas:** Mínimo 2, Máximo 5
- **Escalado en aumento:** Inmediato (0 segundos), sube 100% o 2 pods
- **Escalado en disminución:** 5 minutos de estabilización, baja 50% o 1 pod
- **Comportamiento:** Asegura rendimiento bajo carga sin sobreasignación de recursos

#### 1.2 Cluster Autoscaler (Infraestructura)
**Servicio:** AWS EC2 Auto Scaling Group (EKS Node Group)
- **Nodos:** 2-4 distribuidos en AZ1a y AZ1b
- **Tipo:** t3.medium (2 vCPU, 4 GB RAM)
- **Política:** Scale-up cuando hay pods pending, scale-down cuando subutilizado
- **Tiempo de ajuste:** 3-5 minutos

### Flujo de Escalado
```
CPU > 70% por 1 minuto 
  ↓
HPA crea nuevos pods (hasta máx 5)
  ↓
Si no hay recursos en nodos
  ↓
Cluster Autoscaler escala EKS Node Group
  ↓
Nuevos nodos se unen al cluster
  ↓
Pods se asignan a nuevos nodos
  ↓
Tráfico se distribuye automáticamente
```

---

## 2. SEGURIDAD

### 2.1 Control de Acceso (RBAC)
**Archivo:** 04-namespace-rbac.yaml

#### ServiceAccount
- Identidad pod específica para `get-products`
- No utiliza credenciales de nodo
- Permisos mínimos necesarios (Least Privilege)

#### Role Permissions
```yaml
- configmaps/secrets: Solo "mysql-config" y "mysql-credentials"
- pods: Solo lectura y list
- logs: Acceso a logs propios
```

#### Beneficio
Limita acceso a recursos del cluster y previene escalada de privilegios.

### 2.2 Network Policy
**Archivo:** 04-namespace-rbac.yaml

#### Ingress Rules
- **Fuente:** ALB Ingress Controller y pods en tier: backend
- **Puerto:** 3001 (TCP)
- Bloquea tráfico no autorizado entre pods

#### Egress Rules
- **DNS:** Permite resolución (Puerto 53/UDP)
- **RDS:** Solo puerto 3306/TCP (MySQL)
- **ECR/CloudWatch:** Puerto 443/HTTPS
- Previene data exfiltration y comunicación no autorizada

### 2.3 Pod Security
**Implementado en Deployment:**

```yaml
securityContext:
  runAsNonRoot: true          # No ejecuta como root
  runAsUser: 1000             # Usuario no privilegiado
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]             # Elimina todas las capacidades Linux
```

**Beneficios:**
- Limita impacto de compromisos de contenedor
- Previene cambios de filesystem
- Reduce superficie de ataque

### 2.4 Security Groups (VPC)
**En Infraestructura:**

#### SG-ALB
- **Ingress:** 0.0.0.0/0:80, 0.0.0.0/0:443 (Internet)
- **Egress:** Solo hacia SG-EKS:3001

#### SG-EKS
- **Ingress:** SG-ALB:3001, SG-EKS:* (pod-to-pod)
- **Egress:** SG-RDS:3306, NAT Gateway:* (outbound)

#### SG-RDS
- **Ingress:** SG-EKS:3306 (Solo pods EKS)
- **Egress:** Ninguno (base de datos)

**Resultado:** Tráfico segmentado, solo acceso necesario entre componentes.

### 2.5 Secrets Management
**Archivo:** 05-configmap-secrets.yaml

#### Recomendaciones Producción
```yaml
# Usar AWS Secrets Manager + External Secrets Operator

apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-store
  namespace: escolaronline
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-operator

---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: mysql-credentials
  namespace: escolaronline
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-store
    kind: SecretStore
  target:
    name: mysql-credentials
    creationPolicy: Owner
  data:
  - secretKey: host
    remoteRef:
      key: rds/mysql/host
  - secretKey: username
    remoteRef:
      key: rds/mysql/username
  - secretKey: password
    remoteRef:
      key: rds/mysql/password
```

**Ventajas:**
- Secrets encriptados en AWS KMS
- Rotación automática
- Auditoría con CloudTrail
- No expone secrets en git

---

## 3. ALTA DISPONIBILIDAD (HA)

### 3.1 Redundancia en Múltiples AZ

#### Distribución de Nodos EKS
```
AZ1a                           AZ1b
├─ EKS Node 1 (t3.medium)      ├─ EKS Node 2 (t3.medium)
│  ├─ Pod: frontend            │  ├─ Pod: update-product
│  ├─ Pod: get-products        │  └─ Pod: delete-product
│  └─ Pod: create-product      │
└─ RDS Primary (MySQL)         └─ RDS Standby (Replica)
```

**Beneficio:** Fallos de zona completa no afectan servicio.

#### RDS Multi-AZ
- **Primary:** AZ1a (escritura)
- **Standby:** AZ1b (lectura y failover automático)
- **RPO:** 0 (replicación síncrona)
- **RTO:** < 2 minutos (failover automático)

### 3.2 Pod Anti-Affinity
**Implementado en Deployment:**

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - get-products
        topologyKey: kubernetes.io/hostname
```

**Resultado:** Evita que múltiples pods de `get-products` se ejecuten en el mismo nodo.

### 3.3 Probes de Salud
**Liveness Probe:**
```yaml
httpGet:
  path: /health
  port: 3001
initialDelaySeconds: 30
periodSeconds: 10
failureThreshold: 3
```
- Reinicia pod si no responde (detecta deadlocks)

**Readiness Probe:**
```yaml
httpGet:
  path: /ready
  port: 3001
initialDelaySeconds: 10
periodSeconds: 5
failureThreshold: 2
```
- Remueve pod de balanceo si no está ready (previene tráfico a pods enfermo)

### 3.4 Resource Requests and Limits
**Garantiza estabilidad:**

```yaml
requests:
  cpu: 250m
  memory: 256Mi
limits:
  cpu: 500m
  memory: 512Mi
```

**Beneficios:**
- Scheduler garantiza recursos disponibles
- Previene starving de otros pods
- Limita consumo máximo (OOMKill si excede)

### 3.5 Rolling Update Strategy
**Deployment:**

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

**Proceso:**
1. Crea 1 pod nuevo con versión nueva
2. Espera a que sea ready
3. Remueve 1 pod viejo
4. Repite hasta todos actualizado

**Resultado:** Cero downtime durante deployments.

### 3.6 ALB Ingress (Punto de entrada único)
**Características:**
- **Multi-AZ:** ALB distribuido en AZ1a y AZ1b
- **Target Groups:** Automáticamente actualizado con pods EKS
- **Health Checks:** Verifica readiness de pods
- **Connection Draining:** 30 segundos para conexiones activas

### 3.7 Monitoreo y Alertas (CloudWatch)
**Métricas críticas:**

```
Pod Metrics:
- CPU utilization (%) → Trigger HPA
- Memory utilization (%)
- Network I/O
- Request latency
- Error rate (4xx, 5xx)

Node Metrics:
- CPU, Memory, Disk utilization
- Network I/O
- Pod count

RDS Metrics:
- Connection count
- Query latency
- Replication lag
- CPU/Memory/Storage utilization
```

**Alertas recomendadas:**
```
- CPU > 85% por 5 min → Page On-Call
- Memory > 90% → Escalamiento manual
- Error rate > 1% → Investigation
- Replica lag > 10s → Database team
```

---

## 4. FLUJO INTEGRADO: Escenarios de Carga

### Escenario 1: Aumento de Tráfico (100 → 1000 RPS)

```
Tiempo    Evento                          Estado
────────────────────────────────────────────────────────
T=0       CPU: 70%                        2 pods running
T=30s     Incoming requests aumentan     CPU → 85%
T=1m      HPA dispara: crear pod 3       3 pods running
T=2m      RPS = 500, CPU = 75%           3 pods healthy
T=3m      RPS = 1000, CPU = 80%          HPA crea pod 4 y 5
T=4m      HPA activa: 5 pods max         Load balanced
          Cluster Autoscaler             Sin necesidad escalado nodo
          verifica recursos              (t3.medium tiene 8GB RAM)

Resultado: 5× capacidad en 4 minutos sin downtime
```

### Escenario 2: Fallos y Recuperación

```
Escenario A: Fallo de Pod
──────────────────────────────
T=0       Pod 2 crash (OOMKill)
T=0s      Liveness probe falla 3 veces
T=30s     Kubelet reinicia Pod 2
T=35s     Readiness probe pasa
T=40s     ALB incluye Pod 2 en target
Resultado: 40 segundos de degraded (4/5 pods)
          Tráfico re-routed automático

Escenario B: Fallo de Nodo (AZ1a)
───────────────────────────────────
T=0       EKS Node 1 falla (hardware)
T=5m      Controller Manager detecta
          Pod 1, 2, 3 no responding
T=6m      Pods rescheduleados a Node 2 (AZ1b)
T=8m      Todos los pods ready
T=10m     Cluster Autoscaler reemplaza Node 1
Resultado: Full recovery en 10 minutos

Escenario C: Fallo de RDS Primary
──────────────────────────────────
T=0       RDS Primary (AZ1a) falla
T=1s      Connection pooling detecta
T=2m      RDS Standby promovido automático
          Nuevo Standby creado en AZ1a
T=3m      Apps reconectan automático
Resultado: ~2 minutos downtime
          RPO = 0 (sin pérdida de datos)
```

### Escenario 3: Deployment de Nueva Versión

```
Rolling Update: get-products v1 → v2
─────────────────────────────────────
Inicial:   Pod-A (v1), Pod-B (v1), Pod-C (v1), Pod-D (v1), Pod-E (v1)
T=0m       Crear Pod-F (v2)
T=1m       Pod-F ready + in ALB target group
T=1.5m     Terminar Pod-A (v1), conexiones drain 30s
T=2m       Pod-A terminado, crear Pod-G (v2)
T=3m       Pod-G ready, terminar Pod-B (v1)
...
T=6m       Todos v2, proceso completo

Resultado: 5 pods simultáneamente disponibles
          Cero conexiones perdidas
          Rollback disponible si error
```

---

## 5. MONITOREO Y VALIDACIÓN

### Comandos de Validación

```bash
# Verificar HPA activo
kubectl get hpa -n escolaronline
kubectl describe hpa get-products-hpa -n escolaronline

# Verificar Pod distribution
kubectl get pods -n escolaronline -o wide
# Confirmar pods en diferentes nodos y AZ

# Verificar Network Policy
kubectl get networkpolicies -n escolaronline
kubectl describe netpol get-products-netpol -n escolaronline

# Verificar Probes
kubectl get deployment get-products -n escolaronline -o yaml | grep -A 10 "Probe"

# Verificar recursos asignados
kubectl top pods -n escolaronline
kubectl top nodes

# Verificar RDS replication
aws rds describe-db-clusters --db-cluster-identifier escolaronline-cluster
# Verificar "ReplicationDelay" y "DBClusterMembers"

# Verificar Security Groups
aws ec2 describe-security-groups --filters "Name=group-name,Values=SG-EKS"

# Ver logs de HPA
kubectl logs -n kube-system deployment/metrics-server
```

---

## 6. MATRIZ DE INTEGRACIÓN

| Mecanismo | Componente | Implementación | Validación |
|-----------|-----------|-----------------|------------|
| **Escalabilidad** | HPA | 3-hpa-get-products.yaml | `kubectl get hpa` |
| | Cluster Autoscaler | EKS Node Group | `kubectl get nodes` |
| | Resource Requests | 1-deployment-get-products.yaml | `kubectl top pods` |
| **Seguridad** | RBAC | 4-namespace-rbac.yaml | `kubectl auth can-i` |
| | Network Policy | 4-namespace-rbac.yaml | `kubectl get netpol` |
| | Pod Security | 1-deployment-get-products.yaml | `kubectl exec` denied |
| | Secrets | 5-configmap-secrets.yaml | AWS Secrets Manager |
| | Security Groups | VPC Infra | aws ec2 describe-security-groups |
| **Alta Disponibilidad** | Multi-AZ | VPC + EKS + RDS | `kubectl get nodes -L topology.kubernetes.io/zone` |
| | Anti-Affinity | 1-deployment-get-products.yaml | `kubectl get pods -o wide` |
| | Probes | 1-deployment-get-products.yaml | `kubectl describe pod` |
| | Backup RDS | RDS Multi-AZ | aws rds describe-db-clusters |
| | Ingress ALB | AWS ELB | aws elbv2 describe-target-groups |

---

## 7. RESUMEN EJECUTIVO

Esta arquitectura integra:

✅ **Escalabilidad Automática**
   - HPA responde a demanda en minutos
   - Cluster Autoscaler escala infraestructura

✅ **Seguridad en Capas**
   - RBAC limita acceso en cluster
   - Network Policy controla tráfico
   - Pods ejecutan no-root, read-only filesystem
   - Secrets encriptados en AWS KMS

✅ **Alta Disponibilidad**
   - Multi-AZ distribution (RTO < 2min, RPO = 0)
   - Pod auto-recovery (liveness probes)
   - Rolling deployments sin downtime
   - Health checks garantizan traffic a pods sanos

**Resultado:** Aplicación production-ready, resiliente ante fallos, segura, escalable.
