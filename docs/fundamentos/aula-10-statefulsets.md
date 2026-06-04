---
title: "Aula 10 — StatefulSets"
parent: Fundamentos
nav_order: 11
---

# Aula 10 — StatefulSets

## Por que não usar Deployment para banco de dados?

O Deployment cria pods **intercambiáveis**: qualquer pod pode morrer e ser substituído por outro idêntico. Isso é ótimo para apps stateless como APIs.

Mas um banco de dados com replicação precisa de:
- **Volume próprio** por pod (primary e replicas têm dados distintos)
- **Identidade fixa** (a aplicação precisa saber exatamente quem é o primary)
- **Ordem de inicialização** (primary deve subir antes das replicas)

O Deployment não garante nenhum disso. É aí que entra o **StatefulSet**.

---

## O que o StatefulSet garante

| Característica | Deployment | StatefulSet |
|---|---|---|
| Pods intercambiáveis | ✅ | ❌ |
| Nome fixo e previsível | ❌ | ✅ (`pod-0`, `pod-1`, `pod-2`) |
| Volume próprio por pod | ❌ | ✅ (via `volumeClaimTemplates`) |
| Ordem de criação/remoção | ❌ | ✅ (sempre em sequência) |
| DNS individual por pod | ❌ | ✅ |

---

## Componentes envolvidos

```
StatefulSet
  ├── Headless Service   → expõe DNS individual por pod
  └── volumeClaimTemplates → cria um PVC exclusivo por pod automaticamente
```

### Headless Service

A diferença para um Service normal é o `clusterIP: None` — em vez de um IP único que balanceia, ele expõe um DNS individual para cada pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-sts
  namespace: laboratorio
spec:
  clusterIP: None        # isso é o que o torna "headless"
  selector:
    app: demo-sts
  ports:
    - port: 80
      name: web
```

### StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: demo-sts
  namespace: laboratorio
spec:
  serviceName: demo-sts  # deve bater com o nome do Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: demo-sts
  template:
    metadata:
      labels:
        app: demo-sts
    spec:
      containers:
        - name: app
          image: busybox
          command:
            - sh
            - -c
            - |
              echo "Sou o pod $POD_NAME" > /dados/identidade.txt
              while true; do sleep 3600; done
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name  # K8s injeta o nome do pod aqui
          volumeMounts:
            - name: dados
              mountPath: /dados

  # cria um PVC exclusivo para cada pod automaticamente:
  # pod demo-sts-0 → PVC dados-demo-sts-0
  # pod demo-sts-1 → PVC dados-demo-sts-1
  # pod demo-sts-2 → PVC dados-demo-sts-2
  volumeClaimTemplates:
    - metadata:
        name: dados
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 50Mi
```

---

## Como o StatefulSet se liga ao Service

Duas ligações distintas:

| Ligação | Como funciona |
|---|---|
| Service descobre os pods | `selector` bate com `labels` dos pods |
| DNS individual por pod | `serviceName` aponta pro Headless Service |

O campo `serviceName` faz o K8s construir endereços no formato:
```
<pod>.<serviceName>.<namespace>.svc.cluster.local
demo-sts-0.demo-sts.laboratorio.svc.cluster.local
demo-sts-1.demo-sts.laboratorio.svc.cluster.local
```

---

## Comportamento dos PVCs

- Os PVCs **não são deletados** quando o pod morre ou quando o StatefulSet é deletado
- Quando o pod reinicia, o K8s busca um PVC com o mesmo nome e o reutiliza — o pod herda os dados anteriores
- A herança é por **convenção de nome**, não mágica: `dados-demo-sts-1` é sempre associado ao `demo-sts-1`

---

## Sincronização de dados (ex: PostgreSQL)

O K8s **não sincroniza dados entre pods** — isso é responsabilidade do banco de dados.

O Postgres usa **streaming replication**:
```
pod-0 (primary)
  └── grava os dados
  └── gera WAL (Write Ahead Log)
  └── transmite o WAL continuamente para pod-1 e pod-2

pod-1 / pod-2 (replicas)
  └── recebem o WAL e aplicam no próprio volume
```

O StatefulSet fornece a **infraestrutura estável** (nome e volume fixos) que o banco usa para fazer a própria sincronização.

A imagem do container (ex: `bitnami/postgresql`) traz um script de inicialização que verifica o nome do pod para decidir o papel:
```
nome termina em -0? → inicializa como PRIMARY
caso contrário?     → inicializa como REPLICA e conecta no pod-0
```

---

## Verificação

```bash
# Ver os pods subindo em ordem
kubectl get pods -n laboratorio -w

# Confirmar que cada pod tem seu próprio PVC
kubectl get pvc -n laboratorio

# Ler o arquivo de cada pod para confirmar volumes separados
kubectl exec demo-sts-0 -n laboratorio -- cat /dados/identidade.txt
kubectl exec demo-sts-1 -n laboratorio -- cat /dados/identidade.txt
kubectl exec demo-sts-2 -n laboratorio -- cat /dados/identidade.txt

# Testar DNS individual via Headless Service
kubectl run dns-test --image=busybox --rm -it --restart=Never -n laboratorio -- \
  nslookup demo-sts-0.demo-sts.laboratorio.svc.cluster.local
```

---

## Quando usar StatefulSet vs Deployment

| Situação | Usar |
|---|---|
| API REST, frontend, worker stateless | Deployment |
| Banco de dados com replicação | StatefulSet |
| Cache com persistência (Redis) | StatefulSet |
| Fila de mensagens (Kafka, RabbitMQ) | StatefulSet |

---

## Comandos de referência

```bash
# Listar StatefulSets
kubectl get statefulsets -n laboratorio

# Ver detalhes
kubectl describe statefulset demo-sts -n laboratorio

# Escalar
kubectl scale statefulset demo-sts --replicas=5 -n laboratorio

# Deletar StatefulSet (PVCs permanecem)
kubectl delete statefulset demo-sts -n laboratorio

# Deletar PVCs manualmente se necessário
kubectl delete pvc dados-demo-sts-0 -n laboratorio
```
