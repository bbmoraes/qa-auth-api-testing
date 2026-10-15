# Test Execution Report — QA PORTFOLIO 02 | Authenticated API Testing

**Autoria:** Bianca | QA Júnior em formação · **Tipo:** projeto de estudos · **Data:** 15/10/2026

## 1. Execuções realizadas

| # | Onde | Quando (BRT) | O que rodou | Resultado |
|---|---|---|---|---|
| E1 | Postman web · Collection Runner | 15/10/2026 13:08:15 | Collection inteira (28 requisições; o Data-Driven é pulado sem arquivo de dados) | 74 verificações: **69 aprovadas, 5 reprovadas**, 0 erros |
| E2 | Postman web · Collection Runner (reexecução) | 15/10/2026 13:10:28 | Mesma configuração, para checar a repetibilidade | 74 verificações: **69 aprovadas, 5 reprovadas**, 0 erros |
| E3 | Postman web · envio individual | 15/10/2026 | CT-023 | 500; 1 de 3 verificações aprovadas (mesmo resultado do Runner) |
| E4 | GitHub Actions · Postman CLI · [run #1](https://github.com/bbmoraes/qa-auth-api-testing/actions/runs/38767277580) (push) | 15/10/2026 13:14:16 | 4 etapas (detalhe abaixo) | **Workflow: sucesso**. Etapas obrigatórias 96/96; etapas informativas 18/32 |
| E5 | GitHub Actions · [run #2](https://github.com/bbmoraes/qa-auth-api-testing/actions/runs/38768438160) (manual, `workflow_dispatch`) | 15/10/2026 13:22:24 | Mesmas 4 etapas | **Workflow: sucesso**. Regressão 66/66; defeitos conhecidos 3/8 |
| E6 | GitHub Actions · [run #3](https://github.com/bbmoraes/qa-auth-api-testing/actions/runs/38769035946) (push, após a correção do aviso de Node 20) | 15/10/2026 13:26:39 | Mesmas 4 etapas | **Workflow: sucesso**, sem avisos. Data-Driven válidos 30/30; etapas informativas com falha, como esperado |

### Detalhe do run #1 no GitHub Actions (Postman CLI)

| Etapa | Tipo | Iterações | Requisições | Verificações | Reprovadas |
|---|---|---|---|---|---|
| Regressão obrigatória (pastas 01–05 e 99) | bloqueia o workflow | 1 | 23 | 66 | **0** |
| Data-Driven com dados válidos (`bookings-valid.json`) | bloqueia o workflow | 5 | 10 | 30 | **0** |
| Defeitos conhecidos (pasta 06) | informativa | 1 | 3 | 8 | 5 |
| Data-Driven com dados inválidos (`bookings-invalid.json`) | informativa | 4 | 8 | 24 | 9 |

As etapas informativas terminam com *exit code 1*, como esperado: os defeitos continuam presentes. Elas usam `continue-on-error`, então o workflow fica verde apenas quando **toda a regressão obrigatória** passa. As falhas informativas aparecem no resumo do job e nos relatórios JUnit (artefato `relatorios-junit`).

## 2. Consolidado dos casos de teste

**Casos funcionais (CT-001 a CT-025):**

- **Planejados:** 25
- **Executados:** 25
- **Aprovados:** 22
- **Reprovados:** 3 (CT-023, CT-024 e CT-025)
- **Bloqueados:** 0
- **Não executados:** 0

**Casos Data-Driven (9 linhas de dados):**

- **Executados:** 9, todos no GitHub Actions
- **Aprovados:** 5 (DD-V01 a DD-V05)
- **Reprovados:** 4 (DD-I01 a DD-I04)

**Taxa de aprovação** = aprovados ÷ executados:

- Funcionais: 22 ÷ 25 = **88%**
- Data-Driven: 5 ÷ 9 = **55,6%**
- Total: 27 ÷ 34 = **79,4%**

Todos os reprovados correspondem a defeitos documentados em [`bug-reports/`](../bug-reports/README.md). Nenhum foi causado por erro de script ou pelo ambiente.

> O **TD-01** (limpeza das variáveis temporárias) é uma etapa técnica e não entra na contagem de casos, mas as 3 verificações dele estão nas 74 do Runner.

### Problema encontrado e corrigido no CI

No run #1 e no run #2, o GitHub mostrou um aviso: as actions `checkout@v4` e `upload-artifact@v4` usam Node.js 20, que está obsoleto. Conferi o `action.yml` de cada versão e atualizei para `checkout@v6` e `upload-artifact@v6`, que usam Node 24. Também fixei o runner em `ubuntu-24.04`, porque o GitHub avisou que o rótulo `ubuntu-latest` vai mudar de versão. O run #3 terminou sem nenhum aviso.

## 3. Classificação das falhas

| Item | Tipo |
|---|---|
| CT-023, CT-024 (500 com campo obrigatório ausente) | Defeito na API → BUG-01 |
| CT-025 (500 com text/plain) | Defeito na API → BUG-02 |
| DD-I01, DD-I02 (datas corrompidas) | Defeito na API → BUG-03 |
| DD-I03 (preço em texto vira null) | Defeito na API → BUG-04 |
| DD-I04 (decimais perdidos) | Defeito na API → BUG-05 |
| Falhas de script | Nenhuma: Errors = 0 em todas as execuções |
| Instabilidade externa | Nenhuma observada nas execuções registradas |
| Limitação do ambiente | Data-Driven indisponível no Postman web gratuito; executado via Postman CLI |

## 4. Investigação

1. **Ambiente antes do teste.** A documentação oficial (`/apidoc`) e o comportamento real foram explorados com requisições manuais, usando apenas reservas criadas e excluídas na própria exploração.
2. **Repetibilidade.** As execuções E1 e E2 deram o mesmo resultado. Cada execução usa um sobrenome único, criado no Pre-request, e exclui a própria reserva no final.
3. **Falhas reais ou falsas.** As falhas se repetiram em três lugares: no Runner, no envio individual e no GitHub Actions. As asserções testam exatamente o requisito definido. A sugestão automática do Postman de ajustar a asserção ("Fix failing tests") foi recusada, porque esconderia o defeito.
4. **Segurança.** O token é guardado só em variável de collection, nunca é impresso e é apagado no TD-01. Os relatórios do CI usam o formato JUnit, que não inclui corpos de resposta.

## 5. Evidências

| Arquivo | Conteúdo |
|---|---|
| [`01-collection-estrutura.jpg`](../evidence/screenshots/01-collection-estrutura.jpg) | Collection com 8 pastas no Postman |
| [`02-runner-completo-resumo-74-testes.jpg`](../evidence/screenshots/02-runner-completo-resumo-74-testes.jpg) | E1: resumo com 74 verificações |
| [`03-runner-completo-falhas.jpg`](../evidence/screenshots/03-runner-completo-falhas.jpg) | E1: as 5 falhas (CT-023 a CT-025) |
| [`04a`](../evidence/screenshots/04a-runner-completo-grid-CT001-CT012.jpg) · [`04b`](../evidence/screenshots/04b-runner-completo-grid-CT023-TD01.jpg) | E1: visão em grade |
| [`05-CT023-execucao-individual-500.jpg`](../evidence/screenshots/05-CT023-execucao-individual-500.jpg) | E3: envio individual |
| [`06-runner-reexecucao-repetibilidade.jpg`](../evidence/screenshots/06-runner-reexecucao-repetibilidade.jpg) | E2: reexecução |
| [`07`](../evidence/screenshots/07-ci-run1-resumo.jpg) · [`08`](../evidence/screenshots/08-ci-run1-etapas.jpg) | E4: resumo e etapas do workflow |
| [`09-ci-run1-regressao-66-assertions.jpg`](../evidence/screenshots/09-ci-run1-regressao-66-assertions.jpg) | E4: tabela da regressão (66/0) |
| [`10-ci-run1-defeitos-conhecidos.jpg`](../evidence/screenshots/10-ci-run1-defeitos-conhecidos.jpg) | E4: defeitos conhecidos (8/5) |
| [`11`](../evidence/screenshots/11-ci-run1-dd-invalidos-tabela.jpg) · [`12`](../evidence/screenshots/12-ci-run1-dd-invalidos-falhas.jpg) | E4: Data-Driven inválidos (24/9) |
| [`13-ci-run3-dd-validos-30-assertions.jpg`](../evidence/screenshots/13-ci-run3-dd-validos-30-assertions.jpg) | E6: Data-Driven válidos (5 iterações, 30/0) |
| [`14-ci-historico-3-execucoes.jpg`](../evidence/screenshots/14-ci-historico-3-execucoes.jpg) | Histórico do Actions com 3 execuções verdes |
| [`15-ci-run3-resumo.jpg`](../evidence/screenshots/15-ci-run3-resumo.jpg) | E6: resumo do run #3 |
