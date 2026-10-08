# Encontro 17 — Integração do projeto com MongoDB

## Objetivo

Adicionar MongoDB à API NestJS, entendendo cada arquivo, decorator, comando e
decisão. A API registrará anotações de acompanhamento:

```text
POST /solicitacoes/:solicitacaoId/anotacoes
GET  /solicitacoes/:solicitacaoId/anotacoes
GET  /solicitacoes/:solicitacaoId/anotacoes?marcador=fornecedor
```

Solicitações, aprovações e orçamento continuam no PostgreSQL. O MongoDB
armazenará anotações com texto, marcadores e dados adicionais flexíveis.

```mermaid
flowchart LR
    C["Cliente"] --> N["API NestJS"]
    N --> P[("PostgreSQL")]
    N --> M[("MongoDB")]
    P --> S["solicitações e orçamento"]
    M --> A["anotações"]
```

## Arquivos que serão adicionados

```text
src/anotacoes/
├── dto/
│   ├── criar-anotacao.dto.ts
│   └── filtrar-anotacoes.dto.ts
├── anotacao.schema.ts
├── anotacoes.controller.ts
├── anotacoes.module.ts
└── anotacoes.service.ts
```

| Arquivo | Responsabilidade |
|---|---|
| schema | definir o documento persistido |
| DTOs | validar corpo e query string |
| service | aplicar regras e acessar os bancos |
| controller | expor endpoints HTTP |
| module | reunir as dependências |

## Passo 0 — Evitar arquivos criados como root

Neste cenário, o comando `docker` já funciona sem `sudo`. Isso significa
que o usuário do computador possui acesso ao daemon Docker, mas não determina
qual usuário será usado **dentro do contêiner**.

Se a imagem ou o serviço iniciar seus processos como `root`, comandos como
`npm install`, geração de migrations e criação de arquivos continuam sendo
executados pelo UID `0`. Quando a pasta do projeto está montada no contêiner,
esses arquivos aparecem no computador como pertencentes ao `root`.

```mermaid
flowchart LR
    U["Usuário do computador<br/>UID 1000"] --> D["Docker sem sudo"]
    D --> C["Processo no contêiner<br/>UID 0 — root"]
    C --> F["Arquivo no volume<br/>pertence ao root"]
```

A solução é iniciar o serviço da API com o mesmo UID e GID do usuário atual.
Não é necessário alterar novamente o grupo `docker`.

### 0.1 — Confirmar o problema

Execute no terminal do computador:

```bash
docker compose run --rm api id
```

`docker compose run` cria um contêiner temporário usando a configuração do
serviço `api`. `--rm` remove esse contêiner após a execução e `id` mostra
a identidade usada internamente.

Se a saída apresentar `uid=0(root)`, o processo do contêiner está criando os
arquivos como `root`, mesmo que o comando Docker tenha sido iniciado sem
`sudo`.

Agora confira a identidade do usuário no computador:

```bash
id -u
id -g
```

- `id -u` mostra o UID do usuário atual;
- `id -g` mostra o GID do grupo principal.

Os valores frequentemente são `1000`, mas devem ser consultados em cada
computador.

### 0.2 — Registrar UID e GID

**Arquivo:** `.env`

Acrescente os valores obtidos por `id -u` e `id -g`:

```dotenv
LOCAL_UID=1000
LOCAL_GID=1000
```

Substitua os valores pelos resultados de `id -u` e `id -g`. O Docker
Compose usará essa identidade ao iniciar o processo da API.

**Arquivo:** `.env.example`

Documente as variáveis:

```dotenv
LOCAL_UID=1000
LOCAL_GID=1000
```

Esses números não são senhas, mas podem ser diferentes em cada computador.

### 0.3 — Configurar o serviço da API

**Arquivo:** `compose.yaml`

Dentro do serviço `api` já existente, acrescente `user` e as variáveis de
ambiente relacionadas ao diretório temporário:

```yaml
services:
  api:
    user: "${LOCAL_UID:-1000}:${LOCAL_GID:-1000}"
    environment:
      HOME: /tmp
      npm_config_cache: /tmp/.npm
```

Esse trecho deve ser combinado com as propriedades já existentes no serviço
`api`; não substitua `build`, `ports`, `volumes`, `depends_on` ou
outras variáveis.

- `user` define o UID e o GID do processo dentro do contêiner;
- `:-1000` fornece o valor padrão quando a variável não está definida;
- `HOME: /tmp` fornece um diretório gravável ao processo;
- `npm_config_cache: /tmp/.npm` impede que o npm tente gravar cache em uma
  pasta pertencente ao `root`.

Não aplique `user: "${LOCAL_UID}:${LOCAL_GID}"` ao serviço `mongo`. A
imagem oficial prepara permissões e executa o banco com sua própria identidade
interna. O volume do banco deve permanecer sob responsabilidade da imagem.

### 0.4 — Recriar e conferir o serviço

Depois de salvar o Compose, recrie a API e confira a identidade:

```bash
docker compose up -d --build --force-recreate api
docker compose run --rm api id
```

- `--build` reconstrói a imagem quando necessário;
- `--force-recreate` substitui o contêiner mesmo que sua configuração pareça
  não ter mudado;
- o segundo comando confirma o UID e o GID usados internamente.

Os valores apresentados agora devem coincidir com `LOCAL_UID` e
`LOCAL_GID`, e não devem mostrar `uid=0(root)`.

## Passo 1 — Adicionar o MongoDB

**Arquivo:** `compose.yaml`

Dentro de `services`, acrescente:

```yaml
  mongo:
    image: mongo:7.0
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_USER}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db
```

- `mongo` é o nome do serviço e seu hostname na rede;
- `image` fixa a linha 7.0, compatível com o laboratório;
- `environment` envia dados à inicialização;
- `${...}` lê valores do `.env`;
- `ports` publica a porta padrão 27017;
- o volume preserva os arquivos em `/data/db`.

**Arquivo:** `compose.yaml`

Na única seção `volumes`, acrescente:

```yaml
volumes:
  postgres_data:
  mongo_data:
```

Preserve o nome do volume PostgreSQL já existente. Não crie duas seções
`volumes`.

## Passo 2 — Configurar o ambiente

**Arquivo:** `.env`

```dotenv
MONGO_USER=app
MONGO_PASSWORD=app
MONGO_DATABASE=solicitacoes
MONGODB_URI=mongodb://app:app@mongo:27017/solicitacoes?authSource=admin
```

| Trecho da URI | Significado |
|---|---|
| `mongodb://` | protocolo do driver |
| `app:app@` | usuário e senha locais |
| `mongo:27017` | serviço e porta |
| `/solicitacoes` | banco usado |
| `authSource=admin` | local em que o usuário foi criado |

Use `mongo`, não `localhost`: dentro do contêiner da API, `localhost`
aponta para a própria API.

**Arquivo:** `.env.example`

```dotenv
MONGO_USER=usuario_local
MONGO_PASSWORD=senha_local
MONGO_DATABASE=solicitacoes
MONGODB_URI=mongodb://usuario_local:senha_local@mongo:27017/solicitacoes?authSource=admin
```

**Arquivo:** `.gitignore`

```gitignore
.env
```

`.env.example` documenta a configuração; `.env` contém segredos e não deve
ser versionado.

## Passo 3 — Testar o serviço

Execute no diretório de `compose.yaml`:

```bash
docker compose up -d mongo
docker compose ps
docker compose logs mongo
```

`up -d mongo` inicia somente o MongoDB em segundo plano. `ps` mostra o
estado. `logs mongo` ajuda a diagnosticar a inicialização.

```bash
docker compose exec mongo mongosh -u app -p app --authenticationDatabase admin
```

`exec mongo` executa dentro do contêiner; `mongosh` abre o shell; `-u` e
`-p` informam as credenciais; `--authenticationDatabase` informa onde o
usuário foi criado. Saia com `exit`.

## Passo 4 — Instalar as dependências

Execute na raiz do projeto:

```bash
npm install @nestjs/mongoose mongoose
```

`mongoose` modela e consulta documentos. `@nestjs/mongoose` integra essa
biblioteca aos módulos e à injeção de dependências do NestJS. O comando atualiza
`package.json` e `package-lock.json`.

### Como as bibliotecas se relacionam

```mermaid
flowchart LR
    A["Código da aplicação"] --> N["@nestjs/mongoose"]
    N --> MG["Mongoose"]
    MG --> DR["Driver do MongoDB"]
    DR --> DB[("MongoDB")]
```

| Camada | Papel |
|---|---|
| `@nestjs/mongoose` | registra conexão e models no NestJS |
| Mongoose | fornece schema, validação e API de consultas |
| driver MongoDB | comunica-se com o servidor pelo protocolo do banco |
| MongoDB | persiste documentos BSON e executa consultas |

`@nestjs/mongoose` não substitui o Mongoose. Ele permite que conexão e models
participem do sistema de módulos do NestJS.

Se o projeto instala dependências pelo contêiner:

```bash
docker compose run --rm api npm install @nestjs/mongoose mongoose
```

`run --rm` cria um contêiner temporário e o remove após o comando.

## Passo 5 — Iniciar o schema

**Novo arquivo:** `src/anotacoes/anotacao.schema.ts`

```ts
import {
  Prop,
  Schema,
  SchemaFactory,
} from '@nestjs/mongoose';
import {
  HydratedDocument,
  SchemaTypes,
} from 'mongoose';

export type AnotacaoDocument =
  HydratedDocument<Anotacao>;
```

- `@Schema` marca uma classe como base do schema;
- `@Prop` descreve uma propriedade persistida;
- `SchemaFactory` converte a classe em schema;
- `HydratedDocument` combina campos e recursos do Mongoose;
- `SchemaTypes` oferece tipos especiais.

### Classe TypeScript, schema e documento

```mermaid
flowchart LR
    C["Classe Anotacao<br/>tipos no código"] --> F["SchemaFactory"]
    F --> S["Schema Mongoose<br/>regras em execução"]
    S --> D["Documento BSON<br/>persistido no MongoDB"]
```

A classe ajuda o TypeScript durante o desenvolvimento, mas seus tipos deixam de
existir quando o JavaScript é executado. O schema continua ativo em tempo de
execução e pode validar os documentos antes da gravação. O documento BSON é o
resultado efetivamente armazenado.

## Passo 6 — Declarar classe e datas

**Arquivo:** `src/anotacoes/anotacao.schema.ts`

Após o tipo, acrescente:

```ts
@Schema({
  collection: 'anotacoes',
  timestamps: {
    createdAt: 'criadaEm',
    updatedAt: 'atualizadaEm',
  },
})
export class Anotacao {
  criadaEm: Date;
  atualizadaEm: Date;
}
```

`collection` fixa o nome da coleção. `timestamps` faz o Mongoose administrar
as datas, renomeadas para `criadaEm` e `atualizadaEm`. O cliente não envia
essas datas.

O ciclo das datas é:

```mermaid
sequenceDiagram
    participant A as Aplicação
    participant M as Mongoose
    participant B as MongoDB
    A->>M: create(dados)
    M->>M: define criadaEm e atualizadaEm
    M->>B: insert do documento
    A->>M: atualização futura
    M->>M: altera atualizadaEm
    M->>B: update do documento
```

`criadaEm` representa a primeira persistência. `atualizadaEm` muda quando o
documento é atualizado por operações que respeitam os timestamps do Mongoose.

## Passo 7 — Adicionar associação e autor

**Arquivo:** `src/anotacoes/anotacao.schema.ts`

Dentro da classe, antes das datas:

```ts
  @Prop({ required: true })
  solicitacaoId: number;

  @Prop({ required: true })
  autorId: number;
```

`required: true` impede a persistência sem o campo. `solicitacaoId` aponta
para o ID numérico do PostgreSQL e `autorId` virá do JWT.

## Passo 8 — Adicionar conteúdo

**Arquivo:** `src/anotacoes/anotacao.schema.ts`

Ainda dentro da classe:

```ts
  @Prop({
    required: true,
    minlength: 3,
    maxlength: 1000,
  })
  texto: string;

  @Prop({
    type: [String],
    default: [],
  })
  marcadores: string[];

  @Prop({
    type: SchemaTypes.Mixed,
    default: {},
  })
  dadosAdicionais: Record<string, unknown>;
```

`minlength` e `maxlength` limitam o texto persistido. `type: [String]`
declara um array. `default` define valores quando o campo é omitido.
`Mixed` aceita objetos de formatos variados; não deve armazenar segredos ou
dados sem finalidade.

### Quando usar `Mixed`

| Uso | Avaliação |
|---|---|
| pequenos dados complementares que variam por anotação | adequado |
| campo principal usado em muitas consultas | prefira propriedades definidas |
| objeto com regras obrigatórias | crie um subdocumento com schema |
| senhas, tokens ou dados sem finalidade | não armazenar |

O Mongoose não conhece a estrutura interna de `Mixed`. Por isso, não consegue
aplicar às suas propriedades a mesma validação declarada nos outros campos.

## Passo 9 — Gerar schema e índice

**Arquivo:** `src/anotacoes/anotacao.schema.ts`

Depois da classe:

```ts
export const AnotacaoSchema =
  SchemaFactory.createForClass(Anotacao);

AnotacaoSchema.index({
  solicitacaoId: 1,
  criadaEm: -1,
});
```

`createForClass` interpreta os decorators. `index` cria índice composto:
`1` organiza pelo ID em ordem crescente e `-1` pela data em ordem
decrescente. Ele atende a listagem das anotações mais recentes de uma
solicitação.

Visualmente, o índice acompanha a consulta:

```mermaid
flowchart LR
    Q["solicitacaoId = 12<br/>ordenar criadaEm desc."] --> I["Índice composto"]
    I --> A["12 — 10:30"]
    I --> B["12 — 09:15"]
    I --> C["12 — 08:40"]
```

A ordem dos campos importa. O banco localiza primeiro todas as entradas de uma
solicitação e já as encontra na ordem necessária. Um índice apenas em
`criadaEm` não atenderia tão bem ao filtro por solicitação.

## Passo 10 — Criar o DTO do corpo

**Novo arquivo:** `src/anotacoes/dto/criar-anotacao.dto.ts`

```ts
import {
  ArrayMaxSize,
  IsArray,
  IsObject,
  IsOptional,
  IsString,
  MaxLength,
  MinLength,
} from 'class-validator';

export class CriarAnotacaoDto {
  @IsString()
  @MinLength(3)
  @MaxLength(1000)
  texto: string;

  @IsOptional()
  @IsArray()
  @ArrayMaxSize(10)
  @IsString({ each: true })
  @MaxLength(30, { each: true })
  marcadores?: string[];

  @IsOptional()
  @IsObject()
  dadosAdicionais?: Record<string, unknown>;
}
```

`@IsOptional` permite omitir o campo. `@IsArray` exige array.
`@ArrayMaxSize` limita itens. `each: true` valida cada marcador.
`@IsObject` exige objeto. O DTO não aceita autor, solicitação ou datas.

### DTO e schema não são a mesma validação

| Camada | Momento | Protege contra |
|---|---|---|
| DTO | entrada HTTP | corpo malformado ou campo não permitido |
| regra do service | execução do caso de uso | solicitação inexistente |
| schema Mongoose | antes da persistência | documento incompatível |
| MongoDB | armazenamento | regras configuradas diretamente na coleção |

```mermaid
flowchart LR
    R["Requisição"] --> D{"DTO válido?"}
    D -->|não| E400["400"]
    D -->|sim| S["Service"]
    S --> M{"Schema válido?"}
    M -->|não| ER["Erro de persistência"]
    M -->|sim| B[("MongoDB")]
```

Manter validações em camadas evita depender exclusivamente do controller ou do
banco.

## Passo 11 — Criar o DTO da query

**Novo arquivo:** `src/anotacoes/dto/filtrar-anotacoes.dto.ts`

```ts
import {
  IsOptional,
  IsString,
  MaxLength,
} from 'class-validator';

export class FiltrarAnotacoesDto {
  @IsOptional()
  @IsString()
  @MaxLength(30)
  marcador?: string;
}
```

Esse DTO valida `?marcador=fornecedor`: o parâmetro é opcional, mas precisa
ser texto de até 30 caracteres quando enviado.

## Passo 12 — Iniciar o service

**Novo arquivo:** `src/anotacoes/anotacoes.service.ts`

```ts
import {
  Injectable,
  NotFoundException,
} from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { InjectRepository } from '@nestjs/typeorm';
import { FilterQuery, Model } from 'mongoose';
import { Repository } from 'typeorm';
import { Solicitacao } from '../solicitacoes/solicitacao.entity';
import {
  Anotacao,
} from './anotacao.schema';
import { CriarAnotacaoDto } from './dto/criar-anotacao.dto';
import { FiltrarAnotacoesDto } from './dto/filtrar-anotacoes.dto';
```

`@Injectable` permite injeção. `NotFoundException` gera HTTP 404.
`@InjectModel` obtém o model MongoDB. `@InjectRepository` obtém o repositório
PostgreSQL. `FilterQuery` tipa filtros do Mongoose.

## Passo 13 — Injetar os acessos

**Arquivo:** `src/anotacoes/anotacoes.service.ts`

```ts
@Injectable()
export class AnotacoesService {
  constructor(
    @InjectModel(Anotacao.name)
    private readonly anotacoes:
      Model<Anotacao>,

    @InjectRepository(Solicitacao)
    private readonly solicitacoes:
      Repository<Solicitacao>,
  ) {}
}
```

`Anotacao.name` corresponde ao model registrado no módulo. `Model` oferece
`create` e `find`; `Repository` consulta a entidade relacional.

### O que está sendo injetado

```mermaid
flowchart TB
    MM["MongooseModule.forFeature"] --> IM["@InjectModel"]
    IM --> MOD["Model de Anotacao"]
    TM["TypeOrmModule.forFeature"] --> IR["@InjectRepository"]
    IR --> REP["Repository de Solicitacao"]
    MOD --> S["AnotacoesService"]
    REP --> S
```

O service não abre conexões manualmente nem cria os objetos com `new`. O
container de injeção do NestJS fornece instâncias configuradas pelos módulos.
Isso centraliza configuração e facilita testes.

## Passo 14 — Confirmar a solicitação

**Arquivo:** `src/anotacoes/anotacoes.service.ts`

Dentro da classe:

```ts
  private async confirmarSolicitacao(id: number) {
    const existe = await this.solicitacoes.exists({
      where: { id },
    });

    if (!existe) {
      throw new NotFoundException(
        'Solicitação não encontrada',
      );
    }
  }
```

`private` restringe o método ao service. `async` permite aguardar o banco.
`exists` verifica sem carregar toda a entidade. A aplicação faz essa
verificação porque o MongoDB não possui chave estrangeira para o PostgreSQL.

## Passo 15 — Criar a anotação

**Arquivo:** `src/anotacoes/anotacoes.service.ts`

```ts
  async criar(
    solicitacaoId: number,
    autorId: number,
    dto: CriarAnotacaoDto,
  ) {
    await this.confirmarSolicitacao(
      solicitacaoId,
    );

    return this.anotacoes.create({
      solicitacaoId,
      autorId,
      texto: dto.texto,
      marcadores: dto.marcadores ?? [],
      dadosAdicionais:
        dto.dadosAdicionais ?? {},
    });
  }
```

`create` valida e persiste. O operador `??` usa o valor à direita quando o
da esquerda é `null` ou `undefined`.

Ao executar `create`, o fluxo é:

```mermaid
sequenceDiagram
    participant S as Service
    participant P as PostgreSQL
    participant M as Mongoose
    participant B as MongoDB
    S->>P: exists(solicitacaoId)
    P-->>S: true
    S->>M: create(dados)
    M->>M: aplica defaults e valida schema
    M->>B: insere BSON
    B-->>S: documento com _id e datas
```

`create` é conveniente porque reúne construção e gravação. Para apenas
construir um documento sem persistir imediatamente, seria possível instanciar o
model e chamar `save` depois.

## Passo 16 — Listar e filtrar

**Arquivo:** `src/anotacoes/anotacoes.service.ts`

```ts
  async listar(
    solicitacaoId: number,
    filtros: FiltrarAnotacoesDto,
  ) {
    await this.confirmarSolicitacao(
      solicitacaoId,
    );

    const consulta:
      FilterQuery<Anotacao> = {
        solicitacaoId,
      };

    if (filtros.marcador) {
      consulta.marcadores = filtros.marcador;
    }

    return this.anotacoes
      .find(consulta)
      .sort({ criadaEm: -1 })
      .lean()
      .exec();
  }
```

- `find` monta a consulta;
- comparar `marcadores` com texto localiza arrays que contêm esse texto;
- `sort` ordena da mais recente para a mais antiga;
- `lean` retorna objetos simples e reduz custo de leitura;
- `exec` executa a consulta e retorna uma Promise.

### A consulta é construída antes de ser executada

```mermaid
flowchart LR
    F["find<br/>define o filtro"] --> O["sort<br/>define a ordem"]
    O --> L["lean<br/>define o formato"]
    L --> E["exec<br/>envia ao banco"]
    E --> R["Promise com resultados"]
```

`find`, `sort` e `lean` configuram um objeto de consulta. O acesso ao banco
ocorre quando a consulta é aguardada ou quando `exec` é chamado. Usar
`exec` torna esse momento explícito.

Comparação de retorno:

| Sem `lean()` | Com `lean()` |
|---|---|
| documentos Mongoose completos | objetos JavaScript simples |
| possui métodos como `save` | não possui métodos do documento |
| mais recursos e maior custo | adequado para respostas somente de leitura |

## Passo 17 — Criar o controller

**Novo arquivo:** `src/anotacoes/anotacoes.controller.ts`

```ts
import {
  Body,
  Controller,
  Get,
  Param,
  ParseIntPipe,
  Post,
  Query,
  Req,
  UseGuards,
} from '@nestjs/common';
import { JwtAuthGuard } from '../auth/jwt-auth.guard';
import { AnotacoesService } from './anotacoes.service';
import { CriarAnotacaoDto } from './dto/criar-anotacao.dto';
import { FiltrarAnotacoesDto } from './dto/filtrar-anotacoes.dto';

type RequisicaoAutenticada = {
  user: {
    id: number;
    papel: string;
  };
};
```

`@Body` lê o corpo, `@Param` lê a URL, `ParseIntPipe` converte e valida o
ID, `@Query` lê a query string, `@Req` fornece o usuário e `@UseGuards`
aplica autenticação.

## Passo 18 — Adicionar a rota POST

**Arquivo:** `src/anotacoes/anotacoes.controller.ts`

Após os imports e o tipo:

```ts
@UseGuards(JwtAuthGuard)
@Controller(
  'solicitacoes/:solicitacaoId/anotacoes',
)
export class AnotacoesController {
  constructor(
    private readonly service:
      AnotacoesService,
  ) {}

  @Post()
  criar(
    @Param('solicitacaoId', ParseIntPipe)
    solicitacaoId: number,
    @Body() dto: CriarAnotacaoDto,
    @Req() request:
      RequisicaoAutenticada,
  ) {
    return this.service.criar(
      solicitacaoId,
      request.user.id,
      dto,
    );
  }
}
```

`@Controller` define o prefixo. `@Post` responde a POST. O guard na classe
protege todas as rotas. O autor vem do JWT, não do corpo.

## Passo 19 — Adicionar a rota GET

**Arquivo:** `src/anotacoes/anotacoes.controller.ts`

Dentro da classe, depois de `criar`:

```ts
  @Get()
  listar(
    @Param('solicitacaoId', ParseIntPipe)
    solicitacaoId: number,
    @Query() filtros:
      FiltrarAnotacoesDto,
  ) {
    return this.service.listar(
      solicitacaoId,
      filtros,
    );
  }
```

`@Get` cria a rota de leitura. `@Query` entrega os parâmetros ao DTO.

## Passo 20 — Criar o módulo

**Novo arquivo:** `src/anotacoes/anotacoes.module.ts`

```ts
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { TypeOrmModule } from '@nestjs/typeorm';
import { Solicitacao } from '../solicitacoes/solicitacao.entity';
import {
  Anotacao,
  AnotacaoSchema,
} from './anotacao.schema';
import { AnotacoesController } from './anotacoes.controller';
import { AnotacoesService } from './anotacoes.service';

@Module({
  imports: [
    MongooseModule.forFeature([
      {
        name: Anotacao.name,
        schema: AnotacaoSchema,
      },
    ]),
    TypeOrmModule.forFeature([
      Solicitacao,
    ]),
  ],
  controllers: [AnotacoesController],
  providers: [AnotacoesService],
})
export class AnotacoesModule {}
```

`MongooseModule.forFeature` registra o model usado por `@InjectModel`.
`TypeOrmModule.forFeature` disponibiliza o repositório. `controllers`
registra rotas e `providers` registra o service.

### Registro global e registro do recurso

```mermaid
flowchart TB
    FR["forRootAsync<br/>abre conexão"] --> CON["Conexão MongoDB"]
    CON --> FF["forFeature<br/>registra schema"]
    FF --> MODEL["Model Anotacao"]
    MODEL --> SERVICE["AnotacoesService"]
```

`forRootAsync` será usado uma vez na raiz da aplicação. `forFeature` é usado
nos módulos que precisam de models específicos. Registrar a conexão sem
`forFeature` não torna automaticamente o model disponível para injeção.

## Passo 21 — Abrir a conexão e registrar o módulo

**Arquivo:** `src/app.module.ts`

Acrescente aos imports:

```ts
import {
  ConfigModule,
  ConfigService,
} from '@nestjs/config';
import { MongooseModule } from '@nestjs/mongoose';
import { AnotacoesModule } from './anotacoes/anotacoes.module';
```

No array `imports` de `@Module`, acrescente:

```ts
ConfigModule.forRoot({
  isGlobal: true,
}),
MongooseModule.forRootAsync({
  inject: [ConfigService],
  useFactory: (
    config: ConfigService,
  ) => ({
    uri: config.getOrThrow<string>(
      'MONGODB_URI',
    ),
  }),
}),
AnotacoesModule,
```

`ConfigModule.forRoot` carrega o `.env`; `isGlobal` compartilha a
configuração. `forRootAsync` constrói a conexão usando uma função.
`inject` fornece `ConfigService`; `useFactory` devolve as opções;
`getOrThrow` falha claramente se a URI estiver ausente.

Durante a inicialização:

```mermaid
sequenceDiagram
    participant N as NestJS
    participant C as ConfigService
    participant MM as MongooseModule
    participant DB as MongoDB
    N->>C: carregar .env
    MM->>C: getOrThrow(MONGODB_URI)
    C-->>MM: URI
    MM->>DB: abrir conexão
    DB-->>MM: conexão pronta ou erro
```

`forRootAsync` não significa que várias conexões serão abertas. O termo
“async” indica que as opções podem depender de serviços injetados ou operações
assíncronas antes da criação da conexão.

Se `MONGODB_URI` estiver ausente, `getOrThrow` interrompe a inicialização em
vez de deixar a aplicação funcionar parcialmente.

Não duplique `ConfigModule` e não remova TypeORM, autenticação ou solicitações.

## Passo 22 — Conferir o ValidationPipe

**Arquivo:** `src/main.ts`

Confirme que existe:

```ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
  }),
);
```

`whitelist` remove propriedades fora do DTO; `forbidNonWhitelisted` gera
erro para elas; `transform` habilita transformações. Não duplique essa
configuração se ela já existir.

## Passo 23 — Construir e iniciar

```bash
docker compose up --build
docker compose logs -f api
```

`--build` reconstrói a imagem com dependências e código novos. `logs -f`
acompanha as novas linhas do log. Se falhar, verifique URI, credenciais e o
estado do serviço `mongo`.

## Passo 24 — Testar criação

```http
POST /solicitacoes/1/anotacoes
Authorization: Bearer TOKEN_VALIDO
Content-Type: application/json

{
  "texto": "Fornecedor confirmou a disponibilidade.",
  "marcadores": ["fornecedor", "acompanhamento"],
  "dadosAdicionais": {
    "previsaoDias": 7,
    "canal": "email"
  }
}
```

`Authorization: Bearer` envia o JWT. `Content-Type` informa JSON. O ID vem
da URL, o autor do token e o corpo contém apenas campos editáveis.

## Passo 25 — Testar listagem e filtro

```http
GET /solicitacoes/1/anotacoes
Authorization: Bearer TOKEN_VALIDO
```

```http
GET /solicitacoes/1/anotacoes?marcador=fornecedor
Authorization: Bearer TOKEN_VALIDO
```

`?marcador=fornecedor` é a query string transformada em filtro no service.

## Passo 26 — Conferir no MongoDB

```bash
docker compose exec mongo mongosh -u app -p app --authenticationDatabase admin
```

```javascript
use solicitacoes

db.anotacoes.find({
  solicitacaoId: 1
}).sort({
  criadaEm: -1
})

db.anotacoes.getIndexes()
```

`use` seleciona o banco, `find` filtra, `sort` ordena e `getIndexes`
confirma os índices.

Detalhamento:

- `use solicitacoes` altera o banco ativo apenas na sessão do `mongosh`;
- `db.anotacoes` referencia a coleção;
- `find({ solicitacaoId: 1 })` seleciona apenas documentos associados ao ID;
- `sort({ criadaEm: -1 })` solicita ordem decrescente;
- `getIndexes()` retorna o índice padrão `_id_` e os índices adicionais;
- o índice composto deve apresentar as chaves
  `solicitacaoId: 1` e `criadaEm: -1`.

Para observar se a consulta utiliza o índice:

```javascript
db.anotacoes
  .find({ solicitacaoId: 1 })
  .sort({ criadaEm: -1 })
  .explain("executionStats")
```

`explain("executionStats")` mostra o plano executado. Procure:

| Informação | Interpretação |
|---|---|
| `IXSCAN` | leitura por índice |
| `COLLSCAN` | varredura da coleção |
| `nReturned` | documentos devolvidos |
| `totalDocsExamined` | documentos examinados |
| `totalKeysExamined` | entradas de índice examinadas |

Com poucos documentos, o otimizador pode escolher uma varredura. O objetivo é
aprender a interpretar o plano, não forçar o uso do índice.

## Passo 27 — Testar erros

| Caso | Esperado | Responsável |
|---|---|---|
| sem token | `401` | guard |
| ID textual | `400` | `ParseIntPipe` |
| texto curto | `400` | DTO |
| mais de dez marcadores | `400` | DTO |
| campo inesperado | `400` | `ValidationPipe` |
| solicitação inexistente | `404` | service |
| marcador inexistente | `200 []` | consulta válida vazia |

Confirme que `autorId`, `solicitacaoId` e datas enviados no corpo são
recusados.

## Limite da integração

A consulta ao PostgreSQL e a gravação no MongoDB não formam uma única
transação. Para o laboratório, solicitações não devem ser removidas fisicamente
e os processos críticos permanecem no PostgreSQL. Não prometa atomicidade entre
os dois bancos.

## Verificação final

- [ ] serviço e volume MongoDB configurados;
- [ ] `.env` ignorado e `.env.example` documentado;
- [ ] schema e índice criados;
- [ ] DTOs validam corpo e query;
- [ ] service consulta PostgreSQL e MongoDB;
- [ ] ator vem do JWT;
- [ ] módulo registra os dois acessos;
- [ ] conexão está no `AppModule`;
- [ ] POST, GET e filtro funcionam;
- [ ] comportamentos de erro foram testados.

## Síntese

O NestJS abriu a conexão por configuração, o Mongoose definiu schema e
consultas, os DTOs validaram a entrada, o TypeORM confirmou a solicitação e o
JWT identificou o autor. Cada arquivo possui uma responsabilidade explícita,
facilitando compreensão, testes e evolução.
