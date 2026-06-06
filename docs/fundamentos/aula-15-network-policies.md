---
title: "Aula 15 — NetworkPolicies"
parent: Fundamentos
nav_order: 16
---

# Aula 15 — NetworkPolicies

## Por que NetworkPolicies existem?

Por padrão, todo pod no Kubernetes consegue falar com qualquer outro pod do cluster — independente de namespace. Isso é conveniente no começo, mas perigoso em produção: se um pod for comprometido, ele pode alcançar o banco de dados, serviços internos e tudo mais.

NetworkPolicy é o mecanismo do Kubernetes para definir **quem pode falar com quem** na camada de rede. Funciona como um firewall declarativo — você descreve as regras em YAML e o CNI as aplica nos nós.

> Analogia: pense em NetworkPolicy como portarias em um prédio corporativo. Sem portaria, qualquer pessoa entra em qualquer sala. Com portaria, você define: "o almoxarifado só aceita entrega do fornecedor X, pela porta dos fundos".

**Importante:** NetworkPolicy só funciona se o CNI instalado no cluster tiver suporte a ela. O CNI padrão do Kind (`kindnet`) **não suporta**. Para este lab usamos o **Calico**.

---

## Componentes envolvidos

| Componente | Função |
|---|---|
| **NetworkPolicy** | Recurso Kubernetes que declara as regras de rede |
| **CNI (Calico)** | Implementa as regras no kernel de cada nó via eBPF/iptables |
| **podSelector** | Define quais pods desta política se aplica |
| **namespaceSelector** | Filtra tráfego por namespace de origem/destino |
| **ingress** | Regras de tráfego de **entrada** no pod |
| **egress** | Regras de tráfego de **saída** do pod |

---

## Cluster com Calico

O Kind precisa ser criado com o CNI padrão desabilitado:

```yaml
# manifests/clusters/kind-calico.yml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true     # desabilita kindnet
  podSubnet: "192.168.0.0/16" # Calico usa esta subnet por padrão
nodes:
  - role: control-plane
  - role: worker
```

```bash
kind create cluster --name lab --config manifests/clusters/kind-calico.yml
```

Os nós ficam `NotReady` até o Calico ser instalado — comportamento esperado.

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/custom-resources.yaml

# Aguarda tudo subir (~2-3 min)
watch kubectl get pods -n calico-system
```

---

## Estrutura de uma NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: exemplo
  namespace: meu-namespace
spec:
  podSelector:        # a quais pods esta política se aplica
    matchLabels:
      app: meu-app
  policyTypes:
    - Ingress         # controla entrada
    - Egress          # controla saída
  ingress:
    - from:           # de onde pode vir o tráfego
        - namespaceSelector:
            matchLabels:
              role: frontend
          podSelector:
            matchLabels:
              app: cliente
      ports:
        - protocol: TCP
          port: 80
```

### Comportamento padrão

- Pod **sem nenhuma** NetworkPolicy: aceita tudo (ingress e egress)
- Pod **com** NetworkPolicy: só o que as regras explicitamente permitem passa

---

## Lab: isolamento frontend → backend → database

### Cenário

Três namespaces com um pod cada, mais um pod `intruso` no namespace `frontend`:

```
frontend (ns) ──→ backend (ns) ──→ database (ns)
intruso  (ns: frontend)
```

```bash
kubectl apply -f manifests/network-policies/01-scenario.yml
```

### Passo 1 — Sem política: tudo passa

Antes de aplicar qualquer regra, o `intruso` consegue acessar o `backend` e o `database` livremente. Isso é o problema que vamos resolver.

### Passo 2 — Deny-all no database

```yaml
# 02-deny-all.yml
spec:
  podSelector: {}   # {} = aplica a TODOS os pods do namespace
  policyTypes:
    - Ingress
  # sem regras de ingress = ninguém entra
```

```bash
kubectl apply -f manifests/network-policies/02-deny-all.yml
```

Agora qualquer curl ao `database` retorna `exit code 28` (timeout). O Calico faz **DROP** silencioso — não rejeita, descarta. Isso é intencional: não revela que o host existe.

> **DROP vs REJECT:** DROP descarta o pacote sem responder (remetente espera até timeout). REJECT responde com erro imediato. DROP é mais seguro; REJECT é mais fácil de debugar.

### Passo 3 — Libera só o backend

```yaml
# 03-allow-backend-to-db.yml
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              role: backend   # namespace de origem
          podSelector:
            matchLabels:
              app: backend    # pod de origem (AND com o namespaceSelector)
      ports:
        - protocol: TCP
          port: 80
```

```bash
kubectl apply -f manifests/network-policies/03-allow-backend-to-db.yml
```

- `backend` → `database`: **passa**
- `intruso` → `database`: **bloqueado** (exit code 28)

### Passo 4 — Libera frontend → backend (bloqueia intruso)

```bash
kubectl apply -f manifests/network-policies/04-allow-frontend-to-backend.yml
```

- `frontend` → `backend`: **passa**
- `intruso` → `backend`: **bloqueado** (mesmo namespace, label diferente)
- `frontend` → `database`: **bloqueado** (deny-all cobre isso)

### Passo 5 — Controle de saída (egress)

```yaml
# 05-egress-example.yml
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
    - Egress
  egress:
    - ports:
        - protocol: UDP
          port: 53   # DNS — sempre libere ou os pods perdem resolução de nomes
        - protocol: TCP
          port: 53
```

O `database` não consegue mais iniciar conexões para fora — só responder as que chegam. Evita exfiltração de dados caso o pod seja comprometido.

---

## A armadilha: AND vs OR nos seletores

Essa é a pegadinha mais comum em NetworkPolicy:

```yaml
# AND lógico — mesmo item da lista
- from:
    - namespaceSelector:
        matchLabels:
          role: frontend
      podSelector:          # ← sem hífen, mesmo objeto
        matchLabels:
          app: frontend
# Resultado: namespace=frontend E pod=frontend (restritivo)
```

```yaml
# OR lógico — itens separados da lista
- from:
    - namespaceSelector:
        matchLabels:
          role: frontend
    - podSelector:          # ← com hífen, objeto separado
        matchLabels:
          app: frontend
# Resultado: namespace=frontend OU pod=frontend em qualquer namespace (permissivo!)
```

O hífen faz toda a diferença. Na dúvida, inspecione com `kubectl describe networkpolicy`.

---

## Comandos úteis

```bash
# Listar todas as NetworkPolicies do cluster
kubectl get networkpolicy -A

# Detalhar uma política (mostra as regras de forma legível)
kubectl describe networkpolicy <nome> -n <namespace>

# Ver se um pod tem política aplicada
kubectl get pod <nome> -n <namespace> -o yaml | grep -A5 networkPolicy

# Testar conectividade entre pods
kubectl exec -n <namespace> <pod> -- curl -s --max-time 3 <IP>
```

---

## Boas práticas

- **Comece sempre com deny-all** e vá abrindo seletivamente
- **Sempre libere o DNS (porta 53 UDP/TCP)** nas políticas de egress — sem isso os pods não resolvem nomes
- **Use labels nos namespaces** — `namespaceSelector` depende deles
- **Documente o motivo** de cada regra — NetworkPolicies são fáceis de empilhar e difíceis de auditar depois
