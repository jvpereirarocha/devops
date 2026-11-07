# ArgoCD sem application-controller e sem services (ResourceQuota)

- **Data:** 2026-11-07
- **Namespace:** `cicd-system`
- **Componente:** ArgoCD v3.3.8
- **Status:** resolvido (ArgoCD removido e manifests refeitos com Kustomize)

## Sintoma

- `kubectl get deploy -A` mostrava os deployments do ArgoCD no namespace `cicd-system`.
- `kubectl get svc -A` não mostrava o namespace `cicd-system`: não existia nenhum Service lá. O `get svc -A` só lista namespaces que têm pelo menos um service.
- O StatefulSet `argocd-application-controller` estava em `0/1` e nenhuma `Application` existia no cluster.

## Diagnóstico

```bash
kubectl get all -n cicd-system
kubectl get events -n cicd-system --sort-by=.lastTimestamp
kubectl describe resourcequota -n cicd-system
helm history argocd -n cicd-system
```

Foram encontrados dois problemas.

**1. A instalação via Helm tinha falhado no meio do caminho:**

```
1  Sat Apr 25 15:47:32 2026  failed  argo-cd-9.5.4  Release "argocd" failed: context canceled
```

`context canceled` indica que o comando foi interrompido (Ctrl+C ou timeout do `--wait`). Os 4 services estavam no manifest da release, mas não existiam no cluster.

**2. O application-controller era barrado pela ResourceQuota:**

```
create Pod argocd-application-controller-0 in StatefulSet argocd-application-controller failed error:
pods "argocd-application-controller-0" is forbidden: exceeded quota: cicd-system-quota,
requested: limits.cpu=500m, used: limits.cpu=1600m, limited: limits.cpu=2
```

## Entendendo o problema da quota

### Requests e limits

Cada container pode declarar dois valores de CPU:

- **request:** quanto de CPU o container tem garantido. O Kubernetes usa esse valor para decidir em qual node o pod cabe.
- **limit:** o máximo que o container pode usar. Se passar disso, ele é limitado (throttling).

Por exemplo, o `argocd-server` tem `requests.cpu: 100m` e `limits.cpu: 500m`. Ele normalmente usa pouco, mas pode chegar a meio core. A unidade `m` significa milicore: `1000m` é 1 core, e `500m` é meio core.

### ResourceQuota: um teto para o namespace inteiro

A `cicd-system-quota` limita a **soma** de todos os pods do namespace:

```yaml
hard:
  requests.cpu: "1"     # soma dos requests de todos os pods: no máximo 1 core
  limits.cpu: "2"       # soma dos limits de todos os pods: no máximo 2 cores
```

Antes de criar um pod, o Kubernetes verifica: "somando este pod com os que já existem, passa do teto?". Se passar, ele **recusa criar o pod**. É isso que o erro acima diz: "este pod pede 500m, os outros já usam 1600m, e 1600 + 500 = 2100m passa do teto de 2000m".

### LimitRange: o valor padrão para quem não declara nada

A `cicd-system-limits` (LimitRange) define valores padrão para containers que não declaram `resources`. O padrão é `limits.cpu: 500m`. Os values do Helm não definiam resources para o application-controller, então ele recebeu esse padrão.

### Por que a soma passava de 2

| Componente | limits.cpu |
|---|---|
| argocd-server | 500m |
| argocd-repo-server | 500m |
| argocd-applicationset-controller | 200m |
| argocd-notifications-controller | 200m |
| argocd-redis | 200m |
| **Subtotal (os que subiram)** | **1600m** |
| argocd-application-controller | 500m (padrão da LimitRange) |
| **Total necessário** | **2100m**, não cabe em 2000m |

O application-controller, que é o componente principal do ArgoCD, ficou de fora. Ele era o último da fila e não sobrou espaço na quota.

## Solução

1. O ArgoCD foi removido por completo: release do Helm, CRDs, ClusterRoles e namespace.
2. A instalação foi refeita sem Helm, com Kustomize, em `manifests/ci-cd/`:
   - `argocd-resources.yaml` define CPU e memória de **todos** os containers, inclusive o application-controller e os init containers, para nada depender do padrão da LimitRange.
   - O `limits.cpu` da quota subiu de `"2"` para `"3"`.

### Por que 3

Na nova instalação, a soma dos limits fica em **2200m**. Entraram o application-controller e o Dex, que vem no manifest oficial. Com o teto em `3`, sobram 800m de folga.

Essa folga importa durante um rolling update. O pod novo sobe antes de o antigo ser removido, então por alguns segundos os dois contam na quota ao mesmo tempo. Sem folga, a atualização poderia travar pelo mesmo motivo.

### Isso pesa no servidor?

Não. Subir o `limits.cpu` não reserva CPU no node, só permite que os pods somados *possam* chegar até 3 cores. O que o Kubernetes reserva de fato são os requests, e esses continuam baixos: 550m no total, dentro do `requests.cpu: "1"`, que não mudou. Na época, o node usava cerca de 5% de CPU.

## Lições

- **Ao definir uma ResourceQuota, some os limits de todos os componentes**, inclusive os que não declaram resources e recebem o padrão da LimitRange, e deixe folga para rolling updates.
- **Se um pod de Deployment ou StatefulSet não aparece, olhe os eventos do namespace** (`kubectl get events -n <ns>`). O erro de quota só aparece lá, não no `get pods`.
- **Não interrompa uma instalação no meio.** Uma release interrompida fica com recursos faltando. Confira o resultado com `helm history` ou `kubectl get all -n <ns>`.
- **`kubectl get <recurso> -A` só mostra namespaces que têm aquele recurso.** O namespace não aparecer na listagem não quer dizer que ele não existe.
