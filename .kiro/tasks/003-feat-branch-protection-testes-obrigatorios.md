# 003 - Tornar o check "Testes Unitários" obrigatório no branch `ia-main`

## 🔧 Configuração Inicial (LEIA ANTES DE INICIAR)

### Agent Responsável
**dev** - Este agent deve iniciar e concluir a implementação sozinho.

> ⚠️ **Fluxo desta task é apenas `dev → po`, sem `qa` e sem `devops`.**
> - O **qa** não tem papel útil aqui: não é uma mudança de UI/browser testável
>   via Playwright, é configuração de repositório GitHub.
> - O **devops** também não entra: não envolve infraestrutura AWS (o
>   `devops` só tem acesso somente-leitura via `aws-mcp`, sem `gh`).
> - O único agent do time com `Bash` liberado o suficiente para rodar `gh
>   api`/`gh` de configuração de repositório é o **dev** (o `po` só tem `gh
>   pr create` na allowlist; o `devops` não tem `gh` de forma alguma).
> - Ao concluir e validar, o **dev** notifica diretamente o **po** para
>   revisão e fechamento — não há etapa intermediária de qa/devops nesta
>   task.

### Branch Base
**SEMPRE `ia-main`**

### Worktree
Esta task será implementada em worktree isolado em
`.kiro/worktrees/003-feat-branch-protection-testes-obrigatorios/`

### Escopo
- **Somente configuração do repositório GitHub** (`FabioCLima/bia`, é o
  fork do usuário configurado como `origin`). Nenhuma alteração de código
  de aplicação (`api/`, `client/`) nem do workflow em si
  (`.github/workflows/testes-pr.yml`) é esperada — a menos que, durante a
  validação, se descubra algo genuinamente quebrado no workflow (nesse
  caso, documentar e avisar o po antes de alterar).
- Para o teste de bloqueio (PR com teste quebrado), pode ser necessário um
  branch/PR descartável — nunca commitar um teste quebrado de forma
  definitiva contra `ia-main`.

---

## ⚠️ CHECKLIST DE INÍCIO (OBRIGATÓRIO)

Antes de começar a implementar, o agent (dev) deve:

- [ ] **Verificar branch atual:** `git branch --show-current`
  - Se não estiver em `ia-main`, **PERGUNTAR** ao usuário se pode trocar
  - Aguardar autorização
  - Após autorização: `git checkout ia-main && git pull origin ia-main`

- [ ] **Mover task para doing:**
  ```bash
  mv .kiro/tasks/003-feat-branch-protection-testes-obrigatorios.md .kiro/tasks/doing/
  git add .kiro/tasks/
  git commit -m "move: task 003 para doing"
  git push origin ia-main
  ```

- [ ] **Criar worktree:**
  ```bash
  git worktree add .kiro/worktrees/003-feat-branch-protection-testes-obrigatorios -b feature/003-feat-branch-protection-testes-obrigatorios ia-main
  cd .kiro/worktrees/003-feat-branch-protection-testes-obrigatorios
  git branch --show-current  # Confirmar branch correto
  ```

---

## 📋 Tipo
**feat** - Habilitar proteção de branch para tornar um check de CI
obrigatório antes do merge.

## 📝 Resumo
Configurar o branch `ia-main` do repositório `FabioCLima/bia` para exigir
que o check "Testes Unitários" (workflow
`.github/workflows/testes-pr.yml`) passe com sucesso antes que um Pull
Request possa ser mergeado, fechando um gap deixado em aberto por uma task
legada do time anterior.

## 📖 Descrição
Como time de desenvolvimento do BIA, eu quero que o GitHub **bloqueie o
merge** de qualquer PR contra `ia-main` cujo check "Testes Unitários"
esteja falhando (ou ainda não tenha rodado), para que testes quebrados
nunca cheguem ao branch principal por descuido — hoje isso não acontece:
o workflow roda e reporta status, mas nada impede o merge mesmo com testes
vermelhos.

## Contexto (leia antes de implementar)

O workflow de testes a cada PR **já existe e já funciona**:
`.github/workflows/testes-pr.yml` (nome do job: **"Testes Unitários"**).
Ele:
- Dispara em `pull_request` contra `ia-main`;
- Roda `npm install` + `npm test` (Jest, `tests/unit/`, testes unitários
  mockados, sem dependência de banco);
- Já rodou com sucesso em dois PRs reais (PR #1 e PR #2, ambos
  `success`).

Essa automação veio de uma task legada do time anterior (Kiro CLI):
`.kiro/tasks/done/_legado-henrylle/006-feat-github-actions-testes-pr.md`.
Essa task legada já apontava exatamente o gap que esta task 003 resolve,
mas nunca foi fechada:
```
- [ ] O status do workflow aparece como check obrigatório no PR (visível na interface do GitHub)
```

**O gap real (escopo desta task):** o check do workflow **não é
obrigatório**. Uma consulta a
`gh api repos/FabioCLima/bia/branches/ia-main/protection` retornou:
```json
{ "message": "Branch not protected" }
```
(HTTP 404 — sem nenhuma regra de proteção configurada). Ou seja, hoje um
PR **pode ser mergeado mesmo que os testes falhem** — nada bloqueia isso
no GitHub.

## ⚠️ Ponto de atenção CRÍTICO — leia antes de aplicar qualquer configuração

O material de curso original (Kiro CLI, ~2025) sobre esse tópico pode
estar **desatualizado** — hoje já estamos bem depois disso, e o GitHub
pode ter mudado a forma recomendada de configurar proteção de branch (ex.:
"Rulesets" como sucessor mais novo e flexível das "Branch protection
rules" clássicas, mudanças na API/CLI). Portanto, o **dev DEVE**:

- **NÃO assumir sintaxe de comando de memória/curso.** Verificar a
  documentação oficial atual do GitHub antes de aplicar qualquer mudança
  — via `gh api` introspection (ex.: `gh api repos/{owner}/{repo} --jq
  '.default_branch'`, explorar os endpoints disponíveis), `gh help api`,
  `gh help ruleset` (se existir no `gh` instalado), ou documentação oficial
  do GitHub (docs.github.com) sobre "branch protection rules" e
  "repository rulesets".
- **Confirmar qual mecanismo está disponível/recomendado no momento da
  execução:**
  - Branch protection clássica: `gh api repos/{owner}/{repo}/branches/{branch}/protection` com `PUT` (endpoint que já foi consultado nesta task, retornou 404 = não configurado);
  - Ou Rulesets: `gh api repos/{owner}/{repo}/rulesets` (mecanismo mais novo).
- **Documentar na task qual mecanismo foi usado e por quê** (ex.:
  disponibilidade no plano do repositório, simplicidade, recomendação
  atual do GitHub, permissões do fork, etc.).
- **Testar de verdade a trava** (não basta o check aparecer como
  obrigatório na tela de configurações — precisa comprovar o bloqueio na
  prática):
  1. Abrir um PR de teste contra `ia-main` com um teste unitário
     **propositalmente quebrado** (ex.: alterar temporariamente uma
     asserção em `tests/unit/`);
  2. Confirmar que o check "Testes Unitários" falha nesse PR;
  3. Confirmar que o **botão de merge fica desabilitado/bloqueado** pelo
     GitHub (não apenas que o check aparece como failed — o merge em si
     precisa estar impedido);
  4. **Reverter o teste quebrado** (o PR de teste não deve ser mergeado
     com o teste quebrado; feche-o ou corrija antes de decidir o que
     fazer com ele — nunca deixe um teste quebrado definitivo em
     qualquer branch, nem mesmo um branch descartável que fique aberto).

## ✅ Critérios de Aceitação

### Funcionalidades Principais
- [ ] Branch `ia-main` protegido/com regra exigindo que o check "Testes
      Unitários" passe antes do merge (via branch protection clássica ou
      via ruleset — o mecanismo escolhido deve ser documentado com
      justificativa na seção "Notas Técnicas" desta task).
- [ ] PR de teste com um teste quebrado propositalmente confirma, na
      prática, que o **merge é bloqueado pelo GitHub** (evidência: print
      ou saída de `gh pr view`/`gh api` mostrando `mergeable_state` ou
      equivalente, e/ou descrição do botão de merge desabilitado na UI).
- [ ] Teste quebrado revertido/removido após a validação — nenhum lixo
      (teste quebrado, branch de teste esquecido) deixado no repositório
      ao final da task.
- [ ] Mecanismo usado (branch protection clássica ou ruleset) documentado
      na task, com a justificativa da escolha.

### Integração
- [ ] Nenhuma alteração de código de aplicação (`api/`, `client/`) — o
      escopo é estritamente configuração de repositório GitHub.
- [ ] O workflow `.github/workflows/testes-pr.yml` continua funcionando
      normalmente para PRs com testes passando (não regredir o que já
      funciona).

## 🧪 Testes (dev)
- [ ] Confirmar estado inicial: `gh api
      repos/FabioCLima/bia/branches/ia-main/protection` (documentar
      resultado antes da mudança, para comparação).
- [ ] Aplicar a configuração de proteção/regra exigindo o check "Testes
      Unitários".
- [ ] Confirmar via `gh api` que a proteção está ativa e que o check
      correto está listado como obrigatório.
- [ ] Cenário de bloqueio: abrir PR de teste (branch descartável) contra
      `ia-main` com teste unitário quebrado propositalmente, confirmar
      falha do check e bloqueio do merge.
- [ ] Reverter o teste quebrado / fechar o PR de teste sem mergeá-lo.
- [ ] (Opcional, se viável sem custo/risco) Confirmar que um PR com testes
      passando continua mergeável normalmente após a proteção ativa.

## 📚 Definição de Pronto (DoD)
- [ ] Configuração implementada e validada na prática (bloqueio real
      testado, não apenas configuração visual)
- [ ] Todos os itens do checklist marcados ✅
- [ ] Commits descritivos e frequentes (se houver alteração de arquivos
      versionados; a mudança principal é via `gh api`, fora do controle de
      versão do código, mas qualquer PR de teste usado na validação deve
      ser documentado)
- [ ] Push do branch da task realizado (mesmo que o conteúdo principal
      seja a atualização desta própria task com as evidências)
- [ ] Nenhum teste quebrado remanescente em qualquer branch do
      repositório
- [ ] Documentação atualizada nesta task (mecanismo escolhido, evidências
      de teste, comandos usados)

---

## 🎯 CHECKLIST DE IMPLEMENTAÇÃO (MARCAR DURANTE O TRABALHO)

### Configuração
- [ ] Worktree criado e branch correto confirmado
- [ ] Confirmado acesso via `gh auth status` ao repositório
      `FabioCLima/bia`

### Investigação (antes de aplicar qualquer mudança)
- [ ] Consultado o estado atual de proteção do branch `ia-main` (`gh api
      repos/FabioCLima/bia/branches/ia-main/protection`) e documentado o
      resultado
- [ ] Verificada a documentação oficial atual do GitHub sobre proteção de
      branch (branch protection rules clássicas vs. rulesets) — **sem
      assumir sintaxe do material de curso**
- [ ] Decidido e justificado qual mecanismo será usado (branch protection
      clássica ou ruleset)

### Desenvolvimento
- [ ] Aplicada a configuração escolhida via `gh api` (com `PUT`/`POST`
      conforme o mecanismo), exigindo o check "Testes Unitários" como
      obrigatório para `ia-main`
- [ ] Confirmado via `gh api` (`GET`) que a regra está ativa e lista o
      check correto

### Validação do bloqueio real
- [ ] Criado branch descartável com um teste unitário propositalmente
      quebrado (ex.: `tests/unit/controllers/...`)
- [ ] Aberto PR de teste contra `ia-main` a partir desse branch
- [ ] Confirmado que o check "Testes Unitários" falhou no PR
- [ ] Confirmado que o GitHub bloqueia o merge (evidência registrada:
      saída de `gh pr view <n> --json mergeable,mergeStateStatus` ou
      equivalente)
- [ ] Revertido o teste quebrado / PR de teste fechado sem merge
- [ ] Branch de teste descartável removido (local e remoto, se aplicável)

### Finalização
- [ ] Código/configuração revisado
- [ ] Task atualizada com todas as evidências e decisões
- [ ] Commits finalizados com mensagens descritivas
- [ ] Push do branch da task realizado
- [ ] Todos os itens acima marcados ✅

---

## ⚠️ FINALIZAÇÃO DA TASK (OBRIGATÓRIO)

Quando o **dev** concluir a implementação e a validação:

### 1. Verificação Final
```bash
# Garantir que está no worktree correto
pwd
# Deve estar em: /caminho/do/projeto/.kiro/worktrees/003-feat-branch-protection-testes-obrigatorios

# Verificar branch
git branch --show-current
# Deve mostrar: feature/003-feat-branch-protection-testes-obrigatorios
```

### 2. Commit e Push Final
```bash
git add .
git commit -m "feat: torna o check Testes Unitários obrigatorio no branch ia-main"
git push origin feature/003-feat-branch-protection-testes-obrigatorios
```

### 3. Notificar diretamente o PO (sem qa/devops nesta task)
Não há etapa de qa ou devops nesta task. Ao concluir e validar o
bloqueio real (PR de teste quebrado + merge bloqueado + reversão), o
**dev** notifica diretamente o **po**.

### 4. Voltar para Raiz e Notificar PO
```bash
cd ../../..  # Voltar para raiz do projeto
```

**NOTIFICAR O PO:**
> "Task 003 concluída pelo dev. Proteção de branch aplicada em `ia-main`
> exigindo o check 'Testes Unitários'. Validação real feita: PR de teste
> com teste quebrado confirmou bloqueio de merge pelo GitHub; teste
> quebrado revertido. Mecanismo usado: [branch protection clássica /
> ruleset], justificativa documentada na task. Branch
> `feature/003-feat-branch-protection-testes-obrigatorios` com push
> realizado. Aguardando revisão do PO para encerramento."

**⚠️ NÃO REMOVER O WORKTREE. Apenas o PO faz isso após o PR ser
mergeado.**

**⚠️ Esta task não gera necessariamente um PR de código de aplicação** —
a mudança principal é configuração de repositório via `gh api`. Ainda
assim, siga o fluxo de PR normal para registrar a atualização desta task
(com as evidências) contra `ia-main`, conforme o encerramento abaixo.

---

## 🎯 ENCERRAMENTO PELO PO (QUANDO NOTIFICADO)

### 1. Revisão
```bash
# Entrar no worktree para revisar
cd .kiro/worktrees/003-feat-branch-protection-testes-obrigatorios

# Revisar a task atualizada com evidências
# Confirmar, via gh api, que a proteção do branch ia-main está ativa
gh api repos/FabioCLima/bia/branches/ia-main/protection

# Verificar se todos os itens estão ✅
```

### 2. Aprovar e Mover para Done
```bash
# Voltar para raiz
cd ../../..

# Mover task para done
mv .kiro/tasks/doing/003-feat-branch-protection-testes-obrigatorios.md .kiro/tasks/done/

# Commit e push no ia-main
git checkout ia-main
git add .kiro/tasks/
git commit -m "move: task 003 para done"
git push origin ia-main
```

### 3. Abrir Pull Request
```bash
# ANTES de abrir PR: confirmar que está no branch da feature (NUNCA em ia-main)
cd .kiro/worktrees/003-feat-branch-protection-testes-obrigatorios
git branch --show-current
# Comparar a saída com o nome esperado: feature/003-feat-branch-protection-testes-obrigatorios
# Se não bater (ex.: mostrar "ia-main"), PARAR — NÃO rodar o gh pr create abaixo

# Só prosseguir se o branch confirmado bater com o esperado:
gh pr create --base ia-main --title "003: Tornar check Testes Unitários obrigatorio em ia-main" --body "Closes task 003"
```

### 4. Após PR Mergeado
```bash
# Voltar para raiz
cd ../../..

# Remover worktree
git worktree remove .kiro/worktrees/003-feat-branch-protection-testes-obrigatorios

# Ou com força se necessário:
# git worktree remove --force .kiro/worktrees/003-feat-branch-protection-testes-obrigatorios

# Limpar registros
git worktree prune

# (Opcional) Deletar branch local
git branch -d feature/003-feat-branch-protection-testes-obrigatorios

# Notificar conclusão
```

---

## 📊 Notas Técnicas

### Estado inicial confirmado (na criação desta task)
```bash
$ gh api repos/FabioCLima/bia/branches/ia-main/protection
{
  "message": "Branch not protected",
  ...
}
```
Ou seja, hoje **não existe nenhuma regra de proteção** em `ia-main` — nem
clássica, nem ruleset.

### Mecanismo escolhido (preencher durante a implementação)
> **[A PREENCHER PELO DEV]** — descrever aqui qual mecanismo foi usado
> (branch protection clássica via `PUT
> /repos/{owner}/{repo}/branches/{branch}/protection` ou ruleset via
> `POST /repos/{owner}/{repo}/rulesets`), e por quê (ex.: disponibilidade
> no plano do repositório/fork, simplicidade de manutenção, recomendação
> atual da documentação oficial do GitHub no momento da execução).

### Evidências de validação (preencher durante a implementação)
> **[A PREENCHER PELO DEV]** — link/número do PR de teste, saída relevante
> de `gh pr view`/`gh api` mostrando o merge bloqueado, e confirmação da
> reversão do teste quebrado.

### Referências úteis
- Workflow existente: `.github/workflows/testes-pr.yml` (job "Testes
  Unitários")
- Task legada de origem do gap: `.kiro/tasks/done/_legado-henrylle/006-feat-github-actions-testes-pr.md`
- Testes unitários do projeto: `tests/unit/` (Jest, mockados, sem
  dependência de banco)

## 💼 Valor de Negócio
**Alto** - Sem essa proteção, a suíte de testes automatizados (task 006
legada) tem valor apenas informativo: qualquer PR pode ser mergeado com
testes quebrados, e o time só percebe depois. Tornar o check obrigatório
fecha esse gap com esforço técnico baixo, mas impacto direto na qualidade
do branch principal.

## 🎯 Estimativa
**2 Story Points** - A configuração em si é simples (uma chamada de API),
mas a task exige investigação da documentação atual do GitHub (evitar
assumir sintaxe desatualizada) e uma validação prática real (PR de teste
com bloqueio confirmado e revertido), o que adiciona esforço e cuidado
extra.

## 🔗 Dependências
Depende do workflow `.github/workflows/testes-pr.yml` já existir e estar
funcional (confirmado: já funciona, rodou com sucesso nos PRs #1 e #2).

---

## 📚 Referências
- [Worktree Workflow](.kiro/docs/worktree-workflow.md)
- [Worktree Steering](.kiro/docs/worktree-steering.md)
- [Task Template](.kiro/docs/task-template-with-worktree.md)
- [Especificação do PO](.kiro/agents/po/especificacao.md)
- [Task legada 006 (origem do gap)](.kiro/tasks/done/_legado-henrylle/006-feat-github-actions-testes-pr.md)
