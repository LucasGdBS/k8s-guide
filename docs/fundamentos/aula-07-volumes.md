---
title: "Aula 07 — Volumes e Armazenamento"
parent: Fundamentos
nav_order: 8
---

# Aula 07 — Volumes e Armazenamento

## O problema do armazenamento efêmero

Containers são efêmeros — quando um Pod morre, tudo que estava no filesystem do container vai junto:

```
Pod (MySQL) escreve dados em /var/lib/mysql
Pod morre e sobe de novo
/var/lib/mysql está vazio — todos os dados perdidos
```

Volumes resolvem isso criando armazenamento que sobrevive ao ciclo de vida do container.

---

## A hierarquia do armazenamento

```
PersistentVolume (PV)
    └── representa o disco real (local, NFS, AWS EBS, etc.)
        └── recurso global — sem namespace

PersistentVolumeClaim (PVC)
    └── solicitação de armazenamento feita pelo Pod
        └── "preciso de 500Mi com ReadWriteOnce"

StorageClass
    └── define como PVs são provisionados automaticamente
```

**Analogia:** o PV é o apartamento disponível, o PVC é o contrato de aluguel, e a StorageClass é a imobiliária que constrói apartamentos sob demanda.

---

## Por que o PVC existe?

O Pod não fala diretamente com o PV — ele faz um pedido descrevendo o que precisa:

```
Pod → "preciso de 500Mi com ReadWriteOnce"
         ↓
      PersistentVolumeClaim
         ↓
      K8s procura um PV que satisfaça o pedido
         ↓
      PV compatível encontrado → BOUND (vinculado)
```

O PVC **desacopla o Pod do disco real**. O mesmo YAML do Pod funciona em qualquer cloud — só muda o PV (ou a StorageClass) que provê o disco físico.

Quando um PVC é vinculado a um PV, esse PV fica **exclusivo** — nenhum outro PVC pode usá-lo simultaneamente (no modo `ReadWriteOnce`).

---

## Access Modes

| Mode | Abreviação | Quem pode montar |
|---|---|---|
| `ReadWriteOnce` | RWO | Um único node, leitura e escrita |
| `ReadOnlyMany` | ROX | Múltiplos nodes, só leitura |
| `ReadWriteMany` | RWX | Múltiplos nodes, leitura e escrita |

---

## PV e StorageClass em produção

Em produção você raramente cria PVs manualmente. A StorageClass provisiona o disco do tamanho exato que o PVC pediu:

```
PVC pede 500Mi → StorageClass cria disco de 500Mi → nada desperdiçado
```

No lab usamos `hostPath` + PV manual para entender o conceito. Em produção, a StorageClass faz isso automaticamente.

---

## Criando PV e PVC manualmente

Para vincular manualmente um PV a um PVC (sem StorageClass), ambos precisam ter `storageClassName: ""`:

```yaml
# PersistentVolume — representa o disco físico disponível no cluster
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-lab
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: ""   # sem StorageClass — vínculo manual
  hostPath:
    path: /tmp/pv-lab    # no Kind, usamos hostPath (disco local do node)
---
# PersistentVolumeClaim — o Pod solicita armazenamento
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-lab
  namespace: laboratorio
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ""   # deve bater com o PV
  resources:
    requests:
      storage: 500Mi
```

```bash
kubectl apply -f storage.yaml
kubectl get pv
kubectl get pvc -n laboratorio
```

Saída esperada — ambos com `STATUS: Bound`:

```
NAME     CAPACITY   ACCESS MODES   STATUS   CLAIM                 RECLAIM POLICY
pv-lab   1Gi        RWO            Bound    laboratorio/pvc-lab   Retain
```

---

## Usando o PVC num Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-storage
  namespace: laboratorio
spec:
  containers:
    - name: app
      image: alpine
      command: ["sleep", "3600"]
      volumeMounts:
        - name: meu-volume
          mountPath: /dados         # pasta dentro do container
  volumes:
    - name: meu-volume
      persistentVolumeClaim:
        claimName: pvc-lab          # referencia o PVC pelo nome
```

```bash
# Escreve um arquivo no volume
kubectl exec -it pod-storage -n laboratorio -- sh -c "echo 'hello k8s' > /dados/teste.txt"

# Deleta e recria o Pod
kubectl delete pod pod-storage -n laboratorio
kubectl apply -f pod-storage.yaml

# O arquivo persiste
kubectl exec -it pod-storage -n laboratorio -- cat /dados/teste.txt
```

---

## Reclaim Policy

Define o que acontece com o PV quando o PVC é deletado:

| Policy | Comportamento |
|---|---|
| `Retain` | PV e dados preservados — requer limpeza manual |
| `Delete` | PV e dados deletados automaticamente |
| `Recycle` | Dados apagados, PV fica disponível novamente (deprecated) |

---

## Comandos de referência

```bash
# PersistentVolume (recurso global, sem -n)
kubectl get pv
kubectl describe pv <nome>
kubectl delete pv <nome>

# PersistentVolumeClaim
kubectl get pvc -n laboratorio
kubectl describe pvc <nome> -n laboratorio
kubectl delete pvc <nome> -n laboratorio
```
