# Deployment Guide: EKS Microservices Architecture

## 📋 Descripción General

Este documento contiene los manifiestos Kubernetes (YAML) necesarios para desplegar el microservicio `get-products` en Amazon EKS con:

- ✅ Escalabilidad automática (HPA)
- ✅ Seguridad avanzada (RBAC, Network Policies)
- ✅ Alta disponibilidad Multi-AZ
- ✅ Monitoring con CloudWatch

## 📁 Archivos Incluidos

```
01-deployment-get-products.yaml    # Deployment del microservicio
02-service-get-products.yaml       # Service ClusterIP
03-hpa-get-products.yaml          # Horizontal Pod Autoscaler
04-namespace-rbac.yaml            # Namespace, RBAC, Network Policies
05-configmap-secrets.yaml         # Configuración y secretos
06-integracion-mecanismos.md      # Documentación técnica
00-README.md                      # Este archivo
```

---

## 🚀 Prerequisitos

### 1. EKS Cluster Operacional
```bash
# Verificar conexión al cluster
kubectl cluster-info
kubectl get nodes

# Confirmar que hay nodos en 2 AZ
kubectl get nodes -L topology.kubernetes.io/zone

# Ejemplo de salida esperada:
# NAME                           STATUS   ROLES    AGE   VERSION   ZONE
# ip-10-0-2-xxx.ec2.internal    Ready    <none>   10d   v1.28.0   us-east-1a
# ip-10-0-5-xxx.ec2.internal    Ready    <none>   10d   v1.28.0   us-east-1b
```

### 2. AWS CLI Configurado
```bash
aws configure
aws sts get-caller-identity  # Verificar credenciales
```

### 3. Metrics Server Instalado
```bash
# Verificar que está instalado (requerido para HPA)
kubectl get deployment -n kube-system | grep metrics-server

# Si no existe, instalar:
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### 4. Amazon ECR Repository
```bash
# Crear repositories para las imágenes Docker
aws ecr create-repository --repository-name frontend --region us-east-1
aws ecr create-repository --repository-name get-products --region us-east-1
aws ecr create-repository --repository-name create-product --region us-east-1
aws ecr create-repository --repository-name update-product --region us-east-1
aws ecr create-repository --repository-name delete-product --region us-east-1

# Obtener URI de los repositories
aws ecr describe-repositories --query 'repositories[*].[repositoryName,repositoryUri]' --output table
```

### 5. RDS MySQL Cluster (Multi-AZ)
```bash
# Verificar que existe
aws rds describe-db-clusters --db-cluster-identifier escolaronline-cluster

# Obtener información de conexión
aws rds describe-db-clusters \
  --db-cluster-identifier escolaronline-cluster \
  --query 'DBClusters[0].[DBClusterIdentifier,Endpoint,ReaderEndpoint,Port,Engine]' \
  --output table
```

---

## 📝 Paso 1: Preparar Configuración

### 1.1 Actualizar valores en los manifiestos

```bash
# 1. Actualizar ECR endpoint en 01-deployment-get-products.yaml
# Cambiar:
# image: 123456789.dkr.ecr.us-east-1.amazonaws.com/get-products:latest
# Por tu cuenta AWS y repositorio actual

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION=us-east-1
ECR_ENDPOINT=${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com

# Reemplazar en el archivo
sed -i "s|123456789.dkr.ecr.us-east-1.amazonaws.com|${ECR_ENDPOINT}|g" 01-deployment-get-products.yaml
```

### 1.2 Actualizar RDS endpoint y credenciales

```bash
# Obtener endpoint del RDS
RDS_ENDPOINT=$(aws rds describe-db-clusters \
  --db-cluster-identifier escolaronline-cluster \
  --query 'DBClusters[0].Endpoint' --output text)

echo "RDS Endpoint: $RDS_ENDPOINT"

# Actualizar en 05-configmap-secrets.yaml
# ⚠️  IMPORTANTE: Usar AWS Secrets Manager en producción, NO hardcodear credenciales
```

### 1.3 Crear IAM Role para EKS Pods (IRSA)

```bash
# Este rol permite que los pods accedan a ECR y CloudWatch
# Sin necesidad de almacenar credenciales en Secrets

cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/oidc.eks.${REGION}.amazonaws.com/id/$(aws eks describe-cluster --name your-cluster-name --query cluster.identity.oidc.issuer --output text | cut -d '/' -f 5)"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.${REGION}.amazonaws.com/id/...:sub": "system:serviceaccount:escolaronline:get-products"
        }
      }
    }
  ]
}
EOF

# Crear el role
aws iam create-role \
  --role-name eks-get-products-role \
  --assume-role-policy-document file://trust-policy.json

# Asignar políticas
aws iam attach-role-policy \
  --role-name eks-get-products-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

aws iam attach-role-policy \
  --role-name eks-get-products-role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
```

---

## 🔧 Paso 2: Aplicar Manifiestos

### 2.1 Crear Namespace y RBAC

```bash
kubectl apply -f 04-namespace-rbac.yaml

# Verificar
kubectl get namespace escolaronline
kubectl get serviceaccount -n escolaronline
kubectl get roles -n escolaronline
```

### 2.2 Crear Secrets y ConfigMaps

```bash
# ⚠️  IMPORTANTE: Actualizar credenciales en 05-configmap-secrets.yaml antes de aplicar

kubectl apply -f 05-configmap-secrets.yaml

# Verificar
kubectl get secrets -n escolaronline
kubectl get configmaps -n escolaronline

# Ver contenido de Secret (solo el host, no el password)
kubectl get secret mysql-credentials -n escolaronline -o jsonpath='{.data.host}' | base64 -d
```

### 2.3 Aplicar Deployment, Service y HPA

```bash
# Aplicar Deployment
kubectl apply -f 01-deployment-get-products.yaml

# Verificar pods están corriendo
kubectl get pods -n escolaronline -w

# Esperar a que todos los pods sean READY
# Esto puede tomar 1-2 minutos

# Aplicar Service
kubectl apply -f 02-service-get-products.yaml

# Verificar Service
kubectl get service -n escolaronline
kubectl get endpoints get-products -n escolaronline

# Aplicar HPA
kubectl apply -f 03-hpa-get-products.yaml

# Verificar HPA
kubectl get hpa -n escolaronline
```

### 2.4 Verificar Estado Completo

```bash
# Revisar todos los recursos
kubectl get all -n escolaronline

# Revisar detalles del Deployment
kubectl describe deployment get-products -n escolaronline

# Ver logs del microservicio
kubectl logs -n escolaronline -l app=get-products --tail=50 -f
```

---

## ✅ Paso 3: Validación

### 3.1 Validar Pods Corren Correctamente

```bash
# Ver status de todos los pods
kubectl get pods -n escolaronline -o wide

# Ver en qué nodo corre cada pod (debe estar distribuido)
kubectl get pods -n escolaronline -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,ZONE:.spec.nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution

# Ver logs de un pod específico
kubectl logs -n escolaronline get-products-xxxxx

# Acceder a un pod para debugging
kubectl exec -it -n escolaronline get-products-xxxxx -- /bin/sh
```

### 3.2 Validar HPA Funciona

```bash
# Ver estado del HPA
kubectl get hpa -n escolaronline -w

# Ver métricas de CPU en tiempo real
kubectl top pods -n escolaronline
kubectl top nodes

# Simular carga para probar escalado
kubectl run -it --image=busybox:1.28 load-generator --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://get-products:3001/products; done"

# En otra terminal, ver como escala
kubectl get hpa -n escolaronline -w
kubectl get pods -n escolaronline -w
```

### 3.3 Validar Seguridad

```bash
# Verificar RBAC
kubectl auth can-i get pods --as=system:serviceaccount:escolaronline:get-products -n escolaronline  # true
kubectl auth can-i delete pods --as=system:serviceaccount:escolaronline:get-products -n escolaronline  # false

# Verificar Network Policy
kubectl get networkpolicies -n escolaronline
kubectl describe netpol get-products-netpol -n escolaronline

# Intentar ejecutar como root (debe fallar)
kubectl exec -u 0 -it -n escolaronline get-products-xxxxx -- id
# uid=1000(nonroot) gid=1000(nonroot) groups=1000(nonroot)

# Verificar volumenes read-only
kubectl exec -it -n escolaronline get-products-xxxxx -- touch /test.txt
# Read-only file system error
```

### 3.4 Validar Conectividad a RDS

```bash
# Desde dentro del cluster
kubectl run -it --image=mysql:8.0 debug --restart=Never -- mysql -h $RDS_ENDPOINT -u admin -p -e "SELECT VERSION();"

# O desde la aplicación, verificar logs de conexión
kubectl logs -n escolaronline -l app=get-products | grep -i "database\|connection"
```

### 3.5 Validar High Availability

```bash
# Ver distribución de pods en múltiples AZ
kubectl get pods -n escolaronline -o wide | grep -E "us-east-1a|us-east-1b"

# Obtener nodos en cada AZ
kubectl get nodes --show-labels | grep topology.kubernetes.io/zone

# Simular fallos: terminar un pod
POD_NAME=$(kubectl get pods -n escolaronline -o name | head -1)
kubectl delete $POD_NAME -n escolaronline

# Verificar que se recrea automáticamente
kubectl get pods -n escolaronline -w
```

---

## 📊 Paso 4: Monitoreo (CloudWatch)

### 4.1 Ver Métricas en CloudWatch

```bash
# CloudWatch Container Insights (si está habilitado)
aws logs describe-log-groups --query 'logGroups[?contains(logGroupName, `eks`)].logGroupName' --output table

# Ver métricas de Container Insights
aws cloudwatch get-metric-statistics \
  --namespace ContainerInsights \
  --metric-name PodCpuUtilization \
  --dimensions Name=PodName,Value=get-products Name=Namespace,Value=escolaronline \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average
```

### 4.2 Crear Dashboard en CloudWatch

```bash
# Dashboard para monitorear la aplicación
aws cloudwatch put-dashboard \
  --dashboard-name EKS-GetProducts \
  --dashboard-body file://dashboard.json
```

### 4.3 Alertas Recomendadas

```bash
# Crear alerta para CPU alta
aws cloudwatch put-metric-alarm \
  --alarm-name get-products-cpu-high \
  --alarm-description "Alert when CPU > 80%" \
  --metric-name PodCpuUtilization \
  --namespace ContainerInsights \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=PodName,Value=get-products Name=Namespace,Value=escolaronline

# Crear alerta para error rate
aws cloudwatch put-metric-alarm \
  --alarm-name get-products-error-rate \
  --alarm-description "Alert when error rate > 1%" \
  --metric-name HTTPRequestCount \
  --namespace AWS/ELB \
  --statistic Sum \
  --period 300 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold
```

---

## 🔄 Paso 5: Actualizar Aplicación (Rolling Update)

```bash
# Actualizar imagen (nueva versión disponible en ECR)
kubectl set image deployment/get-products \
  get-products=${ECR_ENDPOINT}/get-products:v2.0 \
  -n escolaronline

# Monitorear el update
kubectl rollout status deployment/get-products -n escolaronline -w

# Ver history de deployments
kubectl rollout history deployment/get-products -n escolaronline

# Hacer rollback si hay problemas
kubectl rollout undo deployment/get-products -n escolaronline
```

---

## 🧹 Paso 6: Limpieza

```bash
# Eliminar todos los recursos
kubectl delete namespace escolaronline

# Verificar que se eliminaron
kubectl get namespace escolaronline
# Error: namespaces "escolaronline" not found
```

---

## 🐛 Troubleshooting

### Pods en estado CrashLoopBackOff

```bash
# Ver por qué el pod está crasheando
kubectl describe pod <pod-name> -n escolaronline

# Ver logs del contenedor
kubectl logs <pod-name> -n escolaronline --previous
```

### HPA no escala

```bash
# Verificar que metrics-server está corriendo
kubectl get deployment -n kube-system metrics-server

# Ver eventos del HPA
kubectl describe hpa get-products-hpa -n escolaronline

# Ver métricas disponibles
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq .
```

### No puede conectar a RDS

```bash
# Verificar Security Groups
aws ec2 describe-security-groups --group-ids sg-xxxxx

# Verificar que el Security Group del RDS permite entrada desde SG-EKS
# SG-RDS debe tener regla: 3306 TCP desde SG-EKS

# Probar conectividad desde pod
kubectl exec -it get-products-xxxxx -n escolaronline -- \
  nc -zv $RDS_ENDPOINT 3306
```

### Pods no pueden bajar imágenes de ECR

```bash
# Verificar que los nodos pueden aceder a ECR
aws ecr get-authorization-token --region us-east-1

# Ver imagePullSecrets en el pod
kubectl describe pod <pod-name> -n escolaronline | grep ImagePullSecrets
```

---

## 📚 Referencias Adicionales

- [Kubernetes Deployment Concepts](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [HPA Best Practices](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)
- [AWS RDS Multi-AZ](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html)

---

**Versión:** 1.0  
**Última actualización:** 2024  
**Autor:** Equipo de Arquitectura DuocUC
