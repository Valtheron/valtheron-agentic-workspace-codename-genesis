This document outlines a critical refactoring initiative within the Valtheron Agentic Workspace: migrating our core data persistence layer from SQLite to a more robust, scalable solution like PostgreSQL or Cloud SQL. This tutorial serves as a blueprint, guiding contributors through the conceptual understanding, architectural considerations, and practical implementation steps required to achieve this transition.

---

# Refactoring Data Persistence: From SQLite to Scalable SQL

**Path:** `docs/roadmap/v2-database-scaling.md`
**Topic:** Database Scaling and Abstraction
**Target Audience:** Valtheron Core Contributors, Architects, and Senior Developers

## 1. Executive Summary

The Valtheron Agentic Workspace, while benefiting from SQLite's simplicity for local development and embedded logging, is encountering significant limitations as it scales. Specifically, high concurrency from numerous background agent threads leads to database locking and exhaustion of file descriptors, hindering performance and reliability.

This document proposes a refactor to migrate our primary data store to a production-grade relational database like PostgreSQL or a managed service such as Cloud SQL. The core of this initiative involves:

1.  **Establishing a Database Abstraction Layer:** Implementing a modular SQL adapter interface to decouple our application logic from the underlying database technology.
2.  **Implementing PostgreSQL/Cloud SQL Support:** Developing a concrete adapter for PostgreSQL, leveraging connection pooling for efficient resource management.
3.  **Standardizing Data Models:** Utilizing Drizzle ORM to define consistent, type-safe schemas across different database backends.
4.  **Developing Robust Migration Tools:** Creating scripts for backfilling existing SQLite data into the new PostgreSQL database with comprehensive validation.

This refactor is crucial for Valtheron's scalability, performance, enterprise readiness, and long-term maintainability.

## 2. Conceptual Explanation

### 2.1 The Challenge: SQLite's Limitations at Scale

SQLite is an excellent choice for embedded, single-user, or low-concurrency applications. Its file-based nature simplifies deployment and management. However, for a multi-threaded, highly concurrent application like Valtheron, where numerous agents might simultaneously attempt to read from and write to the database:

*   **Database Locking:** SQLite implements database-level locking for write operations. When dozens of background threads contend for writes, this leads to significant bottlenecks, degraded performance, and potential deadlocks.
*   **File Descriptor Exhaustion:** As Valtheron scales, the number of open files (including the SQLite database file) can exceed system limits, leading to operational instability.
*   **Lack of Advanced Features:** SQLite lacks native support for connection pooling, replication, advanced backup/restore strategies, and robust concurrency control mechanisms (like MVCC), which are essential for production environments.

### 2.2 The Solution: PostgreSQL/Cloud SQL for Enterprise Readiness

PostgreSQL is a powerful, open-source object-relational database system renowned for its reliability, feature robustness, and performance. Cloud SQL offers PostgreSQL as a fully managed service, further simplifying operations. Key advantages include:

*   **Robust Concurrency (MVCC):** PostgreSQL uses Multi-Version Concurrency Control (MVCC), allowing readers and writers to operate on different versions of data without blocking each other, significantly improving concurrent access.
*   **Connection Pooling:** Native support for connection pooling allows efficient management of database connections, reducing overhead and improving throughput for high-volume applications.
*   **Scalability & Replication:** PostgreSQL supports various scaling strategies, including read replicas and logical replication, enabling horizontal scaling and high availability.
*   **Advanced Features:** Offers robust indexing, stored procedures, sophisticated querying capabilities, and strong ACID compliance.
*   **Managed Services (Cloud SQL):** Reduces operational burden for patching, backups, and scaling, allowing contributors to focus on application logic.

### 2.3 Architectural Philosophy: Database Abstraction and Drizzle ORM

To ensure a smooth transition and maintain architectural flexibility, we will implement a clear **Database Abstraction Layer** using the **Adapter Pattern**. This layer will define a contract (interface) that all database implementations must adhere to, decoupling our business logic from the specific database technology.

**Key Components:**

*   **`IDatabaseAdapter` Interface:** This TypeScript interface will define common database operations (e.g., `connect`, `disconnect`, `query`, `insert`, `update`, `findById`).
*   **Concrete Adapters:** Implementations of `IDatabaseAdapter` for specific databases (e.g., `SQLiteAdapter`, `PostgresAdapter`).
*   **Drizzle ORM:** We will leverage Drizzle ORM for schema definition, query building, and type safety. Drizzle's ability to generate SQL for various dialects (SQLite, PostgreSQL, MySQL) makes it ideal for our migration strategy, ensuring our model definitions remain consistent.

This approach offers several benefits:

*   **Modularity:** Easy to switch or add new database backends in the future.
*   **Testability:** Simplifies unit and integration testing of data access logic.
*   **Maintainability:** Centralizes database interaction logic, making it easier to manage and update.
*   **Type Safety:** Drizzle ORM, combined with TypeScript, provides end-to-end type safety from schema definition to query results.

### 2.4 Migration Strategy: A Phased Approach

The migration will follow a structured, phased approach to minimize downtime and ensure data integrity:

1.  **Define Adapter Interface:** Formalize the `IDatabaseAdapter` interface.
2.  **Implement PostgreSQL Adapter:** Develop the `PostgresAdapter` conforming to the interface, including connection pooling.
3.  **Refactor Data Access Layer:** Update existing data access services and repositories to utilize the `IDatabaseAdapter` interface, initially pointing to the existing `SQLiteAdapter`.
4.  **Data Model Translation:** Define all core entity schemas using Drizzle ORM, ensuring compatibility with both SQLite and PostgreSQL.
5.  **Develop Migration Scripts:** Create idempotent scripts to read data from the existing SQLite database and write it into the new PostgreSQL database. This includes comprehensive data validation (count checks, checksums).
6.  **Dual-Write/Shadow Mode (Optional but Recommended):** For critical data, consider a period where writes go to both SQLite and PostgreSQL, allowing for real-time validation without committing to the new database immediately.
7.  **Cutover:** Switch the `IDatabaseAdapter` implementation in production from `SQLiteAdapter` to `PostgresAdapter`.
8.  **Monitoring & Rollback Plan:** Implement robust monitoring and have a clear rollback strategy in case of unforeseen issues.

## 3. Step-by-Step Code Examples

These examples illustrate the core concepts using TypeScript and Express 5.1 conventions.

### 3.1 Defining the Database Adapter Interface (`src/database/interfaces/IDatabaseAdapter.ts`)

```typescript
// src/database/interfaces/IDatabaseAdapter.ts

import { SQL } from 'drizzle-orm';
import { BaseSchema, InferSelectModel } from 'drizzle-orm'; // Assuming BaseSchema for generic types

/**
 * Represents a generic Valtheron entity with common fields like ID, creation/update timestamps.
 */
export interface IValtheronEntity {
  id: string;
  createdAt: Date;
  updatedAt: Date;
  // Potentially other common fields like `createdBy`, `isDeleted`, `version` for optimistic locking
}

/**
 * Defines the contract for any database adapter used within Valtheron.
 * This ensures loose coupling between our application logic and the specific database implementation.
 */
export interface IDatabaseAdapter {
  /**
   * Establishes a connection to the database.
   * @returns A promise that resolves when the connection is established.
   */
  connect(): Promise<void>;

  /**
   * Disconnects from the database, releasing resources.
   * @returns A promise that resolves when the disconnection is complete.
   */
  disconnect(): Promise<void>;

  /**
   * Executes a raw SQL query.
   * @param query The SQL query string or Drizzle SQL object.
   * @param params Optional parameters for the query.
   * @returns A promise resolving to the query result.
   */
  query<T = unknown>(query: SQL | string, params?: any[]): Promise<T[]>;

  /**
   * Inserts a new record into a specified table.
   * @param table The Drizzle schema table object.
   * @param values The values to insert.
   * @returns A promise resolving to the inserted record(s).
   */
  insert<TTable extends BaseSchema, TValues extends TTable['$inferInsert']>(
    table: TTable,
    values: TValues
  ): Promise<InferSelectModel<TTable>[]>;

  /**
   * Updates records in a specified table based on a condition.
   * @param table The Drizzle schema table object.
   * @param values The values to update.
   * @param where A Drizzle SQL condition for the update.
   * @returns A promise resolving to the updated record(s).
   */
  update<TTable extends BaseSchema, TValues extends TTable['$inferInsert']>(
    table: TTable,
    values: TValues,
    where: SQL
  ): Promise<InferSelectModel<TTable>[]>;

  /**
   * Finds records in a specified table based on a condition.
   * @param table The Drizzle schema table object.
   * @param where A Drizzle SQL condition for the selection.
   * @param limit Optional limit for the number of records.
   * @param offset Optional offset for pagination.
   * @returns A promise resolving to an array of found records.
   */
  find<TTable extends BaseSchema>(
    table: TTable,
    where?: SQL,
    limit?: number,
    offset?: number
  ): Promise<InferSelectModel<TTable>[]>;

  /**
   * Finds a single record by its primary ID.
   * @param table The Drizzle schema table object.
   * @param id The ID of the record to find.
   * @returns A promise resolving to the found record or undefined.
   */
  findById<TTable extends BaseSchema>(
    table: TTable,
    id: string
  ): Promise<InferSelectModel<TTable> | undefined>;

  /**
   * Deletes records from a specified table based on a condition.
   * @param table The Drizzle schema table object.
   * @param where A Drizzle SQL condition for the deletion.
   * @returns A promise resolving to the deleted record(s).
   */
  delete<TTable extends BaseSchema>(
    table: TTable,
    where: SQL
  ): Promise<InferSelectModel<TTable>[]>;

  /**
   * Begins a database transaction.
   * @returns A promise resolving to a transaction object/interface.
   */
  beginTransaction?(): Promise<any>; // Specific transaction type depends on ORM/driver

  /**
   * Commits an active database transaction.
   * @param transaction The transaction object.
   * @returns A promise resolving on commit.
   */
  commitTransaction?(transaction: any): Promise<void>;

  /**
   * Rolls back an active database transaction.
   * @param transaction The transaction object.
   * @returns A promise resolving on rollback.
   */
  rollbackTransaction?(transaction: any): Promise<void>;

  // Add methods for schema migrations if handled by the adapter
  // migrateUp?(): Promise<void>;
  // migrateDown?(): Promise<void>;
}
```

### 3.2 Data Model Translation with Drizzle ORM (`src/database/schema/agents.ts`)

This example defines a schema for an `Agent` entity, which will be consistent across SQLite and PostgreSQL.

```typescript
// src/database/schema/agents.ts

import { pgTable, text, timestamp, boolean, pgEnum } from 'drizzle-orm/pg-core'; // For PostgreSQL
import { sqliteTable, text as sqliteText, integer as sqliteInteger } from 'drizzle-orm/sqlite-core'; // For SQLite

// Define an enum for agent status, mapping to database types
export const agentStatusEnum = pgEnum('agent_status', ['active', 'inactive', 'paused', 'error']);

/**
 * Agent Schema Definition for PostgreSQL.
 * This leverages Drizzle ORM's `pgTable` to define the table structure.
 */
export const agents = pgTable('agents', {
  id: text('id').primaryKey().notNull(), // UUID or KSUID for primary keys
  name: text('name').notNull(),
  description: text('description'),
  status: agentStatusEnum('status').notNull().default('inactive'),
  config: text('config').$type<Record<string, any>>().notNull().default('{}'), // Store JSON config as text
  isPublic: boolean('is_public').notNull().default(false),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
  // Add columns for audit trailing, e.g., createdByUserId, lastModifiedByUserId
  createdByUserId: text('created_by_user_id').notNull(),
  lastModifiedByUserId: text('last_modified_by_user_id'),
});

// If we need to support SQLite for local dev or specific use cases, we can define a similar schema:
// Note: SQLite has fewer native types, so some might need mapping (e.g., boolean to integer)
export const sqliteAgents = sqliteTable('agents', {
  id: sqliteText('id').primaryKey().notNull(),
  name: sqliteText('name').notNull(),
  description: sqliteText('description'),
  status: sqliteText('status', { enum: ['active', 'inactive', 'paused', 'error'] }).notNull().default('inactive'),
  config: sqliteText('config').notNull().default('{}'),
  isPublic: sqliteInteger('is_public', { mode: 'boolean' }).notNull().default(false),
  createdAt: sqliteInteger('created_at', { mode: 'timestamp' }).notNull().default(new Date()),
  updatedAt: sqliteInteger('updated_at', { mode: 'timestamp' }).notNull().default(new Date()),
  createdByUserId: sqliteText('created_by_user_id').notNull(),
  lastModifiedByUserId: sqliteText('last_modified_by_user_id'),
});

// Infer types for use throughout the application
export type Agent = typeof agents.$inferSelect; // Type for selecting an Agent
export type NewAgent = typeof agents.$inferInsert; // Type for inserting a new Agent
```

### 3.3 Implementing a PostgreSQL Adapter (`src/database/adapters/PostgresAdapter.ts`)

This adapter uses the `pg` driver and Drizzle ORM for PostgreSQL.

```typescript
// src/database/adapters/PostgresAdapter.ts

import { IDatabaseAdapter, IValtheronEntity } from '../interfaces/IDatabaseAdapter';
import { BaseSchema, InferSelectModel, SQL, eq, sql } from 'drizzle-orm';
import { PgDatabase, drizzle } from 'drizzle-orm/pg-core';
import { Pool } from 'pg';
import * as schema from '../schema'; // Import all Drizzle schemas
import { Logger } from '../../utils/logger'; // Assuming a logger utility
import { Config } from '../../config'; // Assuming a config utility for DB_URL

const logger = new Logger('PostgresAdapter');

/**
 * PostgresAdapter implements the IDatabaseAdapter interface for PostgreSQL.
 * It utilizes pg-pool for connection management and Drizzle ORM for type-safe queries.
 */
export class PostgresAdapter implements IDatabaseAdapter {
  private pool: Pool;
  private db: PgDatabase<typeof schema>;

  constructor() {
    this.pool = new Pool({
      connectionString: Config.get('DATABASE_URL'), // e.g., 'postgresql://user:password@host:port/database'
      max: Config.get('DB_CONNECTION_POOL_LIMIT', 20), // Connection pooling limit, from original draft
      ssl: Config.get('NODE_ENV') === 'production' ? { rejectUnauthorized: false } : false, // Enable SSL in production
    });
    // Initialize Drizzle with the pool and schema
    this.db = drizzle(this.pool, { schema });
  }

  /**
   * Connects to the PostgreSQL database.
   */
  public async connect(): Promise<void> {
    try {
      await this.pool.connect();
      logger.info('Connected to PostgreSQL database.');
    } catch (error) {
      logger.error('Failed to connect to PostgreSQL:', error);
      throw error;
    }
  }

  /**
   * Disconnects from the PostgreSQL database.
   */
  public async disconnect(): Promise<void> {
    try {
      await this.pool.end();
      logger.info('Disconnected from PostgreSQL database.');
    } catch (error) {
      logger.error('Failed to disconnect from PostgreSQL:', error);
      throw error;
    }
  }

  /**
   * Executes a raw SQL query using Drizzle's `sql` template literal.
   */
  public async query<T = unknown>(query: SQL | string, params?: any[]): Promise<T[]> {
    try {
      if (typeof query === 'string') {
        // For raw string queries, use the pool directly
        const result = await this.pool.query(query, params);
        return result.rows as T[];
      } else {
        // For Drizzle SQL objects
        const result = await this.db.execute(query);
        return result as T[]; // Drizzle's execute returns rows directly
      }
    } catch (error) {
      logger.error('PostgreSQL query failed:', error);
      throw error;
    }
  }

  /**
   * Inserts a new record into a specified table.
   */
  public async insert<TTable extends BaseSchema, TValues extends TTable['$inferInsert']>(
    table: TTable,
    values: TValues
  ): Promise<InferSelectModel<TTable>[]> {
    try {
      // Add common fields if not provided
      const now = new Date();
      const insertValues = {
        ...values,
        id: (values as IValtheronEntity).id || sql`gen_random_uuid()`, // Assume UUID generation in DB if not provided
        createdAt: (values as IValtheronEntity).createdAt || now,
        updatedAt: (values as IValtheronEntity).updatedAt || now,
      } as TValues;

      const result = await this.db
        .insert(table)
        .values(insertValues)
        .returning() // Return the inserted record(s)
        .execute();
      return result;
    } catch (error) {
      logger.error(`PostgreSQL insert into ${table.tableName} failed:`, error);
      throw error;
    }
  }

  /**
   * Updates records in a specified table.
   */
  public async update<TTable extends BaseSchema, TValues extends TTable['$inferInsert']>(
    table: TTable,
    values: TValues,
    where: SQL
  ): Promise<InferSelectModel<TTable>[]> {
    try {
      const updateValues = {
        ...values,
        updatedAt: new Date(), // Always update `updatedAt` on update
      } as TValues;

      const result = await this.db
        .update(table)
        .set(updateValues)
        .where(where)
        .returning()
        .execute();
      return result;
    } catch (error) {
      logger.error(`PostgreSQL update in ${table.tableName} failed:`, error);
      throw error;
    }
  }

  /**
   * Finds records in a specified table.
   */
  public async find<TTable extends BaseSchema>(
    table: TTable,
    where?: SQL,
    limit?: number,
    offset?: number
  ): Promise<InferSelectModel<TTable>[]> {
    try {
      let query = this.db.select().from(table).$dynamic(); // Use $dynamic for conditional clauses

      if (where) {
        query = query.where(where);
      }
      if (limit !== undefined) {
        query = query.limit(limit);
      }
      if (offset !== undefined) {
        query = query.offset(offset);
      }

      const result = await query.execute();
      return result;
    } catch (error) {
      logger.error(`PostgreSQL find in ${table.tableName} failed:`, error);
      throw error;
    }
  }

  /**
   * Finds a single record by its primary ID.
   */
  public async findById<TTable extends BaseSchema>(
    table: TTable,
    id: string
  ): Promise<InferSelectModel<TTable> | undefined> {
    try {
      const result = await this.db
        .select()
        .from(table)
        .where(eq((table as any).id, id)) // Assuming 'id' is always the primary key column name
        .limit(1)
        .execute();
      return result[0];
    } catch (error) {
      logger.error(`PostgreSQL findById in ${table.tableName} failed for ID ${id}:`, error);
      throw error;
    }
  }

  /**
   * Deletes records from a specified table.
   */
  public async delete<TTable extends BaseSchema>(
    table: TTable,
    where: SQL
  ): Promise<InferSelectModel<TTable>[]> {
    try {
      const result = await this.db
        .delete(table)
        .where(where)
        .returning()
        .execute();
      return result;
    } catch (error) {
      logger.error(`PostgreSQL delete from ${table.tableName} failed:`, error);
      throw error;
    }
  }

  // Transaction methods (simplified for example)
  public async beginTransaction(): Promise<PgDatabase<typeof schema>> {
    // Drizzle's transaction context is typically passed as a callback parameter
    // Here we return the Drizzle instance for direct transaction management
    logger.debug('Beginning PostgreSQL transaction...');
    return this.db;
  }

  public async commitTransaction(tx: any): Promise<void> {
    logger.debug('Committing PostgreSQL transaction...');
    // Drizzle handles commit implicitly when the transaction callback finishes
    // For explicit control, one might use `db.transaction()` and await its completion.
    // This method might be more abstract if using an ORM's transaction wrapper.
  }

  public async rollbackTransaction(tx: any): Promise<void> {
    logger.debug('Rolling back PostgreSQL transaction...');
    // Drizzle handles rollback implicitly if the transaction callback throws an error.
  }
}
```

### 3.4 Data Migration/Backfilling Script Snippet (`scripts/migrate-sqlite-to-postgres.ts`)

This script demonstrates reading data from SQLite and writing it to PostgreSQL.

```typescript
// scripts/migrate-sqlite-to-postgres.ts

import { drizzle as drizzlePg } from 'drizzle-orm/pg-core';
import { drizzle as drizzleSqlite } from 'drizzle-orm/sqlite-core';
import { Database } from 'better-sqlite3';
import { Pool } from 'pg';
import * as pgSchema from '../src/database/schema/agents'; // Import PostgreSQL schemas
import * as sqliteSchema from '../src/database/schema/agents'; // Import SQLite schemas (can be the same definition)
import { Agent, NewAgent } from '../src/database/schema/agents';
import { Logger } from '../src/utils/logger';
import { Config } from '../src/config';

const logger = new Logger('MigrationScript');

async function migrateAgents() {
  logger.info('Starting agent data migration from SQLite to PostgreSQL...');

  // 1. Initialize SQLite Database (Source)
  const sqliteDbPath = Config.get('SQLITE_DATABASE_PATH', './valtheron.sqlite');
  const sqliteClient = new Database(sqliteDbPath);
  const sqlite = drizzleSqlite(sqliteClient, { schema: sqliteSchema });
  logger.info(`Connected to SQLite database at: ${sqliteDbPath}`);

  // 2. Initialize PostgreSQL Database (Target)
  const pgPool = new Pool({
    connectionString: Config.get('DATABASE_URL'),
    max: 5, // Use a smaller pool for migration
    ssl: Config.get('NODE_ENV') === 'production' ? { rejectUnauthorized: false } : false,
  });
  const postgres = drizzlePg(pgPool, { schema: pgSchema });
  logger.info('Connected to PostgreSQL database.');

  try {
    // Ensure PostgreSQL table exists and is empty or ready for import
    // In a real scenario, you'd run Drizzle migrations on Postgres first.
    // For this example, we'll just try to insert.

    // 3. Fetch data from SQLite
    logger.info('Fetching agents from SQLite...');
    const sqliteAgentsData: Agent[] = await sqlite.select().from(sqliteSchema.sqliteAgents).execute();
    logger.info(`Found ${sqliteAgentsData.length} agents in SQLite.`);

    if (sqliteAgentsData.length === 0) {
      logger.warn('No agent data found in SQLite. Skipping migration for agents.');
      return;
    }

    // 4. Transform and Insert into PostgreSQL
    const batchSize = 100;
    let migratedCount = 0;
    let failedCount = 0;

    for (let i = 0; i < sqliteAgentsData.length; i += batchSize) {
      const batch = sqliteAgentsData.slice(i, i + batchSize);
      logger.info(`Processing batch ${Math.floor(i / batchSize) + 1}/${Math.ceil(sqliteAgentsData.length / batchSize)}...`);

      const insertPromises = batch.map(async (agentData) => {
        // Ensure data types are compatible and handle any transformations
        const newAgent: NewAgent = {
          id: agentData.id,
          name: agentData.name,
          description: agentData.description,
          status: agentData.status,
          config: agentData.config,
          isPublic: agentData.isPublic,
          createdAt: agentData.createdAt,
          updatedAt: agentData.updatedAt,
          createdByUserId: agentData.createdByUserId,
          lastModifiedByUserId: agentData.lastModifiedByUserId,
        };

        try {
          await postgres.insert(pgSchema.agents).values(newAgent).execute();
          migratedCount++;
        } catch (error) {
          logger.error(`Failed to migrate agent ${agentData.id}:`, error);
          failedCount++;
        }
      });
      await Promise.all(insertPromises);
    }

    // 5. Validation (as per original draft: "run math checks on counts")
    logger.info('Performing post-migration validation...');
    const pgAgentsCountResult = await postgres.select({ count: sql<number>`count(*)` }).from(pgSchema.agents).execute();
    const pgAgentsCount = pgAgentsCountResult[0].count;

    logger.info(`SQLite Agents Count: ${sqliteAgentsData.length}`);
    logger.info(`PostgreSQL Agents Count: ${pgAgentsCount}`);
    logger.info(`Successfully Migrated: ${migratedCount}`);
    logger.info(`Failed Migrations: ${failedCount}`);

    if (pgAgentsCount === sqliteAgentsData.length && failedCount === 0) {
      logger.success('Agent migration successful! All counts match.');
    } else {
      logger.error('Agent migration completed with discrepancies or failures. Manual review required.');
      process.exit(1); // Exit with error code
    }

  } catch (error) {
    logger.error('An unhandled error occurred during migration:', error);
    process.exit(1);
  } finally {
    // 6. Clean up connections
    sqliteClient.close();
    await pgPool.end();
    logger.info('Database connections closed.');
  }
}

migrateAgents();
```

### 3.5 Integrating with Express 5.1 (`src/server/index.ts` and `src/services/agentService.ts`)

We'll use dependency injection to provide the correct `IDatabaseAdapter` instance to our services.

```typescript
// src/server/index.ts (Snippet)

import express, { Request, Response, NextFunction } from 'express';
import { IDatabaseAdapter } from '../database/interfaces/IDatabaseAdapter';
import { PostgresAdapter } from '../database/adapters/PostgresAdapter';
import { SQLiteAdapter } from '../database/adapters/SQLiteAdapter'; // Assuming an SQLite adapter exists
import { Config } from '../config';
import { Logger } from '../utils/logger';
import { AgentService } from '../services/agentService'; // Our service that uses the adapter
import { agentRouter } from './routes/agentRoutes'; // Example API routes

const logger = new Logger('ExpressServer');
const app = express();

// --- Database Initialization ---
let dbAdapter: IDatabaseAdapter;

// Determine which adapter to use based on configuration
if (Config.get('USE_POSTGRES') === 'true') {
  dbAdapter = new PostgresAdapter();
  logger.info('Using PostgreSQL database adapter.');
} else {
  // Fallback to SQLite or for local development
  dbAdapter = new SQLiteAdapter(Config.get('SQLITE_DATABASE_PATH', './valtheron.sqlite'));
  logger.info('Using SQLite database adapter.');
}

// Connect to the database
dbAdapter.connect().catch(err => {
  logger.error('Failed to connect to the database, shutting down:', err);
  process.exit(1);
});

// Middleware to make the database adapter available to request handlers
// We can attach it to `res.locals` or `req.context` for type safety (if using custom Request types)
declare module 'express-serve-static-core' {
  interface Request {
    db: IDatabaseAdapter; // Augment Request type
  }
}

app.use((req: Request, res: Response, next: NextFunction) => {
  req.db = dbAdapter; // Attach the database adapter to the request object
  next();
});

// --- API Routes (Example) ---
// Initialize services with the database adapter
const agentService = new AgentService(dbAdapter);
app.use('/api/agents', agentRouter(agentService)); // Pass service to route factory

// ... other middleware and routes ...

// Graceful shutdown
process.on('SIGTERM', async () => {
  logger.info('SIGTERM signal received: closing HTTP server and database connection');
  await dbAdapter.disconnect();
  // ... close http server ...
  process.exit(0);
});

// Start the server
const PORT = Config.get('PORT', 3000);
app.listen(PORT, () => {
  logger.info(`Server running on port ${PORT}`);
});
```

```typescript
// src/services/agentService.ts

import { IDatabaseAdapter } from '../database/interfaces/IDatabaseAdapter';
import { Agent, NewAgent, agents } from '../database/schema/agents'; // Import Drizzle schema
import { eq } from 'drizzle-orm';
import { Logger } from '../utils/logger';

const logger = new Logger('AgentService');

/**
 * Service layer for managing Agent entities.
 * This service is decoupled from the specific database implementation
 * by depending on the IDatabaseAdapter interface.
 */
export class AgentService {
  constructor(private db: IDatabaseAdapter) {}

  /**
   * Creates a new agent.
   * @param newAgentData The data for the new agent.
   * @returns The created agent.
   */
  public async createAgent(newAgentData: NewAgent): Promise<Agent> {
    try {
      // Drizzle's insert method returns an array of inserted rows
      const [createdAgent] = await this.db.insert(agents, newAgentData);
      if (!createdAgent) {
        throw new Error('Failed to create agent: No data returned.');
      }
      logger.info(`Agent '${createdAgent.name}' created with ID: ${createdAgent.id}`);
      return createdAgent;
    } catch (error) {
      logger.error('Error creating agent:', error);
      throw new Error(`Could not create agent: ${(error as Error).message}`);
    }
  }

  /**
   * Retrieves an agent by its ID.
   * @param id The ID of the agent.
   * @returns The agent or undefined if not found.
   */
  public async getAgentById(id: string): Promise<Agent | undefined> {
    try {
      const agent = await this.db.findById(agents, id);
      logger.debug(`Fetched agent with ID ${id}: ${agent ? 'found' : 'not found'}`);
      return agent;
    } catch (error) {
      logger.error(`Error getting agent by ID ${id}:`, error);
      throw new Error(`Could not retrieve agent: ${(error as Error).message}`);
    }
  }

  /**
   * Retrieves all agents.
   * @returns An array of agents.
   */
  public async getAllAgents(): Promise<Agent[]> {
    try {
      const allAgents = await this.db.find(agents);
      logger.debug(`Fetched ${allAgents.length} agents.`);
      return allAgents;
    } catch (error) {
      logger.error('Error getting all agents:', error);
      throw new Error(`Could not retrieve agents: ${(error as Error).message}`);
    }
  }

  /**
   * Updates an existing agent.
   * @param id The ID of the agent to update.
   * @param updateData The partial data to update.
   * @returns The updated agent or undefined if not found.
   */
  public async updateAgent(id: string, updateData: Partial<NewAgent>): Promise<Agent | undefined> {
    try {
      const [updatedAgent] = await this.db.update(agents, updateData, eq(agents.id, id));
      if (!updatedAgent) {
        logger.warn(`Agent with ID ${id} not found for update.`);
        return undefined;
      }
      logger.info(`Agent '${updatedAgent.name}' (ID: ${updatedAgent.id}) updated.`);
      return updatedAgent;
    } catch (error) {
      logger.error(`Error updating agent ${id}:`, error);
      throw new Error(`Could not update agent: ${(error as Error).message}`);
    }
  }

  /**
   * Deletes an agent by its ID.
   * @param id The ID of the agent to delete.
   * @returns The deleted agent or undefined if not found.
   */
  public async deleteAgent(id: string): Promise<Agent | undefined> {
    try {
      const [deletedAgent] = await this.db.delete(agents, eq(agents.id, id));
      if (!deletedAgent) {
        logger.warn(`Agent with ID ${id} not found for deletion.`);
        return undefined;
      }
      logger.info(`Agent '${deletedAgent.name}' (ID: ${deletedAgent.id}) deleted.`);
      return deletedAgent;
    } catch (error) {
      logger.error(`Error deleting agent ${id}:`, error);
      throw new Error(`Could not delete agent: ${(error as Error).message}`);
    }
  }
}
```

## 4. Key Best Practices

### 4.1 Architectural Best Practices

*   **Database Abstraction Layer:** Always program to an interface (`IDatabaseAdapter`) rather than a concrete implementation. This enhances modularity, testability, and future-proofing.
*   **Dependency Injection:** Inject the `IDatabaseAdapter` into services or controllers. This allows for easy swapping of database implementations and improves unit testing.
*   **Drizzle ORM for Schema Management:** Use Drizzle ORM to define schemas and perform migrations. This provides strong type safety and consistency across different SQL dialects.
*   **Configuration Management:** Database connection strings, pool limits, and other sensitive settings **must** be managed via environment variables (e.g., `process.env.DATABASE_URL`) and not hardcoded.
*   **Separation of Concerns:** Keep data access logic (repositories/services) distinct from business logic and presentation layers (controllers/routes).
*   **Transaction Management:** Implement explicit transaction boundaries for operations involving multiple database modifications to maintain ACID properties.

### 4.2 Operational and Migration Best Practices

*   **Phased Rollout:** Never perform a "big bang" migration. Plan for a phased approach, starting with non-critical services or read-only access to the new database.
*   **Comprehensive Data Validation:** Implement rigorous checks during migration:
    *   **Row Counts:** Verify that the number of records matches between source and target.
    *   **Checksums/Hashes:** Compute hashes of critical data columns to ensure data integrity.
    *   **Sample Data Checks:** Manually verify a representative sample of migrated records.
    *   **Referential Integrity:** Ensure all foreign key relationships are correctly maintained.
*   **Idempotent Migration Scripts:** Design migration scripts to be re-runnable without causing duplicate data or errors.
*   **Robust Error Handling and Logging:** Implement detailed logging and error handling within migration scripts and the new database adapter to quickly identify and diagnose issues.
*   **Backup Strategy:** Perform a full backup of the existing SQLite database *before* starting any migration. Ensure the new PostgreSQL database has a robust backup and recovery plan in place from day one.
*   **Performance Testing:** Benchmark the new PostgreSQL setup under realistic load conditions to ensure it meets performance requirements.
*   **Monitoring and Alerting:** Set up comprehensive monitoring for the new database (connection health, query performance, error rates, resource utilization) and configure alerts for anomalies.
*   **Rollback Plan:** Have a clear, tested rollback strategy in case the migration encounters critical issues.

### 4.3 Security Best Practices

*   **Least Privilege:** Configure database users with the minimum necessary permissions. Application users should not have administrative privileges.
*   **Connection String Security:** Store database credentials securely using environment variables, a secrets manager (e.g., AWS Secrets Manager, HashiCorp Vault), or a `.env` file that is excluded from version control.
*   **Encryption in Transit (SSL/TLS):** Always enforce SSL/TLS for database connections, especially in production, to protect data from eavesdropping.
*   **Data at Rest Encryption:** For sensitive data, ensure the PostgreSQL instance or Cloud SQL service has data-at-rest encryption enabled.
*   **Audit Logging:** Leverage PostgreSQL's robust logging capabilities to capture all relevant database operations. This is critical for Valtheron's existing audit trailing requirements, ensuring all agent actions and user interactions with data are traceable.

### 4.4 Code Quality Best Practices

*   **Strict TypeScript Types:** Utilize TypeScript's full potential, especially with Drizzle ORM, for end-to-end type safety in database interactions.
*   **Clear, Concise Function Definitions:** Each function should have a single responsibility and be well-documented.
*   **Unit and Integration Tests:** Write comprehensive tests for both the database adapters and the migration scripts. Mock the `IDatabaseAdapter` for unit testing services.
*   **Meaningful Comments:** Provide comments for complex logic, design decisions, and public API interfaces.
*   **Consistent Code Style:** Adhere to the project's established code style guidelines (e.g., Prettier, ESLint).

By following these guidelines, contributors can ensure a smooth, secure, and performant transition to a scalable database solution, enhancing Valtheron's capabilities for future growth and enterprise adoption.