---
title: "Aula 01 — Arquitetura e Cluster Kind"
parent: Fundamentos
nav_order: 2
---

# Aula 01 — Arquitetura do Kubernetes e Primeiro Cluster

## O que é Kubernetes e por que ele existe?

Imagina que você tem uma aplicação rodando em um único servidor. Funciona, mas:

- E se o servidor cair?
- E se a aplicação precisar de mais capacidade de repente?
- E se você tiver 20 serviços diferentes para gerenciar?

Kubernetes (K8s) resolve isso. Ele é um **orquestrador de containers** — um sistema que decide _onde_ e _como_ seus containers rodam, e garante que eles continuem rodando mesmo quando algo dá errado.

---

## Arquitetura: Control Plane vs Worker Nodes

Pensa no K8s como uma empresa:

```yml
┌─────────────────────────────────────────────┐
│              CONTROL PLANE (Gerência)       │
│                                             │
│  API Server  ←── você fala com ele via      │
│                  kubectl                    │
│  etcd        ←── banco de dados do cluster  │
│  Scheduler   ←── decide em qual node roda   │
│  Controller  ←── garante o estado desejado  │
└──────────────────┬──────────────────────────┘
                   │ comanda
       ┌───────────┴───────────┐
       ▼                       ▼
┌─────────────┐         ┌─────────────┐
│ Worker Node │         │ Worker Node │
│             │         │             │
│  kubelet    │         │  kubelet    │
│  [Pod]      │         │  [Pod][Pod] │
└─────────────┘         └─────────────┘
```

| Componente             | Função                                            |
| ---------------------- | ------------------------------------------------- |
| **API Server**         | Porta de entrada de tudo — `kubectl` fala com ele |
| **etcd**               | Banco de dados do cluster — guarda todo o estado  |
| **Scheduler**          | Escolhe em qual node cada Pod vai rodar           |
| **Controller Manager** | Garante que a realidade bate com o que você pediu |
| **kubelet**            | Agente em cada node que executa e monitora Pods   |

---

## A pasta `~/.kube`

É onde o `kubectl` guarda tudo que ele precisa para saber com qual cluster falar e como se autenticar. O arquivo principal é `~/.kube/config` — chamado de **kubeconfig**.

```yaml
clusters: # lista de clusters que você conhece
  - name: kind-lab
    cluster:
      server: https://127.0.0.1:PORT # onde está o API Server

users: # credenciais para cada cluster
  - name: kind-lab
    user:
      client-certificate-data: ... # certificado de autenticação

contexts: # combina "cluster + usuário + namespace"
  - name: kind-lab
    context:
      cluster: kind-lab
      user: kind-lab
```

**Contexto** é o conceito-chave: ele diz ao kubectl _"quando eu rodar um comando, fale com esse cluster, usando esse usuário"_. Você pode ter vários clusters no mesmo arquivo e alternar entre eles com `kubectl config use-context`.

---

## Tipos de recursos (Kind no YAML)

O campo `kind` nos manifests YAML identifica o tipo de recurso. Os principais:

**Workloads**

| Kind          | Para que serve                                     |
| ------------- | -------------------------------------------------- |
| `Pod`         | Unidade mínima — container(s) rodando              |
| `Deployment`  | Gerencia Pods com réplicas, updates e rollback     |
| `ReplicaSet`  | Garante N cópias de um Pod                         |
| `StatefulSet` | Pods com identidade estável (bancos de dados, etc) |
| `DaemonSet`   | Roda um Pod em cada node do cluster                |
| `Job`         | Executa uma tarefa até completar                   |
| `CronJob`     | Job agendado                                       |

**Rede**

| Kind            | Para que serve                       |
| --------------- | ------------------------------------ |
| `Service`       | Expõe Pods dentro ou fora do cluster |
| `Ingress`       | Roteamento HTTP/HTTPS externo        |
| `NetworkPolicy` | Regras de firewall entre Pods        |

**Configuração**

| Kind        | Para que serve                         |
| ----------- | -------------------------------------- |
| `ConfigMap` | Configurações em texto (não sensíveis) |
| `Secret`    | Dados sensíveis (senhas, tokens)       |

**Armazenamento**

| Kind                    | Para que serve                        |
| ----------------------- | ------------------------------------- |
| `PersistentVolume`      | Disco disponível no cluster           |
| `PersistentVolumeClaim` | Solicitação de disco por um Pod       |
| `StorageClass`          | Define como volumes são provisionados |

**Controle de acesso**

| Kind                                 | Para que serve                         |
| ------------------------------------ | -------------------------------------- |
| `ServiceAccount`                     | Identidade de um Pod dentro do cluster |
| `Role` / `ClusterRole`               | Define permissões                      |
| `RoleBinding` / `ClusterRoleBinding` | Liga permissões a usuários/contas      |

**Cluster**

| Kind            | Para que serve                       |
| --------------- | ------------------------------------ |
| `Namespace`     | Isolamento lógico de recursos        |
| `Node`          | Representa uma máquina no cluster    |
| `ResourceQuota` | Cota total de recursos por namespace |

---

## Criando o cluster Kind

O **Kind** (Kubernetes in Docker) simula o cluster inteiro dentro de containers Docker na sua máquina.

```bash
# Criar o cluster
kind create cluster --name lab

# Verificar contexto
kubectl config get-contexts
kubectl config use-context kind-lab
```

### Verificação

```bash
# Ver os nodes do cluster
kubectl get nodes

# Ver os componentes do control plane
kubectl get pods -n kube-system
```

Saída esperada do `kubectl get pods -n kube-system`:

```
NAME                                        READY   STATUS    RESTARTS   AGE
coredns-*                                   1/1     Running   0          2m
etcd-lab-control-plane                      1/1     Running   0          2m
kube-apiserver-lab-control-plane            1/1     Running   0          2m
kube-controller-manager-lab-control-plane   1/1     Running   0          2m
kube-scheduler-lab-control-plane            1/1     Running   0          2m
kindnet-*                                   1/1     Running   0          2m
kube-proxy-*                                1/1     Running   0          2m
```

---

## Namespaces

Namespaces são isolamentos lógicos dentro do cluster. Os namespaces padrão:

| Namespace         | Para que serve                                            |
| ----------------- | --------------------------------------------------------- |
| `default`         | Onde recursos vão parar se você não especificar namespace |
| `kube-system`     | Componentes internos do K8s                               |
| `kube-public`     | Informações públicas do cluster                           |
| `kube-node-lease` | Heartbeats dos nodes pro control plane                    |

Crie o namespace para os labs:

```bash
kubectl create namespace laboratorio
kubectl get namespaces
```

---

## Comandos de referência

```bash
# Cluster Kind
kind create cluster --name lab
kind get clusters
kind delete cluster --name lab

# Contexto
kubectl config get-contexts
kubectl config use-context kind-lab

# Inspeção
kubectl get nodes
kubectl get pods -n kube-system
kubectl describe node <nome>
kubectl get all -n laboratorio
```
