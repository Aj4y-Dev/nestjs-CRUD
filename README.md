### TypeORM is responsible for:

- Connecting to PostgreSQL
- Creating SQL queries
- Mapping database tables to TypeScript classes
- Saving data
- Updating data
- Deleting data

This is the PostgreSQL driver. TypeORM doesn't know how to talk to PostgreSQL by itself. It uses pg underneath.

```
pnpm install --save @nestjs/typeorm typeorm pg
```

@nestjs/config is NestJS's official configuration module. It is mainly used to:

Read environment variables from .env
Access configuration anywhere in your application
Validate configuration
Keep secrets (database URLs, JWT secrets, API keys) out of your source code

```
pnpm i --save @nestjs/config
```

This is the NestJS integration for TypeORM. It allows NestJS to work nicely with TypeORM using dependency injection. Without it, you would have to manually create database connections. ORM (Object Relational Mapper).

```
TypeOrmModule.forRootAsync({
  imports: [ConfigModule], //Before TypeORM is created, Nest asks: "Does TypeORM depend on another module?"
  inject: [ConfigService], //This is Dependency Injection.
  useFactory: (configService: ConfigService) => ({ //This is the heart of everything. It automatically connect the Typeorm
    type: 'postgres',
    host: configService.get('DB_HOST'),
    port: Number(configService.get('DB_PORT')),
    username: configService.get('DB_USERNAME'),
    password: configService.get('DB_PASSWORD'),
    database: configService.get('DB_NAME'),
    entities: [join(process.cwd(), 'dist/**/*.entity.js')],//An Entity is simply a TypeScript class that represents a database table. current workin dir inside dist Search all folders recursively, matches every file ending with: .entity.js like user.entity.js
    synchronize: true, //It's fantastic for development but the problem is that it's dangerous in production because TypeORM is allowed to modify your database automatically. Basically in Development `synchronize: true` in Production `synchronize: false`
  }),
});
```

When your application starts, TypeORM asks: "Hey NestJS, I need my database configuration. Where do I get it?" . NestJS replies: "Don't worry. I'll call a function that returns your configuration." That function is useFactory.

### synchronize: true

Scenario 1 — You add a new column
You change your entity:

```
@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @Column()
  email: string;
}
```

When you restart the application:

TypeORM automatically runs something similar to:

```
ALTER TABLE users
ADD COLUMN email VARCHAR;

Perfect!
This is why synchronize is amazing during development.
```

Scenario 2 — You accidentally remove a property
Yesterday:

```
@Column()
name: string;
```

Today you accidentally delete it.

```
@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;
}
```

Now your database still has:

| id  | name |
| --- | ---- |
| 1   | Ajay |
| 2   | Ram  |

TypeORM starts. It compares:

- Entity: id
- Database: id name

It thinks: "The name column shouldn't exist anymore."

```
It may execute:

ALTER TABLE users
DROP COLUMN name;
```

Now every user's name is gone. You didn't type any SQL. TypeORM did.
