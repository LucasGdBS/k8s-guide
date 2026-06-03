---
title: "Aula 02 — Pods"
parent: Fundamentos
nav_order: 3
---

# Aula 02 — Pods

## O que é um Pod?

O Pod é a **menor unidade do Kubernetes**. Não é um container diretamente — é um invólucro que contém um ou mais containers que compartilham rede e armazenamento.

**Analogia:** pensa no Pod como um "apartamento". Os containers são os moradores. Eles compartilham o mesmo endereço IP e podem se falar via `localhost`.

Na prática, 90% dos casos é um Pod com um único container.

---

## Pod vs Container

|                       | Container                    | Pod        |
| --------------------- | ---------------------------- | ---------- |
| Gerenciado por        | Docker/containerd            | Kubernetes |
| IP próprio            | Não (compartilha com o host) | Sim        |
| Unidade de scheduling | Não                          | Sim        |

O K8s nunca agenda containers diretamente — sempre Pods.

---

## Criando um Pod

```yaml
# Um Pod simples rodando nginx — servidor web leve, ótimo pra testes
apiVersion: v1
kind: Pod
metadata:
  name: meu-primeiro-pod
  namespace: laboratorio
  labels:
    app: nginx
    aula: "02-pods"
spec:
  containers:
    - name: nginx
      image: nginx:alpine # versão leve do nginx
      ports:
        - containerPort: 80 # porta que o container expõe
```

```bash
kubectl apply -f pod.yaml
```

---

## Verificação

```bash
# Status do pod
kubectl get pods -n laboratorio

# Detalhes completos (metadados, eventos, condições, recursos)
kubectl describe pod meu-primeiro-pod -n laboratorio

# Logs do container
kubectl logs meu-primeiro-pod -n laboratorio

# Entrar no terminal do container
kubectl exec -it meu-primeiro-pod -n laboratorio -- sh

# Dentro do container, testar o nginx:
wget -qO- localhost
```

Saída esperada do `kubectl get pods`:

```
NAME               READY   STATUS    RESTARTS   AGE
meu-primeiro-pod   1/1     Running   0          1m
```

| Campo            | Significado                       |
| ---------------- | --------------------------------- |
| `READY 1/1`      | 1 container rodando de 1 esperado |
| `STATUS Running` | Pod saudável                      |
| `RESTARTS 0`     | Nunca reiniciou                   |

---

## Limitação do Pod puro

**Pod sozinho não se recupera.** Se ele morrer, acabou — ninguém sobe um novo automaticamente.

Por isso, na prática você **nunca cria Pods diretamente em produção**. Você usa um **Deployment**, que gerencia os Pods por você.

---

## Deletando Pods

```bash
kubectl delete pod meu-primeiro-pod -n laboratorio

# Deletar múltiplos de uma vez
kubectl delete pod pod1 pod2 pod3 -n laboratorio
```

---

## Comandos de referência

```bash
kubectl get pods -n laboratorio
kubectl describe pod <nome> -n laboratorio
kubectl logs <nome> -n laboratorio
kubectl exec -it <nome> -n laboratorio -- sh
kubectl delete pod <nome> -n laboratorio
```
