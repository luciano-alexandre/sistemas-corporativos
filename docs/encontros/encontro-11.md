# Encontro 11 — Atividade prática de revisão

## Tema

Persistência, migrations, transações, auditoria e concorrência.

## O que vamos revisar

- Modelagem de uma entidade persistida no PostgreSQL.
- Validação de entrada com DTOs.
- Consultas e alterações com repositório TypeORM.
- Evolução do schema por migration.
- Dados iniciais preparados por seed.
- Escritas atômicas com transação.
- Registro de auditoria com o ator obtido do JWT.
- Detecção de versão antiga com resposta `409 Conflict`.

## Antes de começar

Continue no projeto construído entre os encontros 07 e 10. Confirme que:

- API e PostgreSQL iniciam pelo Docker Compose;
- `synchronize` está desativado;
- migrations e seed podem ser executados;
- existem solicitações pendentes para teste;
- login, JWT e papéis continuam funcionando;
- a tabela de auditoria existe.

## A atividade

A equipe de compras precisa rejeitar solicitações que não possuam
justificativa suficiente ou que estejam associadas ao centro de custo errado.
Implemente uma nova transição:

```http
PATCH /solicitacoes/:id/rejeitar
```

A operação deve mudar uma solicitação `pendente` para `rejeitada` e registrar
a decisão na auditoria. As duas escritas devem ser confirmadas ou desfeitas
juntas.

## Alterar o modelo

Amplie o tipo do status:

```ts
export type StatusSolicitacao =
  | 'pendente'
  | 'aprovada'
  | 'rejeitada';
```

Crie uma migration para representar a mudança no schema. Se `status` estiver
armazenado como `varchar` sem uma restrição de valores, acrescente uma
`CHECK CONSTRAINT` que aceite somente os três estados.

Exemplo de SQL que pode aparecer na migration:

```sql
ALTER TABLE solicitacoes
ADD CONSTRAINT chk_solicitacoes_status
CHECK (status IN ('pendente', 'aprovada', 'rejeitada'));
```

O método `down` deve remover essa restrição.

> Se uma migration anterior já criou uma restrição para `status`, altere-a em
> uma nova migration. Não edite uma migration que já tenha sido aplicada.

## Criar o DTO

Crie `src/solicitacoes/dto/rejeitar-solicitacao.dto.ts`:

```ts
import { IsInt, IsString, MaxLength, Min, MinLength } from 'class-validator';

export class RejeitarSolicitacaoDto {
  @IsInt()
  @Min(1)
  versao: number;

  @IsString()
  @MinLength(10)
  @MaxLength(200)
  justificativa: string;
}
```

A justificativa vem do corpo. O ator não: ele deve vir do JWT validado pelo
servidor.

## Implementar a transação

Acrescente ao `SolicitacoesService` um método com esta assinatura:

```ts
rejeitar(
  id: number,
  versaoEsperada: number,
  justificativa: string,
  atorId: number,
)
```

Dentro de `dataSource.transaction`:

1. localize a solicitação;
2. responda `404` se ela não existir;
3. responda `409` se ela não estiver pendente;
4. execute um `UPDATE` condicionado por id, status e versão;
5. responda `409` se nenhuma linha for alterada;
6. insira a auditoria `SOLICITACAO_REJEITADA`;
7. devolva a solicitação atualizada.

O trecho central pode seguir esta estrutura:

```ts
const resultado = await manager
  .createQueryBuilder()
  .update(Solicitacao)
  .set({
    status: 'rejeitada',
    versao: () => 'versao + 1',
  })
  .where('id = :id', { id })
  .andWhere('status = :status', { status: 'pendente' })
  .andWhere('versao = :versao', { versao: versaoEsperada })
  .execute();

if (resultado.affected !== 1) {
  throw new ConflictException(
    'A solicitação foi alterada; consulte novamente',
  );
}

await manager.insert(Auditoria, {
  atorId,
  acao: 'SOLICITACAO_REJEITADA',
  recursoTipo: 'solicitacao',
  recursoId: id,
  detalhes: {
    statusAnterior: 'pendente',
    statusAtual: 'rejeitada',
    justificativa,
    versaoAnterior: versaoEsperada,
  },
});
```

Use o `manager` transacional nas duas escritas. Não use o repositório injetado
para uma operação que pertença à mesma transação.

## Criar a rota

No controller, mantenha a mesma proteção usada na aprovação:

```ts
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles('gestor')
@Patch(':id/rejeitar')
rejeitar(
  @Param('id', ParseIntPipe) id: number,
  @Body() dto: RejeitarSolicitacaoDto,
  @Req() request: RequisicaoAutenticada,
) {
  return this.service.rejeitar(
    id,
    dto.versao,
    dto.justificativa,
    request.user.id,
  );
}
```

Combine os imports com os existentes. Não remova as rotas de criação,
consulta, aprovação ou relatório.

## Atualizar o seed

Acrescente ao seed uma solicitação pendente destinada ao teste de rejeição:

```ts
{
  titulo: 'Compra sem centro de custo confirmado',
  centroCusto: 'COMPRAS',
  prioridade: 'normal' as const,
}
```

Execute o seed duas vezes e confirme que o registro não foi duplicado.

## Testes

| Caso | Resultado esperado |
|---|---|
| rejeição válida por gestor | `200`, status `rejeitada` |
| justificativa com menos de 10 caracteres | `400`, sem escrita |
| rota sem token | `401`, sem escrita |
| rejeição por solicitante ou auditor | `403`, sem escrita |
| id inexistente | `404`, sem auditoria |
| solicitação já aprovada ou rejeitada | `409`, sem nova auditoria |
| versão antiga | `409`, estado preservado |
| falha simulada antes da auditoria | rollback para `pendente` |
| seed executado duas vezes | nenhum registro duplicado |

## Conferir no PostgreSQL

```bash
docker compose exec db psql -U app -d solicitacoes -c "SELECT id, titulo, status, versao FROM solicitacoes ORDER BY id;"
docker compose exec db psql -U app -d solicitacoes -c "SELECT ator_id, acao, recurso_id, detalhes FROM auditorias ORDER BY id;"
```

Para uma rejeição confirmada, deve existir uma solicitação com status
`rejeitada` e uma auditoria com o mesmo recurso. Credenciais rejeitadas, papel
incorreto, versão antiga ou rollback não devem produzir uma auditoria de sucesso.

## Dicas

- Gere uma nova migration; não volte a usar `synchronize`.
- Obtenha o ator de `request.user`, nunca do corpo.
- Valide a justificativa no DTO.
- Inclua a versão na condição do `UPDATE`.
- Mantenha alteração e auditoria no mesmo manager transacional.
- Use `409` para conflito com o estado atual.

## Confira o resultado

- A migration funciona em um banco vazio.
- O seed pode ser repetido sem duplicar registros.
- A rota exige JWT e papel `gestor`.
- A rejeição atualiza status e versão.
- A auditoria registra ator, ação, recurso, instante e justificativa.
- Uma versão antiga não sobrescreve a decisão atual.
- Uma falha entre as escritas provoca rollback.

Esta atividade reúne os conhecimentos dos encontros 07 a 10 e prepara o
projeto para a Prática 2 do encontro 12.
