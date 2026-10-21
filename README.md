# aplicacao_ORM_apiTarefas

# Dia 2 — CRUD de Tarefas com Prisma ORM

API de Tarefas (Fastify + Zod + TypeScript) · Prisma 7 + SQLite

## 1. O que vamos fazer

Na api original as tarefas ficavam guardadas em um array dentro da classe `Database` (`src/db.ts`), então tudo se perdia quando o servidor reiniciava. Hoje vamos trocar esse array por um banco de dados de verdade usando o **Prisma ORM** e implementar o CRUD completo de tarefas:

| Operação | Método e rota | Método do Prisma |
| --- | --- | --- |
| Criar | `POST /api/tarefas` | `create` |
| Listar todas | `GET /api/tarefas` | `findMany` |
| Buscar uma | `GET /api/tarefas/:id` | `findUnique` |
| Atualizar | `PATCH /api/tarefas/:id` | `update` |
| Excluir | `DELETE /api/tarefas/:id` | `delete` |

**O que é um ORM?** Em vez de escrever SQL na mão, descrevemos as tabelas em um arquivo (o *schema*) e o Prisma gera um *client* TypeScript com métodos tipados para consultar o banco. Usaremos o SQLite porque ele guarda tudo em um único arquivo, sem precisar instalar nada.

## 2. Antes de começar

- Node.js 20.19 ou superior (ou 22.12+). Confira com `node -v`.
- Projeto `api-tarefas` do Dia 1 com `npm install` já executado.
- Rode todos os comandos **na raiz do projeto** (a pasta onde está o `package.json`).

> **Atenção à versão.** Desde agosto de 2026, `npm install prisma` instala o **Prisma 8**, uma reescrita com API diferente (versão final prevista para outubro de 2026). Neste tutorial usamos o **Prisma 7.10.0**, a versão estável, com a versão fixada nos comandos. Não use `npx prisma@latest`.

## 3. Passo 1 — Instalar as dependências

```bash
npm install prisma@7.10.0 @types/better-sqlite3 --save-dev
npm install @prisma/client@7.10.0 @prisma/adapter-better-sqlite3 dotenv
```

- `prisma`: a CLI, que cria as migrations e gera o client.
- `@prisma/client`: a biblioteca que usamos no código.
- `@prisma/adapter-better-sqlite3`: o driver que conecta o Prisma ao SQLite.
- `@types/better-sqlite3`: tipos TypeScript desse driver.
- `dotenv`: carrega as variáveis do arquivo `.env` (o Prisma 7 não faz isso sozinho).

## 4. Passo 2 — Inicializar o Prisma

```bash
npx prisma init --datasource-provider sqlite --output ../src/generated/prisma
```

O comando cria três coisas:

- `prisma/schema.prisma`: onde descrevemos as tabelas.
- `.env`: com a linha `DATABASE_URL="file:./dev.db"`, que diz onde fica o arquivo do banco.
- `prisma7.config.ts`: configuração da CLI (a partir da versão 7.10 o arquivo tem esse nome; antes era `prisma.config.ts`).

O `prisma7.config.ts` gerado fica assim, e não precisa ser alterado:

```ts
import "dotenv/config";
import { defineConfig } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: {
    path: "prisma/migrations",
  },
  datasource: {
    url: process.env["DATABASE_URL"],
  },
});
```

O parâmetro `--output ../src/generated/prisma` define onde o Prisma vai gerar o client: dentro de `src/generated/prisma` (o caminho é relativo ao `schema.prisma`).

## 5. Passo 3 — Modelar a tabela de tarefas

Abra `prisma/schema.prisma` e deixe o conteúdo assim:

```prisma
generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}

datasource db {
  provider = "sqlite"
}

model Tarefa {
  id        String   @id @default(uuid())
  titulo    String
  descricao String
  prazo     DateTime
  feito     Boolean  @default(false)
}
```

Cada `model` vira uma tabela, e esse model é o equivalente da interface `Tarefa` que tínhamos no `db.ts`:

- `@id` marca a chave primária.
- `@default(uuid())` gera o id automaticamente, igual ao `randomUUID()` do Dia 1.
- `DateTime` corresponde ao `Date` do JavaScript.
- `@default(false)` faz toda tarefa nascer com `feito = false`.

## 6. Passo 4 — Criar a tabela e gerar o client

```bash
npx prisma migrate dev --name criar-tabela-tarefa
npx prisma generate
```

- `migrate dev` compara o schema com o banco, cria o arquivo SQL em `prisma/migrations/` e executa esse SQL, criando o `dev.db` com a tabela `Tarefa`.
- `generate` cria o client TypeScript em `src/generated/prisma`. No Prisma 7 o `migrate dev` não gera o client sozinho, por isso os dois comandos.

Sempre que alterar o `schema.prisma`, rode os dois de novo.

## 7. Passo 5 — Criar a instância do Prisma Client

Crie o arquivo `src/lib/prisma.ts`:

```ts
import "dotenv/config"
import { PrismaBetterSqlite3 } from "@prisma/adapter-better-sqlite3"
import { PrismaClient } from "../generated/prisma/client.js"

const url = process.env.DATABASE_URL

if (!url) {
    throw new Error("DATABASE_URL não está definida no .env")
}

const adapter = new PrismaBetterSqlite3({ url })
const prisma = new PrismaClient({ adapter })

export { prisma }
```

Criamos **uma única instância** e a exportamos, assim como fizemos com o `db` no Dia 1. Todas as rotas importam esse mesmo `prisma`.

## 8. Passo 6 — Ajustar os schemas do Zod

Substitua o conteúdo de `src/schemas/index.ts`:

```ts
import z from "zod"

export const createTaskBodySchema = z.object({
    titulo:    z.string(),
    descricao: z.string(),
    prazo:     z.coerce.date()
})

export const updateTaskBodySchema = z.object({
    titulo:    z.string(),
    descricao: z.string(),
    prazo:     z.coerce.date(),
    feito:     z.boolean()
}).partial()

export const taskParamsSchema = z.object({
    id: z.string()
})
```

O que mudou:

- `prazo` agora usa `z.coerce.date()`. O JSON não tem tipo data: o cliente envia o prazo como texto (`"2026-12-15T18:00:00.000Z"`), e o antigo `z.date()` rejeitaria qualquer requisição. O `coerce` converte o texto em `Date` antes de validar.
- `updateTaskBodySchema` aceita os quatro campos, mas todos opcionais (`.partial()`), porque na atualização o cliente envia só o que quer mudar. É aqui que entra o `feito`, para marcar a tarefa como concluída.
- `taskParamsSchema` valida o `:id` que vem na URL.

## 9. Passo 7 — Implementar o CRUD no servidor

Substitua o conteúdo de `src/server.ts`:

```ts
import fastify, { type FastifyReply, type FastifyRequest } from "fastify"
import { createTaskBodySchema, taskParamsSchema, updateTaskBodySchema } from "./schemas/index.js"
import { prisma } from "./lib/prisma.js"

const app = fastify()

const PORT = 5000

app.get("/api", (req: FastifyRequest, res: FastifyReply) => {

    return res.status(200).send({
        message: `API de Tarefas Rodando na Porta ${PORT}...`
    })

})

// CREATE: cria uma tarefa
app.post("/api/tarefas", async (req: FastifyRequest, res: FastifyReply) => {

    const { titulo, descricao, prazo } = createTaskBodySchema.parse(req.body)

    const tarefa = await prisma.tarefa.create({
        data: { titulo, descricao, prazo }
    })

    return res.status(201).send(tarefa)

})

// READ: lista todas as tarefas
app.get("/api/tarefas", async (req: FastifyRequest, res: FastifyReply) => {

    const tarefas = await prisma.tarefa.findMany()

    return res.status(200).send(tarefas)

})

// READ: busca uma tarefa pelo id
app.get("/api/tarefas/:id", async (req: FastifyRequest, res: FastifyReply) => {

    const { id } = taskParamsSchema.parse(req.params)

    const tarefa = await prisma.tarefa.findUnique({
        where: { id }
    })

    if (!tarefa) {
        return res.status(404).send({ message: "Tarefa não encontrada" })
    }

    return res.status(200).send(tarefa)

})

// UPDATE: atualiza os campos enviados
app.patch("/api/tarefas/:id", async (req: FastifyRequest, res: FastifyReply) => {

    const { id } = taskParamsSchema.parse(req.params)
    const data = updateTaskBodySchema.parse(req.body)

    const existe = await prisma.tarefa.findUnique({
        where: { id }
    })

    if (!existe) {
        return res.status(404).send({ message: "Tarefa não encontrada" })
    }

    const tarefa = await prisma.tarefa.update({
        where: { id },
        data
    })

    return res.status(200).send(tarefa)

})

// DELETE: exclui uma tarefa
app.delete("/api/tarefas/:id", async (req: FastifyRequest, res: FastifyReply) => {

    const { id } = taskParamsSchema.parse(req.params)

    const existe = await prisma.tarefa.findUnique({
        where: { id }
    })

    if (!existe) {
        return res.status(404).send({ message: "Tarefa não encontrada" })
    }

    await prisma.tarefa.delete({
        where: { id }
    })

    return res.status(204).send()

})

app.listen({
    port: PORT
}, () => {
    console.log(`Servidor Rodando em: http://localhost:${PORT}/api/tarefas`)
})
```

Pontos para explicar em sala:

- **O model `Tarefa` vira `prisma.tarefa`** (com a primeira letra minúscula). Todos os métodos ficam nesse objeto.
- **Os handlers agora são `async`**, porque toda consulta ao banco devolve uma Promise e precisamos do `await`.
- **`create`** recebe os dados em `data` e devolve a tarefa criada, já com `id` e `feito: false`. Respondemos com `201 Created`.
- **`findMany`** sem filtros devolve todas as linhas da tabela.
- **`findUnique`** busca por um campo único (aqui, o `id`) e devolve `null` quando não encontra. Por isso o `if (!tarefa)` com resposta `404`.
- **`update`** recebe o `where` (qual registro) e o `data` (o que mudar). Usamos `PATCH` porque o cliente envia só os campos que quer alterar.
- **`delete`** remove o registro. Respondemos `204 No Content`, sem corpo.
- **Por que consultar antes de `update` e `delete`?** Porque o Prisma lança um erro quando o registro não existe. Verificando antes, devolvemos um `404` claro em vez de um erro `500`.

## 10. Passo 8 — Limpeza

O array em memória não é mais usado:

1. Apague o arquivo `src/db.ts`.
2. Confirme que nenhum arquivo ainda importa `./db.js`.

Depois, acrescente estas linhas ao `.gitignore`, para não versionar o banco local nem o client gerado:

```text
src/generated/
*.db
*.db-journal
```

A pasta `prisma/migrations/` **deve** ser versionada: ela é o histórico do banco. Quem clonar o projeto roda `npm install`, `npx prisma migrate dev` e `npx prisma generate` para ter tudo funcionando.

## 11. Passo 9 — Testar

Inicie o servidor:

```bash
npm run server
```

Com o servidor rodando, teste cada rota (use Insomnia, Postman ou Thunder Client com os mesmos dados, ou o `curl` abaixo).

**Criar** (guarde o `id` da resposta):

```bash
curl -X POST http://localhost:5000/api/tarefas \
  -H "Content-Type: application/json" \
  -d '{"titulo":"Estudar Prisma","descricao":"Fazer o CRUD de tarefas","prazo":"2026-12-15T18:00:00.000Z"}'
```

Resposta esperada (status 201):

```json
{
  "id": "3f1c2b9e-...",
  "titulo": "Estudar Prisma",
  "descricao": "Fazer o CRUD de tarefas",
  "prazo": "2026-12-15T18:00:00.000Z",
  "feito": false
}
```

**Listar todas:**

```bash
curl http://localhost:5000/api/tarefas
```

**Buscar uma** (troque `ID_DA_TAREFA` pelo id real):

```bash
curl http://localhost:5000/api/tarefas/ID_DA_TAREFA
```

**Marcar como feita:**

```bash
curl -X PATCH http://localhost:5000/api/tarefas/ID_DA_TAREFA \
  -H "Content-Type: application/json" \
  -d '{"feito":true}'
```

**Excluir** (resposta 204, sem corpo):

```bash
curl -X DELETE http://localhost:5000/api/tarefas/ID_DA_TAREFA
```

Para provar que agora os dados persistem: crie uma tarefa, pare o servidor (Ctrl+C), inicie de novo e liste. A tarefa continua lá, o que não acontecia com o array do Dia 1.

## 12. Como o projeto fica

```text
api-tarefas/
├── .env
├── .gitignore
├── dev.db
├── package.json
├── prisma7.config.ts
├── tsconfig.json
├── prisma/
│   ├── schema.prisma
│   └── migrations/
│       └── <data>_criar-tabela-tarefa/
│           └── migration.sql
└── src/
    ├── generated/prisma/    (gerado pelo Prisma)
    ├── lib/
    │   └── prisma.ts
    ├── schemas/
    │   └── index.ts
    └── server.ts
```

## 13. Problemas comuns

**`npm install prisma` instalou a versão 8 e os comandos não batem com o tutorial.** Remova com `npm uninstall prisma @prisma/client` e instale de novo com as versões fixadas do Passo 1.

**`Cannot find module './internal/class.js'` ao subir o servidor.** O client gerado usa imports `.js` e o `tsx` pode não resolvê-los. Adicione `importFileExtension = "ts"` ao bloco `generator client` do `schema.prisma` e rode `npx prisma generate`. Se o editor passar a reclamar de imports terminados em `.ts`, adicione `"allowImportingTsExtensions": true` e `"noEmit": true` ao `compilerOptions` do `tsconfig.json`.

**`DATABASE_URL não está definida no .env`.** Confira se o `.env` existe na raiz com `DATABASE_URL="file:./dev.db"` e se o servidor foi iniciado a partir da raiz do projeto.

**O editor sublinha o `data` no `update` com erro de tipos.** O `tsconfig.json` tem `exactOptionalPropertyTypes` ativado, e nesse modo os tipos do Prisma não aceitam um campo opcional com valor `undefined` explícito (que é como o Zod tipa os campos opcionais). Troque para `"exactOptionalPropertyTypes": false`. O servidor roda normalmente mesmo com o erro, pois o `tsx` não checa tipos.

**`Invalid input: expected date` ao criar uma tarefa.** O `prazo` do schema de criação ficou `z.date()`. Troque por `z.coerce.date()`.

## 14. Observação sobre erros de validação

Se o corpo da requisição for inválido (um campo faltando, por exemplo), o `parse` do Zod lança um erro e o Fastify responde com `500`. Isso já era assim no Dia 1 e fica fora do escopo de hoje, que é só o CRUD com Prisma. Tratar esses erros e devolver `400` pode ser um tópico para a próxima aula.
