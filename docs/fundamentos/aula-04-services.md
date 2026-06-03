---
title: "Aula 04 — Services"
parent: Fundamentos
nav_order: 5
---

# Aula 04 — Services

## O problema que o Service resolve

Pods têm IPs — mas esses IPs são **efêmeros**. Toda vez que um Pod morre e sobe de novo, ele ganha um IP diferente. Isso torna impossível apontar diretamente para um Pod.

O **Service** resolve isso: é um endereço estável e permanente que fica na frente dos Pods. Ele usa os **labels** para saber quais Pods pertencem a ele, e distribui o tráfego entre eles (load balancing básico).

```
           ┌─────────────┐
cliente →  │   Service   │ → Pod A (ip: 10.0.0.1)
           │  (IP fixo)  │ → Pod B (ip: 10.0.0.2)
           └─────────────┘ → Pod C (ip: 10.0.0.3)
```

---

## Deployment + Service — o par padrão

Todo microsserviço no K8s tem esses dois recursos andando juntos:

- **Deployment** → gerencia os Pods (garante que estão rodando)
- **Service** → expõe os Pods (dá um endereço estável pra eles)

A única exceção são Jobs e CronJobs — tarefas pontuais que não recebem tráfego de rede.

Você pode colocar os dois no mesmo arquivo YAML separados por `---`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: laboratorio
spec:
  # ...

---

apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: laboratorio
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8080
```

---

## Os três tipos de Service

| Tipo | Acessível de | Uso |
|---|---|---|
| **ClusterIP** | Dentro do cluster apenas | Comunicação interna entre serviços |
| **NodePort** | Fora do cluster via IP do node | Testes e desenvolvimento local |
| **LoadBalancer** | Fora do cluster via IP externo | Produção em cloud |

---

## ClusterIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: laboratorio
  labels:
    app: web
    aula: "04-services"
spec:
  type: ClusterIP          # padrão — só acessível de dentro do cluster
  selector:
    app: web               # seleciona todos os Pods com esse label
  ports:
    - port: 80             # porta do Service
      targetPort: 80       # porta do container
```

```bash
kubectl apply -f service.yaml
kubectl get services -n laboratorio
```

### Testando o ClusterIP

Como o ClusterIP só é acessível de dentro do cluster, use um Pod temporário:

```bash
kubectl run test --image=alpine --rm -it --restart=Never -n laboratorio -- sh

# Dentro do Pod:
wget -qO- web-service
wget -qO- web-service.laboratorio.svc.cluster.local
```

Os dois funcionam. O segundo é o nome completo do DNS interno — o formato é sempre:

```
<service>.<namespace>.svc.cluster.local
```

> **Importante:** você sempre chama pelo nome do **Service**, nunca pelo nome do Deployment ou pelo IP do Pod. O DNS interno resolve o nome para o ClusterIP correto automaticamente.

---

## NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: laboratorio
  labels:
    app: web
    aula: "04-services"
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080      # porta fixa no node (range: 30000-32767)
```

```bash
kubectl apply -f service.yaml
```

O fluxo completo do NodePort:

```
Você (externo)
    │
    │  curl http://<IP-do-node>:30080
    ▼
Node (porta 30080 aberta)
    │
    ▼
Service (ClusterIP interno)
    │
    ├──▶ Pod A
    ├──▶ Pod B
    └──▶ Pod C
```

Para descobrir o IP do node no Kind:

```bash
kubectl get nodes -o wide
# coluna INTERNAL-IP é o IP do node

curl http://<INTERNAL-IP>:30080
```

---

## LoadBalancer

Usado em produção em cloud (AWS, GCP, Azure). O provedor cria automaticamente um load balancer externo com IP fixo.

No Kind, `EXTERNAL-IP` fica em `<pending>` por padrão — o Kind não tem um provedor de cloud real. Para usar LoadBalancer no Kind é necessário instalar o **MetalLB** ou o **cloud-provider-kind** (veremos em aulas avançadas).

---

## DNS interno do cluster

Dentro do cluster, todo Service ganha automaticamente um registro DNS:

```
<nome-do-service>.<namespace>.svc.cluster.local
```

No mesmo namespace, o nome curto resolve:

```
web-service  →  web-service.laboratorio.svc.cluster.local
```

É assim que microsserviços se comunicam — sem hardcode de IP, só pelo nome do Service.

---

## Comandos de referência

```bash
# Criar / atualizar
kubectl apply -f service.yaml

# Listar
kubectl get services -n laboratorio

# Detalhes
kubectl describe service web-service -n laboratorio

# Pod temporário para testar conectividade interna
kubectl run test --image=alpine --rm -it --restart=Never -n laboratorio -- sh

# IP dos nodes
kubectl get nodes -o wide

# Deletar
kubectl delete service web-service -n laboratorio
```
