# Bug Reports — QA PORTFOLIO 02 | Authenticated API Testing

**Autoria:** Bianca | QA Júnior em formação
**Sistema testado:** [Restful-Booker](https://restful-booker.herokuapp.com), API pública de prática criada para ensinar testes de API.

> **Contexto importante.** O próprio site do Restful-Booker informa que a API foi feita para treino e vem com bugs de propósito. Os defeitos abaixo são reais e reproduzíveis, mas existem para fins didáticos. Eles foram documentados e **não** reportados ao mantenedor.

> **Critério usado.** A documentação oficial descreve as respostas de **sucesso** e os campos (nome e tipo), mas não define respostas de erro. Quando a documentação é omissa, o resultado esperado segue a semântica HTTP da RFC 9110: erro do cliente gera **4xx**, nunca **5xx**, e dados aceitos devem ser gravados como foram enviados. Comportamentos não documentados que não contrariam esse requisito foram registrados como **observação**, e não como bug.

## Resumo

| ID | Título | Classificação | Severidade | Prioridade | Casos |
|---|---|---|---|---|---|
| BUG-01 | Campos obrigatórios ausentes geram 500 em vez de 400 | Defeito confirmado | Média | Média | CT-023, CT-024 |
| BUG-02 | Content-Type não suportado gera 500 em vez de 400/415 | Defeito confirmado | Baixa | Baixa | CT-025 |
| BUG-03 | Datas fora do padrão são aceitas e gravadas corrompidas | Defeito confirmado | Alta | Alta | DD-I01, DD-I02 |
| BUG-04 | totalprice em texto é aceito e gravado como null | Defeito confirmado | Alta | Média | DD-I03 |
| BUG-05 | totalprice com decimais é gravado com perda de valor | Defeito confirmado | Média | Média | DD-I04 |
| OBS-01 | Credenciais inválidas retornam 200 com `reason` | Observação (não documentado) | — | — | CT-003, CT-004 |
| OBS-02 | Excluir a mesma reserva de novo retorna 405 | Observação (não documentado) | — | — | CT-019 |
| OBS-03 | `depositpaid: "sim"` vira `true` e checkout antes do checkin é aceito | Observação da exploração, **sem teste automatizado** | — | — | — |

**Ambiente de todos os itens:** Postman web (Cloud Agent) e Postman CLI no GitHub Actions (ubuntu-latest), ambiente `Restful-Booker | QA`, `baseUrl = https://restful-booker.herokuapp.com`. Data: 15/10/2026.

---

## BUG-01 — Campos obrigatórios ausentes geram 500

| Campo | Informação |
|---|---|
| **Descrição** | `POST /booking` sem `firstname`, ou sem o objeto `bookingdates`, responde **500 Internal Server Error** com o corpo `Internal Server Error`. |
| **Pré-condições** | Nenhuma |
| **Passos** | 1. `POST {{baseUrl}}/booking` com `Content-Type: application/json`.<br>2. Corpo sem `firstname`: `{"lastname":"Sem-Firstname","totalprice":100,"depositpaid":true,"bookingdates":{"checkin":"2026-11-10","checkout":"2026-11-12"}}`.<br>3. Repetir sem `bookingdates` (CT-024). |
| **Esperado** | `400 Bad Request`, porque o erro é do cliente. |
| **Obtido** | `500 Internal Server Error` nos dois casos. |
| **Severidade / prioridade** | Média / média: o cliente não recebe a indicação de qual campo corrigir, e o erro parece ser do servidor. |
| **Evidências** | [`05-CT023-execucao-individual-500.jpg`](../evidence/screenshots/05-CT023-execucao-individual-500.jpg) · [`03-runner-completo-falhas.jpg`](../evidence/screenshots/03-runner-completo-falhas.jpg) · [`10-ci-run1-defeitos-conhecidos.jpg`](../evidence/screenshots/10-ci-run1-defeitos-conhecidos.jpg) |
| **Status** | Reproduzido no Runner (2 execuções), em execução individual e no GitHub Actions. |

## BUG-02 — Content-Type não suportado gera 500

| Campo | Informação |
|---|---|
| **Passos** | `POST /booking` com `Content-Type: text/plain` e corpo `firstname=Ana;lastname=Texto-Puro`. |
| **Esperado** | `415 Unsupported Media Type` ou `400 Bad Request`. |
| **Obtido** | `500 Internal Server Error`. |
| **Severidade / prioridade** | Baixa / baixa: é um cenário de uso incorreto, mas a API não informa o motivo. |
| **Evidências** | [`03-runner-completo-falhas.jpg`](../evidence/screenshots/03-runner-completo-falhas.jpg) · [`10-ci-run1-defeitos-conhecidos.jpg`](../evidence/screenshots/10-ci-run1-defeitos-conhecidos.jpg) |

## BUG-03 — Datas fora do padrão são aceitas e gravadas corrompidas

| Campo | Informação |
|---|---|
| **Descrição** | A documentação mostra as datas no formato `AAAA-MM-DD`. Datas fora desse padrão não são rejeitadas: a API responde **200** e grava um valor diferente do enviado. |
| **Passos** | Data-Driven com `data/bookings-invalid.json`:<br>• **DD-I01:** `checkin: "10/11/2026"` (padrão brasileiro DD/MM).<br>• **DD-I02:** `checkin: "2026-13-45"` (data impossível). |
| **Esperado** | `400 Bad Request`. |
| **Obtido** | `200 OK` nos dois casos. **DD-I01:** a data volta reinterpretada e diferente da enviada (na exploração, `10/11/2026` virou `2026-10-11`, lida como mês/dia). **DD-I02:** a data gravada não segue o padrão `AAAA-MM-DD` e reprova no JSON Schema (na exploração, o valor devolvido foi `0NaN-aN-aN`). |
| **Severidade / prioridade** | Alta / alta: a reserva é gravada com datas erradas sem nenhum aviso, o que pode levar a uma hospedagem na data errada. |
| **Evidências** | [`12-ci-run1-dd-invalidos-falhas.jpg`](../evidence/screenshots/12-ci-run1-dd-invalidos-falhas.jpg) · [`11-ci-run1-dd-invalidos-tabela.jpg`](../evidence/screenshots/11-ci-run1-dd-invalidos-tabela.jpg) |
| **Limpeza** | As reservas criadas foram excluídas pelo DD-02 (201). |

## BUG-04 — totalprice em texto é aceito e gravado como null

| Campo | Informação |
|---|---|
| **Passos** | DD-I03: `totalprice: "cento e cinquenta"` (texto). A documentação define o campo como número. |
| **Esperado** | `400 Bad Request`. |
| **Obtido** | `200 OK`, com `totalprice` gravado como `null`. O JSON Schema reprova com a mensagem `data.booking.totalprice should be number`. |
| **Severidade / prioridade** | Alta / média: uma reserva fica gravada sem valor. |
| **Evidências** | [`12-ci-run1-dd-invalidos-falhas.jpg`](../evidence/screenshots/12-ci-run1-dd-invalidos-falhas.jpg) |

## BUG-05 — totalprice com decimais é gravado com perda de valor

| Campo | Informação |
|---|---|
| **Passos** | DD-I04: `totalprice: 199.9`. |
| **Esperado** | `200 OK`, com o valor gravado exatamente como enviado (o campo é um número na documentação). |
| **Obtido** | `200 OK`, mas o valor gravado é diferente do enviado (na exploração, `199.9` virou `199`). |
| **Severidade / prioridade** | Média / média: há perda silenciosa de valor monetário. |
| **Observação de QA** | Este caso mostra por que não basta checar o status: o código HTTP está certo, mas o dado está errado. Só a verificação de igualdade encontrou o problema. |

## Observações (não classificadas como bug)

- **OBS-01:** credenciais inválidas retornam `200 OK` com `{"reason":"Bad credentials"}`. A documentação não define o código de erro e, do ponto de vista de segurança, nenhum token é emitido. Por isso os testes validam a ausência do token e não presumem 401.
- **OBS-02:** um segundo `DELETE` na mesma reserva retorna `405 Method Not Allowed`. O código esperado seria `404`, mas a documentação é omissa. O teste aceita qualquer 4xx e registra o código.
- **OBS-03:** na exploração manual, `depositpaid: "sim"` foi gravado como `true`, e uma reserva com checkout antes do checkin foi aceita. Isso **não** está coberto por testes automatizados, então fica como sugestão de próximos casos.

## Limitações do ambiente

- A API é **pública e compartilhada**, volta ao estado inicial a cada 10 minutos e pode ficar instável. Os testes usam apenas registros criados pela própria execução (sobrenome único) e os excluem ao final.
- No plano gratuito do Postman web, rodar a collection com arquivo de dados exige upgrade. Por isso, o Data-Driven foi executado com o **Postman CLI no GitHub Actions**, que aceita arquivos locais sem custo.
