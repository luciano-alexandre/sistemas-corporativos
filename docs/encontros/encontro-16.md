# Encontro 16 — Integração do projeto com MongoDB

## Tema

Adicionar MongoDB à API NestJS desenvolvida na disciplina.

## Objetivos

- Configurar o MongoDB junto aos serviços existentes.
- Conectar a aplicação NestJS usando Mongoose.
- Criar schema, DTO, service, controller e módulo.
- Associar documentos a solicitações existentes no PostgreSQL.
- Proteger as novas rotas com JWT.
- Criar índices e validar os dados gravados.
- Testar a integração e os comportamentos de erro.

## Funcionalidade

A API passará a registrar anotações de acompanhamento das solicitações. Uma
anotação conterá texto, marcadores e dados adicionais que podem variar conforme
o tipo de solicitação.

As solicitações, aprovações, centros de custo e reservas continuam no
PostgreSQL. O MongoDB será usado somente para as anotações. A auditoria da
aprovação criada nos encontros anteriores também deve ser preservada.

Endpoints:

```http
POST /solicitacoes/:id/anotacoes
GET /solicitacoes/:id/anotacoes
```

## Passo 1 — Adicionar o MongoDB ao ambiente

No `compose.yaml` do projeto, acrescente:

```yaml
services:
  mongo:
    image: mongo:8
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_USER}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db

volumes:
  mongo_data:
```

Combine esse trecho com os serviços e volumes já existentes. No `.env` local:

```dotenv
MONGO_USER=app
MONGO_PASSWORD=app
MONGO_DATABASE=solicitacoes
MONGODB_URI=mongodb://app:app@mongo:27017/solicitacoes?authSource=admin
```

No `.env.example`, mantenha apenas valores de exemplo. O arquivo `.env`
continua ignorado pelo Git.

Inicie e confira:

```bash
docker compose up -d mongo
docker compose ps
docker compose logs mongo
```

## Passo 2 — Instalar a integração do NestJS

```bash
npm install @nestjs/mongoose mongoose
```

Se a aplicação é executada apenas pelo contêiner, execute o comando no serviço
da API conforme a configuração do projeto.

## Passo 3 — Configurar a conexão

No `AppModule`, acrescente o módulo do Mongoose:

```ts
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { MongooseModule } from '@nestjs/mongoose';
import { AnotacoesModule } from './anotacoes/anotacoes.module';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
    }),
    MongooseModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        uri: config.getOrThrow<string>('MONGODB_URI'),
      }),
    }),
    AnotacoesModule,
  ],
})
export class AppModule {}
```

Combine o exemplo com os módulos existentes. Não remova a configuração do
TypeORM nem os módulos de autenticação e solicitações.

## Passo 4 — Criar o schema

Crie `src/anotacoes/anotacao.schema.ts`:

```ts
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { HydratedDocument, SchemaTypes } from 'mongoose';

export type AnotacaoDocument = HydratedDocument<Anotacao>;

@Schema({
  collection: 'anotacoes',
  timestamps: {
    createdAt: 'criadaEm',
    updatedAt: 'atualizadaEm',
  },
})
export class Anotacao {
  @Prop({ required: true, index: true })
  solicitacaoId: number;

  @Prop({ required: true })
  autorId: number;

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

  criadaEm: Date;
  atualizadaEm: Date;
}

export const AnotacaoSchema =
  SchemaFactory.createForClass(Anotacao);

AnotacaoSchema.index({
  solicitacaoId: 1,
  criadaEm: -1,
});
```

`dadosAdicionais` permite guardar informações específicas sem alterar o
schema a cada variação. Ele não deve receber senhas, tokens ou dados sem
necessidade definida.

## Passo 5 — Criar o DTO

Crie `src/anotacoes/dto/criar-anotacao.dto.ts`:

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

O identificador da solicitação vem da URL e o autor vem do JWT. Nenhum dos dois
deve ser aceito livremente no corpo.

## Passo 6 — Implementar o service

Crie `src/anotacoes/anotacoes.service.ts`:

```ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { InjectRepository } from '@nestjs/typeorm';
import { Model } from 'mongoose';
import { Repository } from 'typeorm';
import { Solicitacao } from '../solicitacoes/solicitacao.entity';
import {
  Anotacao,
  AnotacaoDocument,
} from './anotacao.schema';
import { CriarAnotacaoDto } from './dto/criar-anotacao.dto';

@Injectable()
export class AnotacoesService {
  constructor(
    @InjectModel(Anotacao.name)
    private readonly anotacoes: Model<AnotacaoDocument>,
    @InjectRepository(Solicitacao)
    private readonly solicitacoes: Repository<Solicitacao>,
  ) {}

  private async confirmarSolicitacao(id: number) {
    const existe = await this.solicitacoes.exists({
      where: { id },
    });

    if (!existe) {
      throw new NotFoundException('Solicitação não encontrada');
    }
  }

  async criar(
    solicitacaoId: number,
    autorId: number,
    dto: CriarAnotacaoDto,
  ) {
    await this.confirmarSolicitacao(solicitacaoId);

    return this.anotacoes.create({
      solicitacaoId,
      autorId,
      texto: dto.texto,
      marcadores: dto.marcadores ?? [],
      dadosAdicionais: dto.dadosAdicionais ?? {},
    });
  }

  async listar(solicitacaoId: number) {
    await this.confirmarSolicitacao(solicitacaoId);

    return this.anotacoes
      .find({ solicitacaoId })
      .sort({ criadaEm: -1 })
      .lean()
      .exec();
  }
}
```

A existência da solicitação é conferida antes da gravação. O MongoDB não cria
uma chave estrangeira para o registro do PostgreSQL, portanto essa regra precisa
ser aplicada pela API.

## Passo 7 — Criar o controller

Crie `src/anotacoes/anotacoes.controller.ts`:

```ts
import {
  Body,
  Controller,
  Get,
  Param,
  ParseIntPipe,
  Post,
  Req,
  UseGuards,
} from '@nestjs/common';
import { JwtAuthGuard } from '../auth/jwt-auth.guard';
import { AnotacoesService } from './anotacoes.service';
import { CriarAnotacaoDto } from './dto/criar-anotacao.dto';

type RequisicaoAutenticada = {
  user: {
    id: number;
    papel: string;
  };
};

@UseGuards(JwtAuthGuard)
@Controller('solicitacoes/:solicitacaoId/anotacoes')
export class AnotacoesController {
  constructor(private readonly service: AnotacoesService) {}

  @Post()
  criar(
    @Param('solicitacaoId', ParseIntPipe) solicitacaoId: number,
    @Body() dto: CriarAnotacaoDto,
    @Req() request: RequisicaoAutenticada,
  ) {
    return this.service.criar(
      solicitacaoId,
      request.user.id,
      dto,
    );
  }

  @Get()
  listar(
    @Param('solicitacaoId', ParseIntPipe) solicitacaoId: number,
  ) {
    return this.service.listar(solicitacaoId);
  }
}
```

## Passo 8 — Criar o módulo

Crie `src/anotacoes/anotacoes.module.ts`:

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
    TypeOrmModule.forFeature([Solicitacao]),
  ],
  controllers: [AnotacoesController],
  providers: [AnotacoesService],
})
export class AnotacoesModule {}
```

Confirme que `AnotacoesModule` foi importado no `AppModule`.

## Passo 9 — Iniciar e conferir as conexões

```bash
docker compose up --build
docker compose logs api
```

A API deve iniciar somente depois de conseguir carregar suas configurações. Se
a conexão falhar, confira:

- nome do serviço `mongo`;
- usuário, senha e `authSource`;
- nome das variáveis;
- rede do Docker Compose;
- disponibilidade do contêiner.

Não use `localhost` na URI quando a API e o MongoDB estão em contêineres
diferentes do mesmo Compose; use o nome do serviço.

## Passo 10 — Testar a criação

Use uma solicitação existente:

```http
POST /solicitacoes/1/anotacoes
Authorization: Bearer TOKEN_VALIDO
Content-Type: application/json

{
  "texto": "Fornecedor confirmou a disponibilidade do equipamento.",
  "marcadores": ["fornecedor", "acompanhamento"],
  "dadosAdicionais": {
    "previsaoDias": 7,
    "canal": "email"
  }
}
```

O resultado deve conter o identificador gerado pelo MongoDB, o ID da
solicitação, o autor obtido do token e as datas.

## Passo 11 — Testar a listagem

```http
GET /solicitacoes/1/anotacoes
Authorization: Bearer TOKEN_VALIDO
```

As anotações devem aparecer da mais recente para a mais antiga.

Confira diretamente no MongoDB:

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

## Passo 12 — Testar erros e segurança

Execute:

| Caso | Resultado esperado |
|---|---|
| criar sem token | `401 Unauthorized` |
| listar sem token | `401 Unauthorized` |
| texto vazio ou longo demais | `400 Bad Request` |
| mais de dez marcadores | `400 Bad Request` |
| solicitação inexistente | `404 Not Found` |
| criação válida | documento persistido |
| listagem válida | documentos ordenados |

Confirme que o cliente não consegue escolher `autorId`, `solicitacaoId` ou
datas por meio do corpo.

## Passo 13 — Observar limites da integração

A consulta ao PostgreSQL e a gravação no MongoDB não participam da mesma
transação. Neste caso, a anotação só é criada depois que a existência da
solicitação é confirmada, mas a solicitação ainda poderia ser removida entre as
duas operações.

Para esta funcionalidade de acompanhamento, aceite essa limitação e impeça a
remoção física de solicitações, mantendo seu histórico. Processos críticos,
como aprovação e reserva de saldo, continuam inteiramente no PostgreSQL.

Também evite copiar para a anotação informações que mudam frequentemente. O
documento guarda o `solicitacaoId`; os demais dados atuais continuam sendo
consultados em sua fonte original.

## Exercício

Acrescente um filtro opcional à listagem:

```http
GET /solicitacoes/:id/anotacoes?marcador=fornecedor
```

O filtro deve:

1. ser validado por um DTO de query;
2. limitar o marcador a 30 caracteres;
3. executar a seleção no MongoDB;
4. preservar a ordenação por data;
5. retornar uma lista vazia quando não houver correspondência;
6. continuar retornando `404` quando a solicitação não existir.

Teste também uma anotação com `dadosAdicionais` diferentes dos exemplos para
observar a flexibilidade do documento.

## Síntese

A integração acrescentou MongoDB ao ambiente, configurou a conexão pelo
NestJS, criou documentos validados e relacionou as anotações às solicitações.
O PostgreSQL continua responsável pelas operações transacionais já existentes,
enquanto o novo módulo atende uma funcionalidade com dados flexíveis e
consultas simples por solicitação.
