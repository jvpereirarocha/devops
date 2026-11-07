---
name: k8s-troubleshoot
description: Conduz a resolução de um problema reportado no homelab (cluster k3s) e, quando o problema é resolvido, registra o incidente em detalhes em ~/devops/troubleshooting. Use sempre que o usuário reportar um problema ou erro na infra, por exemplo "tenho um problema", "está dando erro", "não funciona", "o pod não sobe", "o serviço sumiu", "me ajude a resolver", ou pedir para registrar/documentar um problema já resolvido.
---

# Troubleshooting com registro do incidente

Objetivo: resolver o problema que o usuário reportou e, no fim, deixar um registro em markdown em `~/devops/troubleshooting/`. Essa pasta é versionada no GitHub e é o histórico dos problemas enfrentados até a infra ficar 100% funcional.

## 1. Perguntar o namespace e a descrição

Sempre comece perguntando duas coisas, antes de rodar qualquer comando:

1. **Em qual namespace** está o problema.
2. **Uma breve descrição** do problema: o que acontece e o que deveria acontecer.

Use a ferramenta `AskUserQuestion` para o namespace. Liste como opções os namespaces de projeto que existem (confira com `kubectl get ns`), e o usuário pode escrever outro em "Other". Peça a descrição na mesma pergunta ou em texto livre.

Se o relato do usuário já trouxer o namespace ou a descrição, não ignore: apresente o que entendeu e peça para ele confirmar ou corrigir (por exemplo, "Entendi que o problema é X no namespace Y. Confere?").

Se o usuário não souber o namespace, ou o problema for do cluster todo, registre isso como "cluster" e siga.

## 2. Investigar

Use a skill `k8s-diagnose` para investigar, focando no namespace informado. Ela tem os comandos e a tabela de sintomas e causas. Guarde ao longo da investigação o que vai entrar no registro:

- os comandos que foram decisivos para o diagnóstico;
- as mensagens de erro e eventos exatos (copie o texto, não parafraseie);
- as hipóteses descartadas e por quê.

## 3. Corrigir

- A investigação é somente leitura. **Antes de qualquer alteração** no cluster ou nos manifests (`apply`, `patch`, `delete`, `edit`, `scale`, `rollout restart`, `helm`), mostre ao usuário exatamente o que vai mudar e espere a autorização.
- Se o usuário quiser acompanhar em tempo real, passe os comandos `watch`/`-w` para ele rodar em outro terminal antes de aplicar.
- Prefira corrigir nos manifests em `~/devops/manifests/` em vez de só no cluster, para a correção não se perder. Recursos gerenciados pelo ArgoCD são revertidos se mudados só no cluster.
- Depois de aplicar, verifique se o sintoma original sumiu.

## 4. Confirmar a resolução

O problema só está resolvido quando o usuário confirmar. Depois da verificação, pergunte se pode considerar resolvido.

- Se o usuário confirmar, siga para o registro.
- Se o usuário desistir ou adiar, pergunte se quer registrar mesmo assim, com status "em aberto" e o que já foi descoberto.

## 5. Registrar o incidente

Crie o arquivo em `~/devops/troubleshooting/` com o nome `AAAA-MM-DD-<resumo-curto>.md`, usando a data de hoje e um resumo em kebab-case (por exemplo, `2026-11-07-argocd-resource-quota.md`). Antes de criar, liste a pasta: se já existir um registro do mesmo problema, atualize-o em vez de criar outro.

Escreva em português, explicando em detalhes, para que o usuário entenda o problema ao reler meses depois. Explique os conceitos de Kubernetes envolvidos como para alguém que está aprendendo, em vez de só citar o nome deles. Use o modelo abaixo e remova seções que não se aplicarem. Como referência de nível de detalhe, veja `2026-11-07-argocd-resource-quota.md` na mesma pasta.

```markdown
# <Título que descreve o problema>

- **Data:** AAAA-MM-DD
- **Namespace:** `<namespace>`
- **Componente:** <app ou recurso afetado, com versão se souber>
- **Status:** resolvido | em aberto

## Sintoma

O que o usuário viu, com os comandos e saídas que mostravam o problema.

## Diagnóstico

Os comandos que levaram à causa, em ordem, e as mensagens de erro e eventos exatos.
Inclua as hipóteses descartadas e por quê.

## Causa

A causa raiz, em uma ou duas frases.

## Entendendo o problema

Explicação detalhada dos conceitos envolvidos (por exemplo: o que é uma ResourceQuota,
como funcionam requests e limits) e de por que eles causaram o sintoma. Use tabelas
e exemplos com os números reais do caso.

## Solução

O que foi alterado (arquivos, valores antes → depois, comandos aplicados) e por quê.
Como foi verificado que resolveu.

## Lições

O que verificar ou fazer diferente para não ter o mesmo problema.
```

### Dados sensíveis

O arquivo vai para o GitHub. Nunca coloque nele senhas, tokens, chaves, valores de Secrets, conteúdo de `kubeconfig` ou credenciais de banco. Se uma saída de comando tiver dados assim, troque o valor por `<redacted>`. IPs da rede local, nomes de recursos e mensagens de erro do Kubernetes podem ficar.

Ao terminar, diga ao usuário o caminho completo do arquivo e resuma em poucas linhas o que foi registrado.
