# Encontro 05 — Prática 1: segurança e controle de acesso

## Informações gerais

- **Valor:** 10 pontos
- **Duração:** 90 minutos
- **Modalidade:** individual
- **Consulta:** permitida aos materiais da disciplina, à documentação oficial e
  às anotações pessoais
- **Entrega:** link do repositório GitHub enviado até o final da aula pelo Google
  Sala de Aula
- **Ponto de partida:** projeto NestJS desenvolvido nos encontros 3 e 4

A consulta não autoriza comunicação ou compartilhamento de solução
entre estudantes durante a atividade.


## Situação-problema

A API de solicitações passou a atender também a equipe de auditoria. A empresa
definiu a seguinte matriz de acesso:

| Operação | Solicitante | Gestor | Auditor |
|---|:---:|:---:|:---:|
| Consultar o próprio perfil | sim | sim | sim |
| Aprovar uma solicitação | não | sim | não |
| Consultar o relatório geral | não | sim | sim |

Sua tarefa é adaptar a API para representar o papel `auditor` e disponibilizar
um relatório protegido, sem enfraquecer a proteção da aprovação já existente.

## Requisitos obrigatórios

### 1. Usuários identificados pelo estudante

Inclua no serviço de usuários duas contas ativas e personalizadas:

1. **Usuário gestor:** use o seu primeiro nome no campo `nome`, o papel `gestor`
   e a sua matrícula como senha;
2. **Usuário auditor:** use o seu último sobrenome no campo `nome`, o papel
   `auditor` e a sua matrícula escrita em ordem inversa como senha.

Exemplo apenas para esclarecer a regra: se a estudante se chama `Maria Silva` e
sua matrícula é `20261234`, as senhas de teste serão `20261234` para a conta
`Maria` e `43216202` para a conta `Silva`. Cada estudante deve usar
exclusivamente o próprio nome, o próprio sobrenome e a própria matrícula.

Defina e informe no `README.md` os e-mails das duas contas. Eles devem ser
distintos e coerentes com os nomes utilizados. Nos dados da aplicação, as duas
senhas devem aparecer **somente como hashes gerados com `bcrypt` e salt**; não é
permitido armazenar a matrícula ou a matrícula invertida em texto puro no campo
de senha. O login deve continuar devolvendo um JWT, e nenhuma resposta da API
pode apresentar a senha ou o hash.

> **Critério obrigatório de identificação:** a atividade não será considerada
> se as contas não seguirem a estrutura acima, com o primeiro nome, o último
> sobrenome e a matrícula do próprio estudante. Também não será considerada uma
> entrega cujos hashes correspondam à matrícula de outro estudante.

### 2. Relatório protegido

Implemente a rota:

```http
GET /solicitacoes/relatorio
```

Ela deve:

- exigir um JWT válido;
- permitir acesso apenas aos papéis `gestor` e `auditor`;
- devolver status `200` e um objeto com, no mínimo, a quantidade total de
  solicitações e a quantidade por status;
- reutilizar o service do módulo de solicitações para calcular os dados.

Exemplo de formato aceito, considerando os dados existentes na aplicação:

```json
{
  "total": 3,
  "porStatus": {
    "pendente": 2,
    "aprovada": 1
  }
}
```

Os números do exemplo não são obrigatórios. Eles devem refletir o estado atual
da aplicação.


### 3. Aprovação restrita

Mantenha ou corrija a rota:

```http
PATCH /solicitacoes/:id/aprovar
```

Ela deve exigir JWT e aceitar exclusivamente o papel `gestor`. Um auditor está
autenticado, mas não pode aprovar uma solicitação.

### 4. Configuração segura

A aplicação deve obter o segredo e o tempo de expiração do JWT por variáveis de
ambiente. Entregue `.env.example` sem segredo real e mantenha `.env` ignorado
pelo Git. Não inclua token, senha em texto puro ou chave secreta nos arquivos
versionados ou nas evidências.

### 5. Tratamento HTTP

A solução deve apresentar os seguintes comportamentos:

- credenciais inválidas no login: `401 Unauthorized`;
- rota protegida sem token ou com token inválido: `401 Unauthorized`;
- usuário autenticado sem o papel exigido: `403 Forbidden`;
- usuário autenticado e autorizado: resposta de sucesso.

## Casos de teste

Execute e registre os oito casos abaixo no Thunder Client, Insomnia, Postman ou
com `curl`.

| Caso | Requisição | Identidade | Resultado esperado |
|---:|---|---|---|
| 1 | `POST /auth/login` com senha incorreta | Auditor (sobrenome) | `401` |
| 2 | `POST /auth/login` com matrícula invertida | Auditor (sobrenome) | `201` e token |
| 3 | `GET /auth/perfil` com token | Auditor (sobrenome) | `200`, papel `auditor`, sem senha/hash |
| 4 | `GET /solicitacoes/relatorio` sem token | — | `401` |
| 5 | `GET /solicitacoes/relatorio` com token de solicitante | Bruno | `403` |
| 6 | `GET /solicitacoes/relatorio` com token de auditor | Auditor (sobrenome) | `200` e contagens |
| 7 | `PATCH /solicitacoes/:id/aprovar` com token de auditor | Auditor (sobrenome) | `403` |
| 8 | `PATCH /solicitacoes/:id/aprovar` com token de gestor | Gestor (primeiro nome) | sucesso |

Se a base do projeto usar outro código de sucesso coerente para a atualização,
registre-o e justifique.

## Entrega pelo GitHub

Crie um repositório no GitHub e envie seu link na atividade correspondente do
Google Sala de Aula até o final da aula. Confirme que o professor possui acesso
ao repositório. A entrega será avaliada a partir do conteúdo disponível no
último commit realizado dentro do prazo; arquivos enviados separadamente ou
somente armazenados na máquina local não substituem o repositório.

O repositório deve conter:

1. código-fonte da aplicação;
2. `package.json`, `package-lock.json`, `Dockerfile` e `compose.yaml`;
3. `.env.example` e `.gitignore`, sem o arquivo `.env`;
4. `README.md` com:
   - nome completo do estudante e número de matrícula;
   - comandos exatos para construir, iniciar, testar e encerrar a aplicação com
     Docker Compose;
   - as duas contas personalizadas, seus e-mails, papéis e a regra de senha
     utilizada: matrícula para o gestor e matrícula invertida para o auditor;
   - fontes consultadas;
   - uma explicação de duas a cinco linhas sobre por que os casos sem token e
     sem papel adequado produzem respostas diferentes;

Não entregue `node_modules`, `dist`, `.env`, tokens JWT ou outros segredos. Antes
do envio, confirme que outra pessoa conseguiria executar a solução apenas com
os arquivos do repositório e as instruções do `README.md`. A aplicação deve ser
construída e executada com Docker; soluções que dependam de Node.js ou npm
instalados diretamente na máquina do avaliador não atendem ao requisito de
execução.

Embora a matrícula conste no `README.md` por exigência de identificação, ela e
sua forma invertida não devem aparecer no código como senhas em texto puro. Os
campos de senha da lista de usuários devem conter apenas os respectivos hashes
`bcrypt`.
