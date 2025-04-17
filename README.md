# Automating Repository Generation in NestJS with Prisma

In a NestJS or any other backend application, repository classes act as the data access layer. Managing repositories in a Prisma + NestJS application can get tedious, especially when dealing with multiple models. Manually creating repository files for each model is inefficient and prone to errors. Instead of writing the same CRUD methods for every model, this article will guide you through dynamically generating repository classes with transaction handling based on the Prisma schema. This approach not only reduces boilerplate code but also ensures consistency across your application.

> **Before you begin:** make sure you have a NestJS application set up with Prisma. If not, see the [NestJS + Prisma docs](https://docs.nestjs.com/recipes/prisma).

## Table of Contents

1. [Working Mechanism](#working-mechanism)  
2. [Creating the Base Repository](#creating-the-base-repository)  
3. [Repository Factory](#repository-factory)  
4. [Using the Generated Repositories](#using-the-generated-repositories)  
5. [Advanced: Type‑Safe Repositories](#advanced-type-safe-repositories)  
6. [CLI Automation for Repo Generation](#cli-automation-for-repo-generation)  
7. [Generating Repositories with Decorators](#generating-repositories-with-decorators)  
8. [Testing the Generated Repositories](#testing-the-generated-repositories)  
9. [Best Practices & Patterns](#best-practices--patterns)  
10. [Comparison: Manual vs. Automated](#comparison-manual-vs-automated)  
11. [Conclusion](#conclusion)  

## Working Mechanism

Example Prisma schema file (`schema.prisma`) has a `Country` model which will act as our running example:

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

> 🚀 **Tip:** Use `prisma db pull` to introspect an existing database into your Prisma schema.

## Creating the Base Repository

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
    return this.prisma[this.modelName].create({ data });
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
    return tx[this.modelName].create({ data });
  }
}
```

## Repository Factory

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

```typescript
// src/country/country.service.ts
import { Injectable } from '@nestjs/common';
import { Country } from '@prisma/client';
import { RepositoryFactory } from '../common/repositories/repository.factory';

@Injectable()
export class CountryService {
  private countryRepository = this.repositoryFactory.createRepository<Country>('country');

  constructor(private repositoryFactory: RepositoryFactory) {}

  async getAllCountries(): Promise<Country[]> {
    return this.countryRepository.findAll();
  }

  async getCountryById(id: number): Promise<Country | null> {
    return this.countryRepository.findById(id);
  }

  async createCountry(data: Omit<Country, 'id' | 'createdAt' | 'updatedAt'>): Promise<Country> {
    return this.countryRepository.create(data);
  }
}
```

## Advanced: Type‑Safe Repositories

```typescript
// src/common/repositories/repository.factory.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { BaseRepository } from './base.repository';
import { Country, User, Product } from '@prisma/client';

type ModelName = 'country' | 'user' | 'product';
type ModelType<T extends ModelName> =
  T extends 'country' ? Country :
  T extends 'user'    ? User :
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

## CLI Automation for Repo Generation

You can script generation via a Node.js CLI:

```typescript
// scripts/generate-repos.ts
import { writeFileSync } from 'fs';
import { PrismaClient } from '@prisma/client';
import path from 'path';

const prisma = new PrismaClient();
async function main() {
  const models = Object.keys(prisma._dmmf.modelMap);
  for (const model of models) {
    const className = `${model.charAt(0).toUpperCase() + model.slice(1)}Repository`;
    const content = `
      import { BaseRepository } from '../common/repositories/base.repository';
      export class ${className} extends BaseRepository<typeof prisma.${model}> {
        constructor(prisma: typeof prisma.${model}) { super(prisma, '${model}'); }
      }
    `;
    writeFileSync(path.join(__dirname, '../src/generated', `${className}.ts`), content);
  }
  console.log('✅ Repositories generated for models:', models.join(', '));
}

main().catch(console.error);
```

- **Run**: `ts-node scripts/generate-repos.ts`  
- **Output**: Generates one file per model under `src/generated/`.

## Generating Repositories with Decorators

```typescript
// src/common/decorators/repository.decorator.ts
import { applyDecorators, Injectable } from '@nestjs/common';

export function AutoRepository(model: string) {
  return applyDecorators(
    Injectable(),
    (target: any) => {
      Reflect.defineMetadata('prisma:model', model, target);
    }
  );
}

// Usage
@AutoRepository('country')
export class CountryRepository {}
```

> ℹ️ **Note:** You can read `Reflect.getMetadata('prisma:model', CountryRepository)` in a factory to register it automatically.

## Testing the Generated Repositories

```typescript
// test/base.repository.spec.ts
import { Test } from '@nestjs/testing';
import { BaseRepository } from '../src/common/repositories/base.repository';
import { PrismaService } from '../src/common/prisma/prisma.service';

describe('BaseRepository', () => {
  let repo: BaseRepository<any>;
  let prisma: PrismaService;

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      providers: [PrismaService],
    }).compile();
    prisma = module.get(PrismaService);
    repo = new BaseRepository(prisma, 'country');
  });

  it('should find all records', async () => {
    const result = await repo.findAll();
    expect(Array.isArray(result)).toBe(true);
  });

  it('should handle createWithTransaction', async () => {
    await prisma.$transaction(async (tx) => {
      const res = await repo.createWithTransaction({ name: 'Test', code: 'T1' }, tx);
      expect(res).toHaveProperty('id');
    });
  });
});
```

## Best Practices & Patterns

- **DRY principle:** Keep your base repository logic in one place.  
- **Single responsibility:** Repositories only handle data access.  
- **Model metadata:** Use decorators or metadata for automated registration.  
- **CI Integration:** Add your CLI script to CI pipelines to regenerate as schema changes.  
- **Documentation:** Keep a README section updated with usage examples.

## Comparison: Manual vs. Automated

| Feature                     | Manual Repositories | Automated Generation |
| --------------------------- | ------------------- | -------------------- |
| Boilerplate                 | High                | Low                  |
| Consistency                 | Varies              | Uniform              |
| Type Safety                 | Medium              | Strong               |
| Transaction Handling        | Manual              | Built‑in             |
| Maintenance Overhead        | High                | Minimal              |

## Conclusion

By dynamically generating repository classes based on your Prisma schema and leveraging decorators, CLI scripts, and type‑safe factories, you can:

1. **Eliminate boilerplate** across all models.  
2. **Ensure consistency** and minimize human error.  
3. **Leverage full type safety** with TypeScript.  
4. **Easily integrate** with your CI/CD and testing pipelines.  

This pattern works not just for NestJS but for any TypeScript backend using Prisma as the ORM. Happy coding!    

