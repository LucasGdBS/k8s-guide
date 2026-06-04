---
title: "Aula 09 — RBAC"
parent: Fundamentos
nav_order: 10
---

# Aula 09 — RBAC

## Por que o RBAC existe?

No Kubernetes, qualquer processo rodando dentro de um pod pode tentar fazer requisições à API do cluster — listar pods, criar deployments, ler secrets. Por padrão, essas requisições são negadas.

O **RBAC (Role-Based Access Control)** define quem pode fazer o quê no cluster. "Quem" pode ser um usuário humano, uma ferramenta de CI/CD, ou um pod rodando uma aplicação como o Prometheus.

---

## Os três recursos do RBAC

```
ServiceAccount   →   quem é (identidade)
Role/ClusterRole →   o que pode fazer (permissões)
RoleBinding/     →   a ligação entre os dois
ClusterRoleBinding
```

### ServiceAccount

É a identidade de um pod dentro do cluster. Todo pod tem uma — se você não especificar, o K8s usa a `default` do namespace, que não tem permissões.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: minha-sa
  namespace: laboratorio
```

### Role vs ClusterRole

| Recurso | Escopo |
|---|---|
| `Role` | Apenas um namespace |
| `ClusterRole` | O cluster inteiro |

Use `ClusterRole` quando a aplicação precisa enxergar recursos em múltiplos namespaces (como o Prometheus descobrindo pods em todo o cluster). Use `Role` para permissões restritas a um namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: leitor-de-pods
rules:
  - apiGroups: [""]          # "" = core API (pods, services, secrets...)
    resources: ["pods", "services", "endpoints"]
    verbs: ["get", "list", "watch"]
```

**apiGroups:** as APIs do K8s são organizadas em grupos. O grupo `""` (vazio) é o core e contém pods, services, secrets, configmaps. Outros grupos como `apps` contém deployments e statefulsets.

**verbs disponíveis:** `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`

### RoleBinding e ClusterRoleBinding

Liga o ServiceAccount ao Role/ClusterRole:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: leitor-de-pods-binding
subjects:
  - kind: ServiceAccount
    name: minha-sa          # o ServiceAccount que receberá as permissões
    namespace: laboratorio
roleRef:
  kind: ClusterRole
  name: leitor-de-pods      # o ClusterRole a ser aplicado
  apiGroup: rbac.authorization.k8s.io
```

---

## O fluxo completo

```
Pod do Prometheus
  └── usa → ServiceAccount "prometheus-sa"
               └── ligado por → ClusterRoleBinding
                                   └── ao → ClusterRole "prometheus-role"
                                               └── pode: get/list/watch
                                                         pods, services, endpoints
```

---

## Referenciando o ServiceAccount no Deployment

Criar o ServiceAccount não é suficiente — o pod precisa declarar que o usa:

```yaml
spec:
  serviceAccountName: minha-sa   # nível do pod, fora do bloco containers
  containers:
    - name: app
      ...
```

Sem isso, o pod usa a ServiceAccount `default` do namespace e não terá as permissões definidas no ClusterRole.

---

## Exemplo completo: Prometheus com acesso à API do K8s

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-sa
  namespace: project

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-role
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "endpoints", "nodes"]
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-rolebinding
subjects:
  - kind: ServiceAccount
    name: prometheus-sa
    namespace: project
roleRef:
  kind: ClusterRole
  name: prometheus-role
  apiGroup: rbac.authorization.k8s.io
```

---

## Verificando permissões

O kubectl tem um comando muito útil para testar se uma identidade pode fazer determinada ação:

```bash
# Pode o ServiceAccount prometheus-sa listar pods?
kubectl auth can-i list pods \
  --as=system:serviceaccount:project:prometheus-sa \
  -n project

# Resultado: yes ou no

# Verificar todas as permissões de um ServiceAccount
kubectl auth can-i --list \
  --as=system:serviceaccount:project:prometheus-sa \
  -n project
```

---

## Role vs ClusterRole — quando usar cada um

| Situação | Usar |
|---|---|
| App que lê seus próprios ConfigMaps | `Role` no mesmo namespace |
| Prometheus descobrindo pods no cluster | `ClusterRole` |
| CI/CD criando Deployments num namespace | `Role` |
| Ferramenta de auditoria lendo tudo | `ClusterRole` com cuidado |

**Princípio do menor privilégio:** conceda apenas as permissões necessárias, no escopo mais restrito possível.

---

## Comandos de referência

```bash
# Listar ServiceAccounts
kubectl get serviceaccounts -n laboratorio

# Listar ClusterRoles
kubectl get clusterroles

# Listar ClusterRoleBindings
kubectl get clusterrolebindings

# Ver detalhes de um ClusterRole
kubectl describe clusterrole <nome>

# Testar permissão de um ServiceAccount
kubectl auth can-i <verb> <resource> \
  --as=system:serviceaccount:<namespace>:<nome-da-sa> \
  -n <namespace>
```
