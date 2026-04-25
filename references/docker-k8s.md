# Referência: Docker / Kubernetes

## Docker

### Diagnóstico Completo
```bash
# Versão e configuração
docker version
docker info

# Listar containers em execução
docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}"

# Verificar containers privilegiados (CRÍTICO)
docker ps -q | xargs docker inspect --format '{{.Name}}: Privileged={{.HostConfig.Privileged}}'

# Verificar montagens de volumes sensíveis
docker ps -q | xargs docker inspect --format '{{.Name}}: {{.HostConfig.Binds}}'

# Verificar containers rodando como root
docker ps -q | xargs -I{} docker exec {} whoami 2>/dev/null

# Verificar imagens com vulnerabilidades
docker scan IMAGE_NAME  # Snyk
trivy image IMAGE_NAME  # Trivy (recomendado)

# Verificar configuração do daemon
cat /etc/docker/daemon.json
```

---

### 🔴 CRÍTICO — Docker

#### Container Privilegiado
```bash
# Diagnóstico
docker inspect CONTAINER_ID | grep -i "Privileged"

# Remediação: NUNCA usar --privileged em produção
# Substituir por capabilities específicas:
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE IMAGE
```

#### Docker Socket Exposto
```bash
# Verificar se /var/run/docker.sock está montado em container
docker ps -q | xargs docker inspect | grep "docker.sock"

# Socket exposto = root no host. Remover imediatamente.
# Se necessário, usar proxy de socket restrito: docker-socket-proxy
```

#### Imagens sem tag específica (latest)
```bash
# Listar imagens usando 'latest'
docker images | grep latest

# Vulnerabilidade: impossível rastrear versão e CVEs
# Sempre usar tags específicas: nginx:1.25.3-alpine
```

---

### 🟠 ALTO — Docker

#### Usuário root dentro do container
```dockerfile
# ❌ Ruim — roda como root
FROM node:18
COPY . .
CMD ["node", "app.js"]

# ✅ Bom — criar usuário não-root
FROM node:18-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --chown=appuser:appgroup . .
USER appuser
CMD ["node", "app.js"]
```

#### Secrets em variáveis de ambiente / Dockerfile
```bash
# Verificar secrets em imagens
docker history --no-trunc IMAGE_NAME | grep -i "ENV\|ARG"
docker inspect IMAGE_NAME | grep -i "env"

# Verificar .env commitado
find . -name ".env" -not -path "./.git/*"

# Remediação: usar Docker Secrets ou gestor de secrets
docker secret create minha_senha - < senha.txt
```

#### Network — containers na mesma rede padrão
```bash
# Verificar redes
docker network ls
docker network inspect bridge

# Containers na rede bridge padrão podem se comunicar entre si
# Criar redes isoladas por serviço:
docker network create --driver bridge app-network
docker network create --driver bridge db-network
```

---

### 🟡 MÉDIO — Docker

#### Dockerfile — boas práticas de segurança
```dockerfile
# ✅ Dockerfile seguro
FROM node:18-alpine AS builder

# Não instalar ferramentas desnecessárias
RUN apk add --no-cache curl

# Copiar apenas o necessário
COPY package*.json ./
RUN npm ci --only=production

# Multi-stage — imagem final mínima
FROM node:18-alpine
WORKDIR /app

# Usuário não-root
RUN addgroup -S app && adduser -S app -G app
COPY --from=builder --chown=app:app /app/node_modules ./node_modules
COPY --chown=app:app . .

# Filesystem read-only quando possível
# docker run --read-only IMAGE

USER app
EXPOSE 3000
CMD ["node", "server.js"]
```

#### .dockerignore
```
# .dockerignore obrigatório
.git
.env
.env.*
node_modules
*.log
*.md
Dockerfile*
docker-compose*
.github
coverage
__tests__
```

#### daemon.json — hardening
```json
{
  "icc": false,
  "no-new-privileges": true,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "userns-remap": "default",
  "live-restore": true
}
```

---

## Kubernetes

### Diagnóstico Completo
```bash
# Verificar versão e contexto
kubectl version
kubectl config current-context

# Listar todos os recursos
kubectl get all -A

# Verificar pods rodando como root ou privilegiados
kubectl get pods -A -o json | jq '.items[] | select(.spec.containers[].securityContext.runAsRoot == true or .spec.containers[].securityContext.privileged == true) | .metadata.name'

# Verificar secrets expostos em env vars
kubectl get pods -A -o json | jq '.items[].spec.containers[].env'

# Verificar network policies
kubectl get networkpolicies -A

# Verificar RBAC
kubectl get clusterrolebindings -o json | jq '.items[] | select(.subjects[]?.name == "default") | .metadata.name'

# Verificar service accounts com permissões excessivas
kubectl auth can-i --list --as=system:serviceaccount:default:default
```

---

### 🔴 CRÍTICO — Kubernetes

#### Pod sem SecurityContext
```yaml
# ❌ Sem segurança
spec:
  containers:
  - name: app
    image: myapp:1.0

# ✅ Com SecurityContext
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

#### RBAC — ServiceAccount com permissões excessivas
```yaml
# ❌ Cluster-admin para serviceaccount
kind: ClusterRoleBinding
subjects:
- kind: ServiceAccount
  name: default
roleRef:
  kind: ClusterRole
  name: cluster-admin

# ✅ Permissões mínimas necessárias
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: app-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

---

### 🟠 ALTO — Kubernetes

#### Network Policies ausentes
```yaml
# Bloquear todo tráfego por padrão
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# Permitir apenas o necessário
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-db
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432
```

#### Secrets não criptografados em etcd
```bash
# Verificar se encryption at rest está habilitado
kubectl get secret meu-secret -o yaml  # não deve mostrar valor em base64 puro

# Verificar encryption config
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep encryption
```

---

### Ferramentas de Auditoria K8s

```bash
# kube-bench — CIS Benchmark
docker run --rm --pid=host -v /etc:/etc:ro -v /var:/var:ro aquasec/kube-bench

# kube-hunter — pentest de cluster
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-hunter/main/job.yaml

# Trivy — scan de imagens e configs
trivy k8s --report summary cluster

# Polaris — validação de boas práticas
kubectl apply -f https://github.com/FairwindsOps/polaris/releases/latest/download/dashboard.yaml
```

---

## Score por Categoria

| Categoria              | Peso | Verificações                              |
|------------------------|------|-------------------------------------------|
| Containers privilegiados| 25% | --privileged, capabilities, socket        |
| Imagens/CVEs           | 20%  | Trivy scan, tags fixas, base mínima       |
| Usuário root           | 15%  | Dockerfile USER, runAsNonRoot             |
| Secrets                | 15%  | Env vars, Docker secrets, K8s secrets     |
| Network                | 15%  | Isolamento, NetworkPolicies               |
| RBAC                   | 10%  | Least privilege, default serviceaccount   |
