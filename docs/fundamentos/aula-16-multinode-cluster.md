---
title: "Aula 16 — Multi-node Cluster com Kind (opcional)"
parent: Fundamentos
nav_order: 17
---

# Aula 16 — Multi-node Cluster com Kind

> **Tópico avançado e opcional.** Relevante quando você começar a trabalhar com clusters gerenciados (EKS, GKE, AKS) ou precisar testar comportamentos de scheduling, afinidade de pods e alta disponibilidade.

---

## Por que múltiplos nós?

No Kind padrão, tudo roda em um único container — control plane e worker juntos. Isso é suficiente para aprender a maioria dos conceitos, mas esconde comportamentos importantes que só aparecem com múltiplos workers:

- **Scheduling real:** o Kubernetes decide em qual nó cada pod vai parar com base em recursos, afinidade e taints
- **DaemonSets:** você vê um pod por nó de verdade
- **Falha de nó:** dá pra simular com `docker stop` e observar o comportamento do cluster
- **Afinidade e anti-afinidade:** forçar pods a ficarem juntos ou separados entre nós
- **PodDisruptionBudget e topologySpreadConstraints:** só fazem sentido com múltiplos nós

---

## Criando um cluster multi-node

```yaml
# kind-multinode.yml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
```

```bash
kind create cluster --name multinode --config kind-multinode.yml
kubectl get nodes
```

Cada nó é um container Docker separado:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```

---

## O que explorar

### Ver o scheduling em ação

```bash
kubectl create deployment nginx --image=nginx --replicas=6
kubectl get pods -o wide  # coluna NODE mostra onde cada pod foi parado
```

### Simular falha de nó

```bash
docker stop multinode-worker2  # derruba um worker

kubectl get nodes               # nó vai para NotReady
kubectl get pods -o wide        # pods do nó morto são reagendados nos outros
```

### DaemonSet em múltiplos nós

```bash
# Um pod por nó — dá pra ver os 3 workers com um pod cada
kubectl get pods -o wide -l app=meu-daemonset
```

---

## Quando você vai usar isso de verdade

Em clusters gerenciados (EKS, GKE, AKS) você sempre tem múltiplos workers — o multi-node no Kind é só pra simular esse ambiente localmente. Os conceitos de afinidade, taints/tolerations e topology spread são os próximos passos naturais quando chegar nesses ambientes.
