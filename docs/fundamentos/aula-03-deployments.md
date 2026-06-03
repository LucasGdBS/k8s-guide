---
title: "Aula 03 — Deployments"
parent: Fundamentos
nav_order: 4
---

# Aula 03 — Deployments

## Por que não usar Pod puro?

Pod sozinho é "fire and forget" — morreu, acabou. Ninguém sobe um novo automaticamente.

O **Deployment** resolve isso. Ele é um objeto que descreve o _estado desejado_ da sua aplicação, e o K8s trabalha continuamente para manter esse estado.

---

## A hierarquia

```yml
Deployment
    └── ReplicaSet
            └── Pod
            └── Pod
            └── Pod
```

- **Deployment** — você declara o que quer (imagem, réplicas, estratégia de update)
- **ReplicaSet** — gerado pelo Deployment, garante que N Pods estejam rodando
- **Pod** — onde o container realmente executa

O Controller Manager fica olhando constantemente: _"o Deployment pediu 3 Pods, tenho 3 rodando?"_ — se um morrer, ele sobe outro automaticamente.

Os nomes gerados seguem o padrão: `<deployment>-<hash-replicaset>-<hash-pod>`

---

## Criando um Deployment

```yaml
# Deployment garante que N réplicas do Pod estejam sempre rodando
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
  namespace: laboratorio
  labels:
    app: web
    aula: "03-deployments"
spec:
  replicas: 3 # quantos Pods queremos
  selector:
    matchLabels:
      app: web # o Deployment gerencia Pods com esse label
  template: # template do Pod que será criado
    metadata:
      labels:
        app: web # deve bater com o selector acima
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f deployment.yaml
```

---

## Verificação

```bash
kubectl get deployments -n laboratorio
kubectl get replicasets -n laboratorio
kubectl get pods -n laboratorio
```

---

## Resiliência — testando a recuperação automática

```bash
# Deleta um Pod na mão
kubectl delete pod <nome-do-pod> -n laboratorio

# Observa o novo Pod sendo criado automaticamente
kubectl get pods -n laboratorio
```

O novo Pod terá um hash diferente no nome — é um Pod novo, não o mesmo ressuscitado.

---

## Escalando

```bash
# Aumenta para 5 réplicas
kubectl scale deployment web-deployment --replicas=5 -n laboratorio

# Reduz para 2
kubectl scale deployment web-deployment --replicas=2 -n laboratorio

kubectl get pods -n laboratorio
```

---

## Rolling Update — atualização sem downtime

O K8s sobe Pods novos e derruba os antigos **gradualmente** — nunca deixa tudo offline ao mesmo tempo.

```bash
# Troca a imagem do container
kubectl set image deployment/web-deployment nginx=nginx:latest -n laboratorio

# Acompanha o progresso
kubectl rollout status deployment/web-deployment -n laboratorio
```

Durante o rolling update, um novo **ReplicaSet** é criado. O antigo fica com 0 pods, mas é mantido para permitir rollback.

---

## Rollback

```bash
# Ver histórico de versões
kubectl rollout history deployment/web-deployment -n laboratorio

# Voltar para a versão anterior
kubectl rollout undo deployment/web-deployment -n laboratorio
```

O rollback **não apaga** o histórico — ele cria uma nova revisão apontando pro ReplicaSet antigo.

### Anotando o motivo do deploy (boa prática)

```bash
kubectl annotate deployment/web-deployment \
  kubernetes.io/change-cause="atualiza nginx para latest" \
  -n laboratorio

# Para sobrescrever uma anotação existente:
kubectl annotate deployment/web-deployment \
  kubernetes.io/change-cause="atualiza nginx para latest" \
  -n laboratorio --overwrite
```

Isso popula a coluna `CHANGE-CAUSE` no `rollout history`, tornando o histórico legível.

---

## Atenção: `kubectl apply` vs comandos imperativos

Quando você cria um Deployment com `kubectl apply -f arquivo.yaml` e depois usa `kubectl set image` ou `kubectl rollout undo`, o arquivo YAML local fica **desatualizado**.

Se rodar `kubectl apply -f deployment.yaml` de novo, ele vai sobrescrever as mudanças feitas via comando.

**Boas práticas:**

- Em produção, o arquivo YAML é sempre a fonte da verdade
- Alterações devem ser feitas no YAML e reaplicadas com `kubectl apply`
- Comandos imperativos (`scale`, `set image`) são úteis para testes rápidos

---

## Comandos de referência

```bash
# Criar / atualizar
kubectl apply -f deployment.yaml

# Inspecionar
kubectl get deployments -n laboratorio
kubectl get replicasets -n laboratorio
kubectl describe deployment web-deployment -n laboratorio

# Escalar
kubectl scale deployment web-deployment --replicas=N -n laboratorio

# Atualizar imagem
kubectl set image deployment/web-deployment <container>=<imagem> -n laboratorio

# Rollout
kubectl rollout status deployment/web-deployment -n laboratorio
kubectl rollout history deployment/web-deployment -n laboratorio
kubectl rollout undo deployment/web-deployment -n laboratorio

# Anotar
kubectl annotate deployment/web-deployment kubernetes.io/change-cause="motivo" -n laboratorio --overwrite

# Deletar
kubectl delete deployment web-deployment -n laboratorio
```
