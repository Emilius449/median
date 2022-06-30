# Generate Nest Js Project

```bash
$ nest new median
```

# cd into project

```bash
$ cd median
```

# Configuration

```bash
$ npm i --save @nestjs/config
```

in app.module.ts => imports

```ts
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
  ],
```

# Add Database (I'm using docker and postgres as database)

- Get docker postgres image

- Create Docker file (docker-compose.yml) in root folder

```yml
services:
  dev-db:
    image: postgres
    ports:
      - 5432:5432
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: 123456@Aa
      POSTGRES_DB: medianDatabase
```

- Run docker postgres image

```bash
$ docker compose up -d
```

- Configure Environment variables

```env
DATABASE_URL=""
```

# Install Prisma

```bash
$ npm install prisma --save-dev
```

## Install Prisma Client (optinal)

```bash
$ npm install @prisma/client
```

## Generate Prisma folder

```bash
$ npx prisma init
```

## Declare Prisma Model in schema.prisma file

example

```prisma

// Article model
model Article {
  id          Int      @id @default(autoincrement())
  title       String   @unique
  description String?
  body        String
  published   Boolean  @default(false)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}

```

## Generate Prisma migration

```bash
$ npx prisma migrate dev --name "init"
```

## Seed the database

- Create seed file (seed.ts) in prisma folder

```ts
// prisma/seed.ts
import { PrismaClient } from '@prisma/client';

// initialize Prisma Client
const prisma = new PrismaClient();

async function main() {
  // create two dummy articles
  const post1 = await prisma.article.upsert({
    where: { title: 'Prisma Adds Support for MongoDB' },
    update: {},
    create: {
      title: 'Prisma Adds Support for MongoDB',
      body: 'Support for MongoDB has been one of the most requested features since the initial release of...',
      description:
        "We are excited to share that today's Prisma ORM release adds stable support for MongoDB!",
      published: false,
    },
  });

  const post2 = await prisma.article.upsert({
    where: { title: "What's new in Prisma? (Q1/22)" },
    update: {},
    create: {
      title: "What's new in Prisma? (Q1/22)",
      body: 'Our engineers have been working hard, issuing new releases with many improvements...',
      description:
        'Learn about everything in the Prisma ecosystem and community from January to March 2022.',
      published: true,
    },
  });

  console.log({ post1, post2 });
}

// execute the main function
main()
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(async () => {
    // close Prisma Client at the end
    await prisma.$disconnect();
  });
```

## Tell Prisma what script to execute when running the seeding command.

- Adding the prisma.seed key to the end of your package.json

```json
 "jest": {    // ...
  },
  "prisma": {
    "seed": "ts-node prisma/seed.ts"
  }
```

- Execute seeding with the following command:

```bash
$ npx prisma db seed
```

## Generate prisma module and service

```bash
$ npx nest generate module prisma
$ npx nest generate service prisma
```

- Update the service file to contain the following code:

```ts
// src/prisma/prisma.service.ts
import { INestApplication, Injectable } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient {
  async enableShutdownHooks(app: INestApplication) {
    this.$on('beforeExit', async () => {
      await app.close();
    });
  }
}
```

- Export Prisma Service, so that any module that imports the PrismaModule will have access to PrismaService and can inject it into its own components/services.

```ts
// src/prisma/prisma.module.ts

import { Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

# Set up Swagger

- Install the required dependencies

```bash
$ npm install --save @nestjs/swagger swagger-ui-express
```

- Now open main.ts and initialize Swagger using the SwaggerModule class:

```ts
// src/main.ts

import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('Median')
    .setDescription('The Median API description')
    .setVersion('0.1')
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document);

  await app.listen(3000);
}
bootstrap();
```

- Open the [Swagger API page](http://localhost:3000/api/) in your browser

# Generate REST resources

```bash
$ npx nest generate resource
```

- You will be given a few CLI prompts. Answer the questions accordingly:

1. What name would you like to use for this resource (plural, e.g., "users")? articles
2. What transport layer do you use? REST API
3. Would you like to generate CRUD entry points? Yes

- Open the [Swagger API page](http://localhost:3000/api/) in your browser

# Import PrismaModule to the ArticlesModule

```ts
// src/articles/articles.module.ts
@Module({
  imports: [PrismaModule],
})
```

# Inject the PrismaService inside the ArticlesService and use it to access the database.

```ts
constructor(private prisma: PrismaService) {}
```

## Update ArticlesService methods to updata and query from database

```ts
// src/articles/articles.service.ts

@Injectable()
export class ArticlesService {
  constructor(private prisma: PrismaService) {}

  create(createArticleDto: CreateArticleDto) {
    return this.prisma.article.create({ data: createArticleDto })
  }

  findAll() {
    return this.prisma.article.findMany({ where: { published: true } });
  }

  findOne(id: number) {
    return this.prisma.article.findUnique({ where: { id } })
  }

  update(id: number, updateArticleDto: UpdateArticleDto) {
    return this.prisma.article.update({
      where: { id },
      data: updateArticleDto,
    });
  }

  remove(id: number) {
    return this.prisma.article.delete({ where: { id } });
  }
```

## Group endpoints together in Swagger

- Add an @ApiTags decorator to the ArticlesController class, to group all the articles endpoints together in Swagger:

```ts
// src/articles/articles.controller.ts

import { ApiTags } from '@nestjs/swagger';
@Controller('articles')
@ApiTags('articles')
export class ArticlesController {
  // ...
}
```

- Open the [Swagger API page](http://localhost:3000/api/) in your browser

## Update Swagger response types

- update the ArticleEntity class in the articles.entity.ts file as follows:

```ts
// src/articles/entities/article.entity.ts

import { Article } from '@prisma/client';
import { ApiProperty } from '@nestjs/swagger';

export class ArticleEntity implements Article {
  @ApiProperty()
  id: number;

  @ApiProperty()
  title: string;

  @ApiProperty({ required: false, nullable: true })
  description: string | null;

  @ApiProperty()
  body: string;

  @ApiProperty()
  published: boolean;

  @ApiProperty()
  createdAt: Date;

  @ApiProperty()
  updatedAt: Date;
}
```

- update the ArticlesController class in the articles.controller.ts file as follows:

```ts
// src/articles/articles.controller.ts

+import { ApiCreatedResponse, ApiOkResponse, ApiTags } from '@nestjs/swagger';
+import { ArticleEntity } from './entities/article.entity';

@Controller('articles')
@ApiTags('articles')
export class ArticlesController {
  constructor(private readonly articlesService: ArticlesService) {}

  @Post()
+ @ApiCreatedResponse({ type: ArticleEntity })
  create(@Body() createArticleDto: CreateArticleDto) {
    return this.articlesService.create(createArticleDto);
  }

  @Get()
+ @ApiOkResponse({ type: ArticleEntity, isArray: true })
  findAll() {
    return this.articlesService.findAll();
  }

  @Get('drafts')
+ @ApiOkResponse({ type: ArticleEntity, isArray: true })
  findDrafts() {
    return this.articlesService.findDrafts();
  }

  @Get(':id')
+ @ApiOkResponse({ type: ArticleEntity })
  findOne(@Param('id') id: string) {
    return this.articlesService.findOne(+id);
  }

  @Patch(':id')
+ @ApiOkResponse({ type: ArticleEntity })
  update(@Param('id') id: string, @Body() updateArticleDto: UpdateArticleDto) {
    return this.articlesService.update(+id, updateArticleDto);
  }

  @Delete(':id')
+ @ApiOkResponse({ type: ArticleEntity })
  remove(@Param('id') id: string) {
    return this.articlesService.remove(+id);
  }
}
```
