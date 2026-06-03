---
title: "Aula 06 — ConfigMaps e Secrets"
parent: Fundamentos
nav_order: 7
---

# Aula 06 — ConfigMaps e Secrets

## Por que separar configuração da aplicação?

A mesma imagem Docker deve rodar em dev, staging e produção — só mudando a configuração. Embutir configurações na imagem ou no YAML do Deployment significa rebuildar ou editar manifests a cada mudança.

O K8s resolve isso com dois recursos:

| Recurso | Para que serve | Exemplo |
|---|---|---|
| **ConfigMap** | Configurações não sensíveis | URL do banco, nível de log, nome do ambiente |
| **Secret** | Dados sensíveis | Senhas, tokens, chaves de API |

---

## ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: laboratorio
data:
  APP_ENV: "producao"
  APP_LOG_LEVEL: "info"
  APP_PORT: "8080"
```

```bash
kubectl apply -f configmap.yaml
kubectl describe configmap app-config -n laboratorio
```

O `describe` mostra os valores em texto puro — por isso ConfigMap não serve para dados sensíveis.

---

## Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: laboratorio
type: Opaque
stringData:            # stringData aceita texto puro — o K8s converte para base64
  DB_PASSWORD: "minha-senha-secreta"
  API_KEY: "abc123xyz"
```

```bash
kubectl apply -f secret.yaml
kubectl describe secret app-secret -n laboratorio
```

O `describe` **não mostra os valores** — só o tamanho em bytes. Para ver o valor (evite em produção):

```bash
kubectl get secret app-secret -n laboratorio -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

---

## Injetando no Pod

O Pod não descobre os recursos automaticamente — você declara explicitamente qual ConfigMap ou Secret usar, pelo nome.

### Como variáveis de ambiente

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-config
  namespace: laboratorio
spec:
  containers:
    - name: app
      image: alpine
      command: ["sleep", "3600"]
      envFrom:
        - configMapRef:
            name: app-config   # injeta todas as chaves do ConfigMap como env vars
        - secretRef:
            name: app-secret   # injeta todas as chaves do Secret como env vars
```

```bash
kubectl exec -it pod-config -n laboratorio -- env | grep -E "APP|DB|API"
```

### Como arquivos montados

```yaml
spec:
  containers:
    - name: app
      image: alpine
      command: ["sleep", "3600"]
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config   # cada chave vira um arquivo nessa pasta
  volumes:
    - name: config-volume
      configMap:
        name: app-config
```

```bash
kubectl exec -it pod-config -n laboratorio -- ls /etc/config
kubectl exec -it pod-config -n laboratorio -- cat /etc/config/APP_ENV
```

---

## Comparação

| | ConfigMap | Secret |
|---|---|---|
| Valores visíveis no `describe` | Sim | Não (só tamanho) |
| Armazenamento no etcd | Texto puro | Base64 |
| Uso | Configurações gerais | Senhas, tokens, chaves |
| Como injetar | `configMapRef` | `secretRef` |

> **Atenção:** base64 não é criptografia — é só encoding. A segurança real do Secret vem do RBAC (controle de quem pode ler o namespace) e da criptografia em repouso no etcd.

---

## Níveis de maturidade para Secrets em produção

| Nível | Abordagem | Quando usar |
|---|---|---|
| 1 | Secret nativo do K8s | Dev, labs, times pequenos com RBAC |
| 2 | Secret nativo + criptografia em repouso no etcd | Produção com requisitos moderados |
| 3 | External Secrets Operator (AWS Secrets Manager, GCP Secret Manager, Vault) | Produção com auditoria e compliance |
| 4 | Vault Agent Sidecar — o valor nunca vira Secret K8s | Máxima segurança |

O padrão mais comum em produção séria é o **External Secrets Operator**: os valores ficam em um cofre externo (AWS/GCP/Vault), e um operador sincroniza automaticamente com o cluster.

---

## Comandos de referência

```bash
# ConfigMap
kubectl apply -f configmap.yaml
kubectl get configmaps -n laboratorio
kubectl describe configmap <nome> -n laboratorio

# Secret
kubectl apply -f secret.yaml
kubectl get secrets -n laboratorio
kubectl describe secret <nome> -n laboratorio
kubectl get secret <nome> -n laboratorio -o jsonpath='{.data.<chave>}' | base64 -d

# Verificar injeção no Pod
kubectl exec -it <pod> -n laboratorio -- env
kubectl exec -it <pod> -n laboratorio -- ls /etc/config
```
