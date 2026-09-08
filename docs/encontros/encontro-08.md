# Encontro 08 — Atividade prática de persistência

## Tema

Modelagem relacional e consultas com PostgreSQL, TypeORM e NestJS.

## O que vamos revisar

- Fixar o conteúdo desenvolvido no encontro 07.
- Alterar o modelo sem retornar ao armazenamento em memória.
- Aplicar validação no contrato HTTP e no modelo persistente.
- Usar o repositório TypeORM para inserir, filtrar e consultar registros.
- Preservar autenticação e autorização anteriores.
- Verificar a persistência depois do reinício da API.
- Relembrar as responsabilidades de DTO, entidade, repositório e service.

## Antes de começar

Continue no projeto do encontro 07 e confirme que:

- `docker compose up --build` inicia API e PostgreSQL;
- o login devolve um JWT;
- `POST /solicitacoes` grava no banco;
- `GET /solicitacoes` apresenta os registros;
- a aprovação continua restrita ao papel `gestor`.

Nesta atividade, `synchronize: true` permanece apenas no ambiente didático.
Migrations e evolução segura do schema serão estudadas no encontro 09.

## A atividade

A equipe de compras passou a receber muitas solicitações. Apenas título e
status não bastam para organizar o trabalho. Cada solicitação deve informar o
centro de custo e sua prioridade. A listagem também deve aceitar filtros sem
carregar todos os registros para um array na aplicação.

### Ampliar o modelo

Acrescente à entidade `Solicitacao`:

| Campo | Tipo TypeScript | Mapeamento | Regra |
|---|---|---|---|
| `centroCusto` | `string` | `varchar(30)` | obrigatório |
| `prioridade` | `'normal' \| 'urgente'` | `varchar(10)` | padrão `normal` |

Use os nomes `centro_custo` e `prioridade` no PostgreSQL. Preserve id, título,
status, versão e datas. Se o exercício do encontro 07 já acrescentou
`centroCusto`, revise o mapeamento e implemente somente o que falta.

### Validar a criação

Atualize `CriarSolicitacaoDto` para aceitar:

```json
{
  "titulo": "Aquisição de teclado ergonômico",
  "centroCusto": "TI-DEV",
  "prioridade": "urgente"
}
```

Regras:

- `titulo`: texto entre 5 e 150 caracteres;
- `centroCusto`: texto entre 2 e 30 caracteres;
- `prioridade`: somente `normal` ou `urgente`;
- campos adicionais devem ser rejeitados pelo `ValidationPipe` global.

O service deve copiar somente os campos validados para a entidade e salvá-la
pelo repositório. Não use o corpo completo nem restaure o array em memória.

### Filtrar no banco

Adapte a rota:

```http
GET /solicitacoes?status=pendente&centroCusto=TI-DEV&prioridade=urgente
```

Todos os filtros são opcionais e combináveis. Crie
`FiltrarSolicitacoesDto` com:

- `status`: `pendente` ou `aprovada`;
- `centroCusto`: texto com no máximo 30 caracteres;
- `prioridade`: `normal` ou `urgente`.

Receba o DTO com `@Query()`. No service, entregue a condição a
`repository.find`; não use `Array.filter` depois da consulta.

```ts
return this.repository.find({
  where: {
    ...(filtros.status && { status: filtros.status }),
    ...(filtros.centroCusto && { centroCusto: filtros.centroCusto }),
    ...(filtros.prioridade && { prioridade: filtros.prioridade }),
  },
  order: { id: 'ASC' },
});
```

O trecho orienta a consulta; ajuste tipos e imports ao seu projeto.

### Manter o que já funciona

- criação, listagem e consulta por id continuam exigindo JWT;
- aprovação continua exclusiva do papel `gestor`;
- nenhuma resposta apresenta senha, hash, segredo ou token;
- id inexistente continua produzindo `404 Not Found`;
- entrada inválida produz `400 Bad Request` sem gravar dados.

### Confirmar a persistência

Depois de criar os registros, reinicie somente a API:

```bash
docker compose stop api
docker compose start api
```

Os dados e os novos campos devem continuar disponíveis.

## Passos

1. Teste a aplicação antes das alterações.
2. Modifique `solicitacao.entity.ts`.
3. Atualize `CriarSolicitacaoDto`.
4. Crie `FiltrarSolicitacoesDto`.
5. Adapte `criar` e `listar` no service.
6. Receba `@Query()` no controller.
7. Reinicie o ambiente e observe o schema.
8. Cadastre os registros de teste.
9. Execute a matriz e inspecione o banco.

## Dados para os testes

| Título | Centro de custo | Prioridade |
|---|---|---|
| Aquisição de monitor | `TI-DEV` | `normal` |
| Substituição de servidor | `TI-INFRA` | `urgente` |
| Licença de ferramenta de testes | `TI-DEV` | `urgente` |

Crie os registros pela API, pois isso também verifica DTO, controller,
service e repositório.

## Testes

| Caso | Requisição | Resultado esperado |
|---:|---|---|
| 1 | criação válida com JWT | `201`, campos persistidos |
| 2 | criação sem `centroCusto` | `400`, nenhuma inserção |
| 3 | criação com prioridade `alta` | `400`, nenhuma inserção |
| 4 | criação com campo desconhecido | `400`, nenhuma inserção |
| 5 | listagem sem filtros | `200`, registros cadastrados |
| 6 | filtro `centroCusto=TI-DEV` | `200`, dois registros novos |
| 7 | filtro `prioridade=urgente` | `200`, dois registros novos |
| 8 | filtros `TI-DEV` e `urgente` | `200`, um registro novo |
| 9 | listagem sem token | `401` |
| 10 | consulta depois de reiniciar a API | dados preservados |

Registros anteriores no volume podem aumentar os totais. Nesse caso, valide os
três registros pelos dados apresentados acima.

## Inspecionar o banco

```bash
docker compose exec db psql -U app -d solicitacoes -c "\d solicitacoes"
docker compose exec db psql -U app -d solicitacoes -c "SELECT id, titulo, centro_custo, prioridade, status FROM solicitacoes ORDER BY id;"
```

Compare o SQL com a resposta da API. SQL é evidência da persistência, mas os
clientes do sistema devem passar pelas regras da API.

## Dicas

- Faça os filtros no banco, e não com `Array.filter`.
- Passe para `save` apenas os campos validados.
- Preserve os guards ao alterar o controller.
- Valide também os filtros opcionais.
- Para testar a persistência, reinicie apenas a API. O comando
  `docker compose down --volumes` apagaria o banco local.

## Confira o resultado

- Ampliei a entidade sem recriar o projeto.
- Atualizei os DTOs e rejeitei valores inválidos.
- Salvei somente campos permitidos.
- Executei filtros pelo repositório.
- Preservei autenticação e autorização.
- Testei todos os cenários da matriz.
- Confirmei os registros com `psql`.
- Reiniciei a API e verifiquei a persistência.
- Não coloquei tokens ou segredos nos arquivos do projeto.
