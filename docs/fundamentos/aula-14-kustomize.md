---
title: "Aula 14 — Kustomize"
parent: Fundamentos
nav_order: 15
---

# Aula 14 — Kustomize

## Por que o Kustomize existe?

Quando você precisa do mesmo Deployment em dev e prod, mas com pequenas diferenças (réplicas, imagem, recursos), as alternativas sem ferramenta são ruins: duplicar YAMLs é difícil de manter, e encher de variáveis vira um template complexo (esse é o caminho do Helm).

O Kustomize resolve isso com **base + overlays**: você mantém os manifestos originais intocados e aplica patches declarativos por ambiente — sem templates, sem variáveis, só YAML puro.

> Analogia: a base é a receita original. Os overlays são adaptações regionais — você não reescreve a receita inteira, só anota "substituir sal por shoyu".

**Kustomize vs Helm:** Helm é mais poderoso, mas mais complexo. Kustomize é nativo no `kubectl` (desde v1.14), não precisa instalar nada, e é ideal para variações simples entre ambientes.

---

## Componentes

| Componente | Função |
|---|---|
| `kustomization.yaml` | Arquivo de controle — lista o que incluir e o que modificar |
| **base** | Manifestos originais, sem conhecimento dos ambientes |
| **overlay** | Diretório por ambiente que referencia a base e aplica patches |
| **patch** | YAML com apenas os campos que mudam |

---

## Estrutura de diretórios

```
kustomize/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    └── prod/
        ├── kustomization.yaml
        └── patch-replicas.yaml
```

---

## Exemplo prático

### base/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: laboratorio
  labels:
    app: web
    aula: "14-kustomize"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:1.25
        ports:
        - containerPort: 80
```

### base/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
```

### overlays/dev/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

nameSuffix: -dev
```

### overlays/prod/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

nameSuffix: -prod

patches:
  - path: patch-replicas.yaml
```

### overlays/prod/patch-replicas.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: laboratorio
spec:
  replicas: 3
```

---

## Como o Kustomize identifica o alvo de um patch

O patch precisa informar `kind`, `metadata.name` e `metadata.namespace` mesmo que esses campos não mudem. Eles funcionam como **chave de identificação** — o Kustomize os usa para localizar qual recurso da base recebe o patch antes de mesclar o restante.

Isso é garantido pelo próprio Kubernetes: a combinação `kind + name + namespace` é única no cluster. Dois recursos do mesmo tipo não podem ter o mesmo nome no mesmo namespace — o API server rejeita. Portanto, a chave sempre aponta para exatamente um recurso.

> Atenção: o `metadata.name` no patch deve ser o nome **antes do suffix**. O Kustomize aplica o patch primeiro, depois adiciona o `-prod`.

---

## Comandos

```bash
# Visualizar o YAML final sem aplicar (muito útil para debug)
kubectl kustomize overlays/dev
kubectl kustomize overlays/prod

# Aplicar no cluster
kubectl apply -k overlays/dev
kubectl apply -k overlays/prod

# Deletar
kubectl delete -k overlays/dev
```

---

## Erros comuns

| Erro | Causa | Correção |
|---|---|---|
| `no such file or directory` | Nome do arquivo diferente do declarado em `resources:` | Conferir ortografia e extensão (`.yaml` vs `.yml`) |
| `'bases' is deprecated` | Campo `bases:` foi removido em versões novas | Substituir por `resources:` |
| Patch não aplicado | `metadata.name` no patch não bate com o nome na base | Usar o nome **sem** o suffix |
