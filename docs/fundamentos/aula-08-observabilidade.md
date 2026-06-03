---
title: "Aula 08 — Observabilidade"
parent: Fundamentos
nav_order: 9
---

# Aula 08 — Observabilidade

## Os três pilares práticos

| Pilar | Ferramenta |
|---|---|
| **Logs** | `kubectl logs` |
| **Eventos** | `kubectl describe` / `kubectl events` |
| **Saúde dos containers** | Probes (liveness, readiness, startup) |

---

## Logs

```bash
# Logs de um Pod
kubectl logs <nome-do-pod> -n laboratorio

# Acompanhar em tempo real (equivalente ao tail -f)
kubectl logs -f <nome-do-pod> -n laboratorio

# Últimas 20 linhas
kubectl logs --tail=20 <nome-do-pod> -n laboratorio

# Logs de todos os Pods com um label
kubectl logs -l app=web -n laboratorio
```

---

## Probes — saúde dos containers

O K8s precisa saber se seu container está realmente funcionando — não só "rodando", mas pronto para receber tráfego e saudável.

| Probe | Pergunta que responde | Consequência se falhar |
|---|---|---|
| **Startup** | O container terminou de inicializar? | K8s aguarda antes de iniciar liveness/readiness |
| **Liveness** | O container está vivo? | K8s reinicia o container |
| **Readiness** | O container está pronto para receber tráfego? | K8s remove o Pod do Service |

### Resumo prático

```
Startup   → "ainda estou inicializando, não me avalie ainda"
Readiness → "estou pronto? Se não, tira meu tráfego"
Liveness  → "estou vivo? Se não, me reinicia"
```

---

## Configurando as três probes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-probes
  namespace: laboratorio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-probes
  template:
    metadata:
      labels:
        app: web-probes
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80

          startupProbe:
            httpGet:
              path: /
              port: 80
            failureThreshold: 30   # tenta 30 vezes antes de desistir
            periodSeconds: 2       # a cada 2 segundos

          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5  # aguarda 5s antes de começar
            periodSeconds: 10       # verifica a cada 10s
            failureThreshold: 3     # reinicia após 3 falhas consecutivas

          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 5
            failureThreshold: 3     # remove do Service após 3 falhas
```

### Campos das probes

| Campo | Significado |
|---|---|
| `initialDelaySeconds` | Aguarda N segundos antes de começar a verificar |
| `periodSeconds` | Verifica a cada N segundos |
| `timeoutSeconds` | O container tem N segundos para responder |
| `successThreshold` | N respostas OK para considerar saudável |
| `failureThreshold` | N falhas consecutivas para agir |

---

## Tipos de verificação

Além de `httpGet`, as probes suportam:

```yaml
# Executa um comando dentro do container
exec:
  command: ["pg_isready", "-U", "postgres"]

# Testa uma porta TCP
tcpSocket:
  port: 5432

# Endpoint gRPC
grpc:
  port: 50051
```

---

## CrashLoopBackOff

Quando um container fica reiniciando repetidamente, o K8s aplica um **backoff exponencial** antes de cada reinício: 10s, 20s, 40s, 80s... até 5 minutos.

Não é necessariamente um crash — pode ser que o processo saiu com código 0 (`Completed`) ou que a liveness probe falhou. O `CrashLoopBackOff` protege o cluster de loops infinitos de reinício.

Para diagnosticar:

```bash
kubectl describe pod <nome> -n laboratorio   # seção Events mostra o motivo
kubectl logs <nome> -n laboratorio --previous  # logs do container anterior ao reinício
```

---

## O ciclo de um Pod com falha de liveness

```
Running           → processo parado, liveness começa a falhar
Completed         → processo saiu (graceful shutdown, código 0)
CrashLoopBackOff  → K8s aplicando backoff antes de reiniciar
0/1 Running       → container vivo, readiness ainda não passou
1/1 Running       → readiness passou, Pod voltou ao Service
```

O `0/1` durante o `Running` é o **readiness probe protegendo os usuários** — nenhum tráfego chega ao Pod até ele estar pronto.

---

## Comandos de referência

```bash
# Logs
kubectl logs <pod> -n laboratorio
kubectl logs -f <pod> -n laboratorio
kubectl logs --tail=20 <pod> -n laboratorio
kubectl logs --previous <pod> -n laboratorio   # logs do container anterior

# Monitorar em tempo real
kubectl get pods -n laboratorio -w

# Eventos do namespace
kubectl events -n laboratorio

# Verificar probes configuradas
kubectl describe pod <pod> -n laboratorio
```
