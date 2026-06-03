---
title: "Aula 05 — Ingress"
parent: Fundamentos
nav_order: 6
---

# Aula 05 — Ingress e Ingress Controller

## Por que Ingress?

Com NodePort você precisa de uma porta diferente para cada serviço (`30080`, `30081`, `30082`...). Ninguém quer expor uma aplicação assim.

O **Ingress** é um roteador HTTP inteligente que recebe tudo na porta 80/443 e direciona para o Service certo baseado na URL:

```
          ┌─────────────────┐
          │  Ingress        │
porta 80 →│                 │─ /api   → Service api
          │  (roteamento    │─ /admin → Service admin
          │   por URL)      │─ /      → Service frontend
          └─────────────────┘
```

---

## Ingress Controller

O Ingress sozinho é só uma regra — ele não faz nada sem um **Ingress Controller**, que é o Pod responsável por ler essas regras e aplicá-las.

O mais comum é o **nginx ingress controller**.

---

## Configurando o Kind para suportar Ingress

O Kind precisa de configuração especial para mapear as portas 80/443 do container pro host. Crie o cluster com este `kind-config.yaml`:

```yaml
# Configuração do cluster Kind com suporte a Ingress
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
```

```bash
kind create cluster --name lab --config kind-config.yaml
```

---

## Instalando o nginx ingress controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Aguarda o controller ficar pronto
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

---

## Exemplo completo — dois apps com roteamento por path

```yaml
# App 1 — nginx
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-nginx
  namespace: laboratorio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-nginx
  template:
    metadata:
      labels:
        app: app-nginx
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
---
apiVersion: v1
kind: Service
metadata:
  name: svc-nginx
  namespace: laboratorio
spec:
  selector:
    app: app-nginx
  ports:
    - port: 80
      targetPort: 80
---
# App 2 — httpd
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-httpd
  namespace: laboratorio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-httpd
  template:
    metadata:
      labels:
        app: app-httpd
    spec:
      containers:
        - name: httpd
          image: httpd:alpine
---
apiVersion: v1
kind: Service
metadata:
  name: svc-httpd
  namespace: laboratorio
spec:
  selector:
    app: app-httpd
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lab-ingress
  namespace: laboratorio
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /nginx(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: svc-nginx
                port:
                  number: 80
          - path: /httpd(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: svc-httpd
                port:
                  number: 80
```

```bash
kubectl apply -f ingress-lab.yaml
curl http://localhost/nginx
curl http://localhost/httpd
```

---

## Como o rewrite-target funciona

O path `/nginx(/|$)(.*)` usa capture groups (regex):

```
/nginx  (/|$)  (.*)
         $1     $2
```

- `$1` — a barra após `/nginx` ou fim da URL
- `$2` — tudo que vem depois (pode ser vazio, ou `foo`, ou `index.html`)

Com `rewrite-target: /$2`, o path é reescrito antes de chegar no backend:

```
localhost/nginx       →  svc-nginx:/       ($2 vazio)
localhost/nginx/foo   →  svc-nginx:/foo    ($2 = "foo")
localhost/httpd       →  svc-httpd:/       ($2 vazio)
```

> **Importante:** use sempre `pathType: ImplementationSpecific` quando o path contiver regex. O `pathType: Prefix` não aceita expressões regulares.

---

## Verificação

```bash
kubectl get ingress -n laboratorio
kubectl describe ingress lab-ingress -n laboratorio
```

---

## Comandos de referência

```bash
# Instalar ingress controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Aguardar controller
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s

# Inspecionar
kubectl get ingress -n laboratorio
kubectl describe ingress <nome> -n laboratorio

# Testar
curl http://localhost/<path>
```
