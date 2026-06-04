---
title: "Aula 13 — DaemonSets"
parent: Fundamentos
nav_order: 14
---

# Aula 13 — DaemonSets

## Por que existem?

Deployments distribuem N réplicas pelos nós sem garantia de onde cada uma vai parar. Mas algumas ferramentas precisam rodar **em toda máquina do cluster**, sem exceção:

- Coletores de log (Fluentd, Filebeat, Promtail)
- Agentes de monitoramento (Prometheus Node Exporter)
- Plugins de rede (Calico, Flannel)
- Agentes de segurança

Para isso existe o **DaemonSet**: ele garante que **exatamente um Pod rode em cada nó**. Novo nó entrou? Pod sobe automaticamente. Nó saiu? Pod é removido junto.

> O próprio `kube-proxy` que viabiliza a comunicação entre Services é um DaemonSet rodando em todos os nós do cluster.

---

## DaemonSet vs Deployment

| | Deployment | DaemonSet |
|---|---|---|
| **Objetivo** | N réplicas distribuídas | 1 cópia por nó |
| **Escala** | Manual ou HPA | Automático conforme nós |
| **Remoção de nó** | Pod é recriado em outro nó | Pod é descartado |
| **Caso de uso** | Aplicações | Agentes de infraestrutura |

---

## Manifesto

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-logger
  namespace: laboratorio
  labels:
    app: node-logger
spec:
  selector:
    matchLabels:
      app: node-logger
  template:
    metadata:
      labels:
        app: node-logger
    spec:
      tolerations:
        # permite rodar no control-plane, que por padrão rejeita pods comuns
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      containers:
        - name: logger
          image: busybox
          command:
            - sh
            - -c
            - 'while true; do echo "rodando no no $NODE_NAME em: $(date)"; sleep 5; done'
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName  # injeta o nome do nó via Downward API
          resources:
            requests:
              cpu: "10m"
              memory: "16Mi"
            limits:
              cpu: "50m"
              memory: "32Mi"
```

---

## Tolerations e o control-plane

Por padrão, o nó de control-plane tem o taint `node-role.kubernetes.io/control-plane:NoSchedule`, que impede pods comuns de serem agendados nele. O DaemonSet respeita esse taint — sem uma toleration explícita, o pod **não é criado** no control-plane.

```
DESIRED = número de nós elegíveis (sem taint bloqueante)
```

Para rodar em todos os nós incluindo o control-plane, adicione a toleration:

```yaml
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
```

Ferramentas de infra como `kube-proxy` e `node-exporter` sempre incluem essa toleration. Workloads de aplicação normalmente não precisam dela.

---

## Verificação

```bash
# Status geral — DESIRED deve bater com o número de nós elegíveis
kubectl get daemonset node-logger -n laboratorio

# Pods e em qual nó cada um está rodando
kubectl get pods -n laboratorio -l app=node-logger -o wide

# Logs de todos os pods do DaemonSet ao mesmo tempo
kubectl logs -n laboratorio -l app=node-logger
```

---

## Comandos de referência

```bash
# Ver todos os DaemonSets do namespace
kubectl get daemonsets -n laboratorio

# Detalhes e eventos
kubectl describe daemonset node-logger -n laboratorio

# Forçar rollout após alterar o manifesto
kubectl rollout restart daemonset/node-logger -n laboratorio

# Status do rollout
kubectl rollout status daemonset/node-logger -n laboratorio

# Deletar — remove o DaemonSet e todos os seus pods
kubectl delete daemonset node-logger -n laboratorio
```
