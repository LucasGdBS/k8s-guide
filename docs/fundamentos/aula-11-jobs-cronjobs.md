---
title: "Aula 11 — Jobs e CronJobs"
parent: Fundamentos
nav_order: 12
---

# Aula 11 — Jobs e CronJobs

## Por que existem?

Deployments e StatefulSets têm um objetivo em comum: **ficar rodando para sempre**. Mas algumas tarefas têm fim:

- Rodar uma migration de banco antes de um deploy
- Gerar um relatório diário
- Fazer backup de dados à meia-noite
- Processar um arquivo e encerrar

Para isso existem o **Job** e o **CronJob**.

---

## Job vs CronJob

| Recurso | Quando usar |
|---|---|
| **Job** | Tarefa que roda **uma vez** e termina |
| **CronJob** | Tarefa que roda **repetidamente** em um horário definido |

O CronJob **cria Jobs** automaticamente no horário configurado. Três camadas:

```
CronJob → cria Job → Job cria Pod
```

---

## Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migration-job
  namespace: laboratorio
spec:
  backoffLimit: 3           # tentativas em caso de falha
  activeDeadlineSeconds: 60 # tempo máximo antes de desistir

  template:
    spec:
      restartPolicy: Never  # o Job controla as tentativas, não o pod
      containers:
        - name: migration
          image: busybox
          command:
            - sh
            - -c
            - |
              echo "Iniciando migration..."
              sleep 3
              echo "Migration concluída com sucesso!"
```

### restartPolicy: Never vs OnFailure

| Política | Comportamento em caso de falha |
|---|---|
| `Never` | Cria um **novo pod** a cada tentativa — o pod falho fica preservado para debug |
| `OnFailure` | **Reinicia o mesmo pod** — sobrescreve o estado anterior |

`Never` é preferido em produção porque preserva os pods falhos para inspeção de logs.

---

## CronJob

O formato do `schedule` segue o padrão cron do Unix:

```
┌── minuto (0-59)
│ ┌── hora (0-23)
│ │ ┌── dia do mês (1-31)
│ │ │ ┌── mês (1-12)
│ │ │ │ ┌── dia da semana (0-7, domingo=0 ou 7)
│ │ │ │ │
* * * * *
```

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-cronjob
  namespace: laboratorio
spec:
  schedule: "0 0 * * *"              # todo dia à meia-noite
  successfulJobsHistoryLimit: 3      # mantém os últimos 3 Jobs completados
  failedJobsHistoryLimit: 1          # mantém o último Job com falha
  concurrencyPolicy: Forbid          # não roda novo Job se o anterior ainda estiver rodando

  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: backup
              image: busybox
              command:
                - sh
                - -c
                - |
                  echo "Iniciando backup - $(date)"
                  sleep 5
                  echo "Backup concluído!"
```

### concurrencyPolicy

| Valor | Comportamento |
|---|---|
| `Forbid` | Pula a execução se o Job anterior ainda estiver rodando |
| `Allow` | Roda em paralelo com o Job anterior |
| `Replace` | Mata o Job anterior e sobe o novo |

`Forbid` é o mais seguro para tarefas que não devem se sobrepor (backups, migrations).

### startingDeadlineSeconds

Se o cluster ficar down e perder um horário agendado, por padrão aquela execução é **descartada**. Para tentar executar mesmo atrasado:

```yaml
startingDeadlineSeconds: 3600  # tenta até 1h depois do horário perdido
```

---

## Verificação

```bash
# Ver status do Job
kubectl get job migration-job -n laboratorio

# Logs do Job
kubectl logs job/migration-job -n laboratorio

# Ver pods completados
kubectl get pods -n laboratorio

# Ver CronJob e último disparo
kubectl get cronjob -n laboratorio

# Ver Jobs criados pelo CronJob
kubectl get jobs -n laboratorio
```

---

## Comandos de referência

```bash
# Forçar disparo imediato de um CronJob (útil para testar)
kubectl create job --from=cronjob/backup-cronjob teste-manual -n laboratorio

# Suspender um CronJob (para de disparar novos Jobs)
kubectl patch cronjob backup-cronjob -n laboratorio -p '{"spec":{"suspend":true}}'

# Reativar
kubectl patch cronjob backup-cronjob -n laboratorio -p '{"spec":{"suspend":false}}'

# Deletar Job e seus pods
kubectl delete job migration-job -n laboratorio
```
