# Automating Repository Generation in NestJS with Prisma

In a NestJS or any other backend applications, repository classes act as the data access layer. Managing repositories in a Prisma + NestJS application can get tedious, especially when dealing with multiple models. Manually creating repository files for each model is inefficient and prone to errors. Instead of writing the same CRUD methods for every model, this article will guide you way to dynamically generate repository classes with transaction handling based on the Prisma schema. This approach not only reduces boilerplate code but also ensures consistency across your application.

> Before starting, make sure you have NestJS application setup with Prisma. If you haven't done that yet, check out the NestJS + Prisma documentation for a quick setup.

## Working Mechanism

Example prisma schema file (schema.prisma) has Country model which will act as example model throughout the article.

```prisma
// prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model Country {
  id        Int      @id @default(autoincrement())
  name      String   @unique
  code      String   @unique
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

## Creating the Base Repository

First, let's create a base repository that all our generated repositories will extend:

```typescript
// src/common/repositories/base.repository.ts
import { PrismaService } from '../prisma/prisma.service';

export class BaseRepository<T> {
  constructor(
    protected readonly prisma: PrismaService,
    private readonly modelName: string,
  ) {}

  async findAll(): Promise<T[]> {
    return this.prisma[this.modelName].findMany();
  }

  async findById(id: number): Promise<T | null> {
    return this.prisma[this.modelName].findUnique({
      where: { id },
    });
  }

  async create(data: any): Promise<T> {
    return this.prisma[this.modelName].create({
      data,
    });
  }

  async update(id: number, data: any): Promise<T> {
    return this.prisma[this.modelName].update({
      where: { id },
      data,
    });
  }

  async delete(id: number): Promise<T> {
    return this.prisma[this.modelName].delete({
      where: { id },
    });
  }

  // Transaction support
  async createWithTransaction(data: any, tx: any): Promise<T> {
    return tx[this.modelName].create({
      data,
    });
  }
  
  // Add more methods as needed
}
```

## Repository Factory

Now, let's create a factory that will generate repositories for each model:

```typescript
// src/common/repositories/repository.factory.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { BaseRepository } from './base.repository';

@Injectable()
export class RepositoryFactory {
  constructor(private readonly prisma: PrismaService) {}

  createRepository<T>(modelName: string): BaseRepository<T> {
    return new BaseRepository<T>(this.prisma, modelName);
  }
}
```

## Using the Generated Repositories

With our factory in place, we can now easily create repositories for any model:

```typescript
// src/country/country.service.ts
import { Injectable } from '@nestjs/common';
import { Country } from '@prisma/client';
import { RepositoryFactory } from '../common/repositories/repository.factory';
import { BaseRepository } from '../common/repositories/base.repository';

@Injectable()
export class CountryService {
  private countryRepository: BaseRepository<Country>;

  constructor(private repositoryFactory: RepositoryFactory) {
    this.countryRepository = repositoryFactory.createRepository<Country>('country');
  }

  async getAllCountries(): Promise<Country[]> {
    return this.countryRepository.findAll();
  }

  async getCountryById(id: number): Promise<Country | null> {
    return this.countryRepository.findById(id);
  }

  async createCountry(data: Omit<Country, 'id' | 'createdAt' | 'updatedAt'>): Promise<Country> {
    return this.countryRepository.create(data);
  }

  // Add more methods as needed
}
```

## Advanced: Type-Safe Repositories

For better type safety, we can enhance our factory to return properly typed repositories:

```typescript
// src/common/repositories/repository.factory.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { BaseRepository } from './base.repository';
import { Country, User, Product } from '@prisma/client';

type ModelName = 'country' | 'user' | 'product'; // Add all your model names

type ModelType<T extends ModelName> = 
  T extends 'country' ? Country :
  T extends 'user' ? User :
  T extends 'product' ? Product :
  never;

@Injectable()
export class RepositoryFactory {
  constructor(private readonly prisma: PrismaService) {}

  createRepository<T extends ModelName>(modelName: T): BaseRepository<ModelType<T>> {
    return new BaseRepository<ModelType<T>>(this.prisma, modelName);
  }
}
```

## Transaction Handling

One of the key benefits of this approach is consistent transaction handling:

```typescript
// Example of using transactions
async transferOwnership(countryId: number, newOwnerId: number) {
  return this.prisma.$transaction(async (tx) => {
    // Update country
    await this.countryRepository.createWithTransaction(
      { id: countryId, ownerId: newOwnerId },
      tx
    );
    
    // Update other related entities in the same transaction
    // ...
    
    return { success: true };
  });
}
```

## Benefits of This Approach

1. **Reduced Boilerplate**: No need to write the same CRUD methods for every model
2. **Consistency**: All repositories follow the same pattern
3. **Type Safety**: With TypeScript, you get full type checking
4. **Transaction Support**: Easy to use transactions across multiple repositories
5. **Maintainability**: Changes to repository logic only need to be made in one place

## Conclusion

By dynamically generating repository classes based on your Prisma schema, you can significantly reduce the amount of code you need to write and maintain. This approach ensures consistency across your application and makes it easier to implement features like transaction handling. As your application grows, you can extend the base repository with additional methods that all models can benefit from.

This pattern works exceptionally well for NestJS applications but can be adapted for any TypeScript backend using Prisma as the ORM.
