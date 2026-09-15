# Encontro 11 — Atividade prática de revisão

## Tema

Persistência, migrations, seeds, transações, auditoria e concorrência.

## Objetivo

Revisar os conhecimentos trabalhados entre os encontros 07 e 10 por meio de
uma alteração no projeto de solicitações.

## Ponto de partida

Utilize o mesmo projeto desenvolvido nas aulas anteriores. A aplicação deve
continuar sendo executada com Docker Compose e possuir:

- PostgreSQL configurado;
- solicitações persistidas pelo TypeORM;
- migrations em lugar de sincronização automática;
- dados iniciais preparados por seed;
- autenticação com JWT e autorização por papéis;
- controle de versão das solicitações;
- registro de auditoria.

## Situação

A equipe de compras percebeu que nem toda solicitação deve ser aprovada.
Algumas precisam ser rejeitadas porque possuem informações insuficientes,
estão associadas ao centro de custo incorreto ou não atendem às regras da
organização.

Atualmente, a API permite criar, consultar e aprovar solicitações, mas não
oferece uma forma segura de registrar uma rejeição. A empresa precisa saber
quem tomou a decisão, quando ela ocorreu e qual foi a justificativa apresentada.

Duas pessoas também podem consultar a mesma solicitação e tentar tomar
decisões diferentes. A aplicação não pode permitir que uma decisão baseada em
informações antigas sobrescreva silenciosamente a mais recente.

## O que deve ser acrescentado

A API deve passar a representar o estado `rejeitada` e disponibilizar a
operação:

| Método e endpoint | Finalidade |
|---|---|
| `PATCH /solicitacoes/:id/rejeitar` | rejeitar uma solicitação pendente |

A requisição deve informar:

- a versão da solicitação consultada pelo cliente;
- uma justificativa para a rejeição.

A justificativa deve possuir entre 10 e 200 caracteres. A identidade do
responsável pela decisão não deve ser recebida no corpo da requisição; ela deve
ser obtida da autenticação já existente.

## Regras da operação

- Somente um usuário com papel `gestor` pode rejeitar uma solicitação.
- Apenas solicitações com status `pendente` podem ser rejeitadas.
- Uma solicitação aprovada ou rejeitada não pode receber outra decisão.
- A versão informada deve corresponder à versão atual do registro.
- Uma rejeição aceita deve alterar o status e incrementar a versão.
- A justificativa deve fazer parte do registro de auditoria da decisão.
- A mudança de estado e a auditoria devem ser confirmadas juntas.
- Se uma delas falhar, nenhuma alteração deve permanecer no banco.
- Senhas, hashes, tokens e segredos não podem aparecer na auditoria.

## Banco de dados

A evolução do modelo deve ser realizada por uma nova migration. Não altere
uma migration que já tenha sido aplicada e não reative a sincronização
automática do TypeORM.

O banco deve aceitar somente os estados previstos pela aplicação. A migration
também deve possuir uma forma coerente de desfazer sua alteração.

Atualize o seed com uma solicitação pendente que possa ser usada nos testes de
rejeição. Executar o seed mais de uma vez não deve duplicar esse registro.

## Comportamentos esperados

| Situação | Resultado esperado |
|---|---|
| rejeição válida por gestor | sucesso, status `rejeitada` e versão incrementada |
| justificativa muito curta ou muito longa | `400 Bad Request` |
| requisição sem token ou com token inválido | `401 Unauthorized` |
| rejeição por solicitante ou auditor | `403 Forbidden` |
| id inexistente | `404 Not Found` |
| solicitação que não está pendente | `409 Conflict` |
| versão diferente da versão atual | `409 Conflict` |
| falha durante o registro da decisão | nenhuma alteração persistida |
| seed executado novamente | nenhum registro duplicado |

## Verificação

Ao terminar, confirme que:

- a migration funciona em um banco vazio;
- o seed pode ser repetido;
- a nova rota está protegida;
- entradas inválidas não alteram o banco;
- a primeira decisão com a versão atual pode ser concluída;
- uma segunda decisão baseada na versão antiga é rejeitada;
- uma rejeição concluída possui a auditoria correspondente;
- uma falha intermediária não deixa apenas parte da operação gravada;
- as rotas de criação, consulta e aprovação continuam funcionando.

A atividade deve ser resolvida com os recursos estudados nos encontros 07 a
10. Escolha a organização do código e os recursos do TypeORM que considerar
adequados, desde que todos os comportamentos descritos sejam atendidos.
