# Encontro 12 — Prática 2: persistência, transação e auditoria

## Informações gerais

- **Duração:** 90 minutos
- **Modalidade:** individual
- **Consulta:** permitida aos materiais da disciplina, à documentação oficial e
  às anotações pessoais
- **Entrega:** link do repositório GitHub enviado até o final da aula pelo Google
  Sala de Aula
- **Ponto de partida:** projeto NestJS desenvolvido nos encontros 07 a 11

A consulta não autoriza comunicação ou compartilhamento de solução entre
estudantes durante a atividade.

## Situação-problema

A aprovação de uma solicitação passou a representar uma reserva real no
orçamento do centro de custo. Quando um gestor aprova uma compra, o sistema
precisa confirmar a solicitação, descontar seu valor do saldo disponível e
registrar a decisão na auditoria.

Essas alterações não podem ocorrer parcialmente. Uma solicitação aprovada sem
desconto no orçamento produziria um saldo incorreto; um desconto sem aprovação
reservaria recursos sem uma decisão válida; e uma decisão sem auditoria não
permitiria identificar o responsável.

Também é possível que dois gestores consultem o mesmo saldo e tentem aprovar
compras simultaneamente. A aplicação deve impedir saldo negativo e rejeitar
decisões baseadas em versões antigas.

Sua tarefa é adaptar a API para realizar essa aprovação de forma persistente,
atômica, auditável e segura.

## O que é um centro de custo?

Um centro de custo é uma forma de a organização identificar qual setor,
departamento, projeto ou unidade é responsável por uma despesa. Ele permite
separar e acompanhar os gastos de diferentes partes da empresa.

Por exemplo, uma empresa pode possuir os seguintes centros de custo:

| Código | Nome | Exemplos de despesas |
|---|---|---|
| `TI-DEV` | Desenvolvimento de Sistemas | monitores, licenças e treinamentos |
| `TI-INFRA` | Infraestrutura de TI | servidores, redes e armazenamento |
| `RH` | Recursos Humanos | recrutamento, capacitação e benefícios |

Nesta atividade, cada centro de custo possui um saldo disponível, que
representa quanto ainda pode ser comprometido em novas solicitações. Esse
saldo é uma simplificação didática de um controle orçamentário; ele não
representa necessariamente o saldo de uma conta bancária.

Considere o seguinte exemplo:

- o centro de custo `TI-DEV` possui saldo disponível de R$ 5.000,00;
- uma solicitação de monitor possui valor estimado de R$ 1.200,00;
- se a solicitação for aprovada, R$ 1.200,00 são reservados;
- o novo saldo disponível passa a ser R$ 3.800,00.

Se outra solicitação exigir R$ 4.000,00, ela não pode ser aprovada usando
esse centro de custo, pois o saldo restante é insuficiente.

Cada solicitação deve estar associada a um centro de custo. Assim, no momento
da aprovação, a API sabe de qual orçamento retirar o valor. O código identifica
o centro de custo; o saldo disponível informa sua capacidade atual; e a versão
permite detectar se outro gestor alterou esse saldo desde a última consulta.

Neste exercício, uma reserva é considerada definitiva no momento da aprovação.
Estorno, suplementação de orçamento, fechamento contábil e integração com um
sistema financeiro estão fora do escopo.

## Requisitos obrigatórios

### 1. Evolução do modelo

Evolua o banco por meio de uma nova migration. A solução deve representar:

- o valor estimado de cada solicitação;
- um centro de custo identificado por código único;
- o saldo disponível do centro de custo;
- a versão atual do centro de custo;
- a associação entre solicitação e centro de custo.

O banco deve impedir valores estimados negativos, saldos negativos, códigos
duplicados e associações com centros de custo inexistentes.

A migration deve possuir aplicação e reversão coerentes. Não edite migrations
já aplicadas e não use `synchronize` para realizar a alteração.

### 2. Seed identificado e repetível

Atualize o seed para criar um centro de custo de teste e duas solicitações
pendentes associadas a ele:

1. uma solicitação cujo valor possa ser coberto pelo saldo;
2. uma solicitação cujo valor seja superior ao saldo disponível.

O código do centro de custo deve conter os quatro últimos dígitos da matrícula
do estudante. Exemplo: para a matrícula `20261234`, pode ser usado
`CC-1234`.

O seed deve poder ser executado repetidamente sem duplicar o centro de custo ou
as solicitações que ele administra.

### 3. Consulta do centro de custo

Implemente o endpoint `GET /centros-custo/:codigo`.

Ele deve:

- exigir JWT válido;
- permitir que o cliente consulte saldo e versão atuais;
- retornar `404 Not Found` quando o código não existir;
- não expor credenciais, segredos ou informações de outros mecanismos internos.

### 4. Aprovação com reserva de orçamento

Adapte o endpoint `PATCH /solicitacoes/:id/aprovar`.

A requisição deve informar:

- a versão da solicitação consultada;
- a versão do centro de custo consultado.

O ator deve ser obtido do JWT. O corpo não pode permitir que o cliente escolha
o responsável pela decisão, o saldo final, o status final ou valores de versão
diferentes dos usados no controle de concorrência.

### 5. Regras da aprovação

A operação deve respeitar as seguintes regras:

- somente o papel `gestor` pode aprovar;
- apenas solicitações `pendente` podem ser aprovadas;
- o centro de custo associado deve existir;
- o saldo disponível deve ser suficiente para o valor estimado;
- saldo e valor devem ser tratados sem perda de precisão monetária;
- uma aprovação aceita muda o status para `aprovada`;
- o valor estimado é descontado do saldo;
- as versões da solicitação e do centro de custo são incrementadas;
- uma versão antiga de qualquer um dos registros impede a operação;
- o saldo nunca pode ficar negativo.

### 6. Transação e rollback

A mudança de status, o desconto do saldo e o registro de auditoria formam uma
única operação de negócio.

Se qualquer etapa falhar, nenhuma alteração pode permanecer confirmada. A
solicitação deve continuar pendente, o saldo deve conservar o valor anterior e
não deve existir auditoria de aprovação concluída.

### 7. Auditoria

Uma aprovação concluída deve registrar, no mínimo:

- identidade do ator;
- ação realizada;
- identificador da solicitação;
- código do centro de custo;
- valor reservado;
- saldo anterior e saldo resultante;
- versões utilizadas na decisão;
- instante da operação.

Tentativas recusadas por autenticação, autorização, saldo, estado ou
concorrência não devem gerar auditoria de aprovação concluída.

### 8. Concorrência

A solução deve lidar com duas tentativas baseadas nas mesmas versões da
solicitação ou do centro de custo. Somente uma operação pode ser confirmada.

A tentativa posterior deve retornar `409 Conflict`, preservar o saldo e não
criar nova auditoria de sucesso.

### 9. Configuração segura

Mantenha configurações do PostgreSQL e do JWT em variáveis de ambiente.
Entregue `.env.example` sem segredos reais e mantenha `.env` ignorado pelo Git.

Não inclua senhas, hashes, tokens, segredos ou credenciais do banco no código,
nas auditorias ou nas evidências.

### 10. Tratamento HTTP

A solução deve apresentar os seguintes comportamentos:

- entrada inválida: `400 Bad Request`;
- token ausente, inválido ou expirado: `401 Unauthorized`;
- usuário autenticado sem papel de gestor: `403 Forbidden`;
- solicitação ou centro de custo inexistente: `404 Not Found`;
- saldo insuficiente, estado incompatível ou versão antiga: `409 Conflict`;
- aprovação válida: resposta de sucesso.

## Casos de teste

Execute e registre os casos abaixo no Thunder Client, Insomnia, Postman ou com
`curl`.

| Caso | Requisição ou condição | Identidade | Resultado esperado |
|---:|---|---|---|
| 1 | consultar centro de custo existente | Gestor | `200`, saldo e versão |
| 2 | consultar centro de custo inexistente | Gestor | `404` |
| 3 | aprovar solicitação coberta pelo saldo | Gestor | sucesso; solicitação, saldo e auditoria atualizados |
| 4 | aprovar sem token | — | `401`, nenhuma alteração |
| 5 | aprovar como solicitante ou auditor | Papel sem permissão | `403`, nenhuma alteração |
| 6 | aprovar solicitação inexistente | Gestor | `404`, saldo preservado |
| 7 | aprovar com saldo insuficiente | Gestor | `409`, solicitação pendente e saldo preservado |
| 8 | aprovar solicitação já decidida | Gestor | `409`, sem novo desconto |
| 9 | aprovar com versão antiga da solicitação | Gestor | `409`, estado preservado |
| 10 | aprovar com versão antiga do centro de custo | Gestor | `409`, estado preservado |
| 11 | provocar falha depois do desconto e antes da auditoria | Gestor | rollback da solicitação e do saldo |
| 12 | executar o seed duas vezes | — | nenhum registro duplicado |
| 13 | recriar banco vazio e executar migrations e seed | — | schema e dados reconstruídos |

No teste de rollback, descreva como a falha foi simulada e remova a simulação
antes da entrega.

## Entrega pelo GitHub

Crie ou atualize um repositório no GitHub e envie o link na atividade
correspondente do Google Sala de Aula até o final do encontro. Confirme que o
professor possui acesso. A avaliação considerará o último commit realizado
dentro do prazo.

O repositório deve conter:

1. código-fonte da aplicação;
2. migrations necessárias para reconstruir o schema;
3. seed identificado e repetível;
4. `package.json`, `package-lock.json`, `Dockerfile` e `compose.yaml`;
5. `.env.example` e `.gitignore`, sem `.env`;

Não entregue `node_modules`, `dist`, `.env`, tokens, senhas, hashes ou outros
segredos. Antes do envio, confirme que outra pessoa consegue reconstruir o
banco e executar a aplicação apenas com os arquivos do repositório e as
instruções do `README.md`.
