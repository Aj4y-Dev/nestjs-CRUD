# NestJS CRUD with PostgreSQL & TypeORM

This repository is a step-by-step demonstration of how to build a full **CRUD (Create, Read, Update, Delete)** API using **NestJS**, **TypeORM**, and **PostgreSQL**.

---

## 🚀 Step-by-Step Guide: How to Create CRUD in NestJS

Here is a detailed guide on how this CRUD system was built, and how you can create your own CRUD resource (e.g., `cities` or `users`) from scratch.

### Step 1: Generate the Resource Scaffold
NestJS CLI provides a powerful generator that creates all the boilerplate files for a REST API. Run the following command in your terminal:

```bash
nest g res cities
```

When prompted:
1. **What transport layer do you use?** Select `REST API`
2. **Would you like to generate CRUD entry points?** Select `Yes`

This will generate the following structure inside `src/cities`:
```
src/cities/
├── dto/
│   ├── create-city.dto.ts
│   └── update-city.dto.ts
├── entities/
│   └── city.entity.ts
├── cities.controller.ts
├── cities.module.ts
└── cities.service.ts
```

---

### Step 2: Define the Database Entity (`city.entity.ts`)
An **Entity** represents your database table structure. We use TypeORM decorators to define columns and relationships.

File: [city.entity.ts](file:///c:/Users/Lenovo/OneDrive/Desktop/Projects/Practise/pg-crud/src/cities/entities/city.entity.ts)
```typescript
import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';

@Entity({ name: 'Cities' }) // Custom table name in PostgreSQL
export class City {
  @PrimaryGeneratedColumn() // Auto-incrementing primary key
  id!: number;

  @Column({ unique: true }) // Unique constraint for city name
  name!: string;

  @Column({ type: 'text', nullable: true }) // Nullable text column
  description!: string;

  @Column({ type: 'boolean', default: true }) // Boolean with default value
  active!: boolean;
}
```

---

### Step 3: Define Data Transfer Objects (DTOs)
DTOs define the shape of data sent in network requests and allow validation.

1. **Create DTO (`create-city.dto.ts`)**:
   File: [create-city.dto.ts](file:///c:/Users/Lenovo/OneDrive/Desktop/Projects/Practise/pg-crud/src/cities/dto/create-city.dto.ts)
   ```typescript
   export class CreateCityDto {
     name!: string;
     description!: string;
     active!: boolean;
   }
   ```

2. **Update DTO (`update-city.dto.ts`)**:
   Inherits all fields from `CreateCityDto` but makes them optional.
   File: [update-city.dto.ts](file:///c:/Users/Lenovo/OneDrive/Desktop/Projects/Practise/pg-crud/src/cities/dto/update-city.dto.ts)
   ```typescript
   import { PartialType } from '@nestjs/mapped-types';
   import { CreateCityDto } from './create-city.dto';

   export class UpdateCityDto extends PartialType(CreateCityDto) {}
   ```

---

### Step 4: Register the Entity in the Modules
To use the `City` repository inside our services, we need to register it in both the resource module and the root module.

1. **Feature Module (`cities.module.ts`)**:
   Import `TypeOrmModule.forFeature` to register the `City` entity.
   File: [cities.module.ts](file:///c:/Users/Lenovo/OneDrive/Desktop/Projects/Practise/pg-crud/src/cities/cities.module.ts)
   ```typescript
   import { Module } from '@nestjs/common';
   import { CitiesService } from './cities.service';
   import { CitiesController } from './cities.controller';
   import { TypeOrmModule } from '@nestjs/typeorm';
   import { City } from './entities/city.entity';

   @Module({
     imports: [TypeOrmModule.forFeature([City])], // Registering City Entity
     controllers: [CitiesController],
     providers: [CitiesService],
   })
   export class CitiesModule {}
   ```

2. **Root Module (`app.module.ts`)**:
   Register it in your root database options so TypeORM knows it exists.
   File: [app.module.ts](file:///c:/Users/Lenovo/OneDrive/Desktop/Projects/Practise/pg-crud/src/app.module.ts)
   ```typescript
   import { Module } from '@nestjs/common';
   import { TypeOrmModule } from '@nestjs/typeorm';
   import { CitiesModule } from './cities/cities.module';
   import { City } from './cities/entities/city.entity';

   @Module({
     imports: [
       // ... other configurations (ConfigModule, etc.)
       TypeOrmModule.forRootAsync({
         // ... connection details
         entities: [City],
         synchronize: true, // Auto-create tables in development
       }),
       CitiesModule,
     ],
   })
   export class AppModule {}
   ```

---

### Step 5: Implement CRUD logic in the Service (`cities.service.ts`)
The Service contains business logic and interacts directly with PostgreSQL using the TypeORM repository patterns.

File: [cities.service.ts](file:///c:/Users/Lenovo/OneDrive/Desktop/Projects/Practise/pg-crud/src/cities/cities.service.ts)
```typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { CreateCityDto } from './dto/create-city.dto';
import { UpdateCityDto } from './dto/update-city.dto';
import { City } from './entities/city.entity';
import { Repository } from 'typeorm';
import { InjectRepository } from '@nestjs/typeorm';

@Injectable()
export class CitiesService {
  constructor(
    @InjectRepository(City)
    private readonly citiesRepository: Repository<City>, // Injecting TypeORM Repository
  ) {}

  // CREATE
  async create(createCityDto: CreateCityDto) {
    const city = this.citiesRepository.create(createCityDto);
    return await this.citiesRepository.save(city);
  }

  // READ (ALL)
  async findAll() {
    return await this.citiesRepository.find();
  }

  // READ (ONE)
  async findOne(id: number) {
    const city = await this.citiesRepository.findOne({ where: { id } });
    if (!city) {
      throw new NotFoundException(`City with ID ${id} not found`);
    }
    return city;
  }

  // UPDATE
  async update(id: number, updateCityDto: UpdateCityDto) {
    const city = await this.findOne(id); // Reuses findOne validation logic
    Object.assign(city, updateCityDto); // Merges updates onto the retrieved entity
    return await this.citiesRepository.save(city);
  }

  // DELETE
  async remove(id: number) {
    const city = await this.findOne(id);
    return await this.citiesRepository.remove(city);
  }
}
```

---

### Step 6: Expose Endpoints in the Controller (`cities.controller.ts`)
The Controller handles HTTP requests and maps them to service functions.

File: [cities.controller.ts](file:///c:/Users/Lenovo/OneDrive/Desktop/Projects/Practise/pg-crud/src/cities/cities.controller.ts)
```typescript
import { Controller, Get, Post, Body, Patch, Param, Delete } from '@nestjs/common';
import { CitiesService } from './cities.service';
import { CreateCityDto } from './dto/create-city.dto';
import { UpdateCityDto } from './dto/update-city.dto';

@Controller('cities') // Base Route: http://localhost:3000/cities
export class CitiesController {
  constructor(private readonly citiesService: CitiesService) {}

  @Post()
  create(@Body() createCityDto: CreateCityDto) {
    return this.citiesService.create(createCityDto);
  }

  @Get()
  findAll() {
    return this.citiesService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.citiesService.findOne(+id); // '+' converts string parameter to number
  }

  @Patch(':id')
  update(@Param('id') id: string, @Body() updateCityDto: UpdateCityDto) {
    return this.citiesService.update(+id, updateCityDto);
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return this.citiesService.remove(+id);
  }
}
```

---

## 🛠️ Installation & Setup

### 1. Install Dependencies
Make sure you have the key packages installed:
```bash
pnpm install --save @nestjs/typeorm typeorm pg @nestjs/config
```
* `@nestjs/typeorm`: TypeORM integration for NestJS.
* `typeorm`: Object Relational Mapper for TypeScript/JavaScript.
* `pg`: PostgreSQL database client driver.
* `@nestjs/config`: Config environment configuration utility.

### 2. Configure `.env` File
Create a `.env` file in the root directory:
```env
DB_HOST=your-neon-database-host.neon.tech
DB_PORT=5432
DB_USERNAME=your-username
DB_PASSWORD=your-password
DB_NAME=your-database-name
```

---

## ⚠️ Important Configuration Details

### Database Sync (`synchronize: true`)
In `app.module.ts`, we set `synchronize: true`.
* **In Development:** Extremely helpful because TypeORM will automatically alter tables and schemas to match your entity classes.
* **In Production:** **Danger!** Turn this off (`false`) to avoid accidental data loss. If you remove an entity field, TypeORM will drop the corresponding column, resulting in complete loss of data for that column.

### SSL Connection for Neon Database
Since Neon.tech PostgreSQL requires secure connections, SSL is configured in `app.module.ts`:
```typescript
ssl: {
  rejectUnauthorized: false; // Establish secure connection bypassing local chain validation
}
```

---

## 📡 API Endpoints Reference

| Method | Endpoint | Description | Request Body |
| :--- | :--- | :--- | :--- |
| **POST** | `/cities` | Create a new city | `{ "name": "Berlin", "description": "Capital of Germany", "active": true }` |
| **GET** | `/cities` | Get all cities | *None* |
| **GET** | `/cities/:id` | Get city details by ID | *None* |
| **PATCH** | `/cities/:id` | Update an existing city | `{ "description": "Updated city description" }` |
| **DELETE** | `/cities/:id` | Remove a city by ID | *None* |
