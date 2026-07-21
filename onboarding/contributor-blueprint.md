# Contributor Blueprint: Ensuring Production-Ready Code for Valtheron

## 1. Executive Summary

This document outlines the essential technical standards and best practices for contributing to the Valtheron Agentic Workspace. Our goal is to foster a culture of high-quality, production-ready code that is secure, maintainable, and performs optimally. This "Contributor Blueprint" serves as a comprehensive guide and an interactive checklist for all pull requests, ensuring that every contribution aligns with Valtheron's core principles and technical stack, including React 19, Express 5.1, TypeScript, SQLite, AES-256-GCM encryption, Multi-Factor Authentication (MFA), and robust audit trailing. Adhering to these guidelines is crucial for seamless integration, efficient code reviews, and the overall stability and security of the platform.

## 2. Conceptual Explanation

The Valtheron Agentic Workspace is built on a foundation of precision, security, and scalability. To uphold these tenets, every line of code contributed must meet stringent quality standards. This blueprint transitions our previous draft guidelines into a structured, actionable resource, intended to be integrated directly into our Pull Request (PR) template.

By adopting this approach, we aim to:

*   **Standardize Code Quality:** Establish clear, consistent benchmarks for code cleanliness, readability, and adherence to Valtheron's architecture.
*   **Enhance Maintainability:** Ensure that all new features and bug fixes are easy to understand, debug, and extend by other contributors.
*   **Strengthen Security Posture:** Embed security considerations from the outset, particularly concerning data handling, authentication, and authorization.
*   **Improve Collaboration:** Streamline the code review process by front-loading common checks, allowing reviewers to focus on architectural decisions and complex logic rather than basic compliance.
*   **Empower Contributors:** Provide clear guidance, reducing guesswork and accelerating the onboarding process for new team members.

This document is not merely a list of rules but a tutorial designed to explain the *why* behind each guideline, equipping you with the knowledge to write code that truly elevates the Valtheron platform.

## 3. Core Development Principles & Checklist Items

The following sections detail the essential principles and provide practical examples for each item in our contributor checklist.

---

### 3.1. Code Quality & Type Safety: Linting and TypeScript

Valtheron leverages TypeScript for robust type safety and comprehensive linting rules to enforce code style and catch potential issues early.

**Principle:** All code must pass linting checks and compile without TypeScript errors.

**Checklist Item:**
`[ ] Code passes `npm run lint` and `tsc --noEmit` without errors.`

**Explanation:**
*   `npm run lint`: Executes ESLint, which checks for stylistic issues, potential bugs, and adherence to our defined coding standards. It ensures consistency across the codebase.
*   `tsc --noEmit`: Runs the TypeScript compiler in a "no-emit" mode, meaning it performs type checking without generating JavaScript files. This is crucial for verifying type correctness and preventing runtime errors.

**Step-by-Step Example (TypeScript):**

Consider a utility function for our backend Express API.

```typescript
// src/server/utils/validation.ts

import { z } from 'zod'; // Assuming Zod for schema validation

/**
 * @interface UserInput
 * Defines the expected shape for new user creation data.
 */
export interface UserInput {
  username: string;
  email: string;
  passwordHash: string; // Storing hash, not plain password
}

/**
 * Zod schema for validating UserInput.
 * This ensures incoming data conforms to our expectations.
 */
export const userInputSchema = z.object({
  username: z.string().min(3, "Username must be at least 3 characters long."),
  email: z.string().email("Invalid email format."),
  passwordHash: z.string().min(60, "Password hash is invalid (expected bcrypt hash length)."), // Example length for bcrypt
});

/**
 * Validates user input against the defined schema.
 * @param data The raw data to validate.
 * @returns The validated data if successful, otherwise throws an error.
 */
export function validateUserInput(data: unknown): UserInput {
  try {
    return userInputSchema.parse(data);
  } catch (error) {
    if (error instanceof z.ZodError) {
      // Log detailed validation errors in development, provide generic in production
      console.error("User input validation failed:", error.errors);
      throw new Error("Invalid input provided for user creation.");
    }
    throw error; // Re-throw other unexpected errors
  }
}

// Example usage in an Express route handler:
// import { Request, Response, NextFunction } from 'express';
// import { validateUserInput } from '../utils/validation';

// export const createUserHandler = (req: Request, res: Response, next: NextFunction) => {
//   try {
//     const validatedData = validateUserInput(req.body);
//     // Proceed with creating user using validatedData
//     res.status(201).json({ message: 'User created successfully', user: validatedData.username });
//   } catch (error) {
//     next(error); // Pass error to Express error handling middleware
//   }
// };
```
**Comments on Example:**
*   **Type Safety:** `UserInput` interface clearly defines the expected data structure. `validateUserInput` ensures that the returned data *always* conforms to `UserInput`.
*   **Validation:** Uses Zod for robust runtime validation, complementing TypeScript's static checks.
*   **Error Handling:** Catches and re-throws specific validation errors, providing useful feedback.
*   **Linting:** This code adheres to typical linting rules (e.g., JSDoc comments, consistent formatting, no unused variables).

---

### 3.2. Styling Conventions: Tailwind CSS

Valtheron embraces Tailwind CSS for a utility-first approach to styling React components, promoting consistency and rapid UI development.

**Principle:** All CSS styling in React components must exclusively use Tailwind CSS utility classes. Direct `@import "tailwindcss";` in global CSS is the only allowed CSS import.

**Checklist Item:**
`[ ] All component styling uses Tailwind CSS utility classes. Global CSS only contains `@import "tailwindcss";`.`

**Explanation:**
*   **Utility-First:** Tailwind CSS provides a highly customizable set of low-level utility classes that can be composed directly in your JSX. This eliminates the need for writing custom CSS for most scenarios.
*   **Consistency:** By restricting styling to Tailwind classes, we ensure a unified look and feel across the application and avoid "snowflake" CSS.
*   **Performance:** Unused CSS is purged during the build process, leading to smaller bundle sizes.

**Step-by-Step Example (React 19 with Tailwind):**

```typescript jsx
// src/client/components/ui/PrimaryButton.tsx

import React from 'react';

/**
 * @interface PrimaryButtonProps
 * Defines the props for the PrimaryButton component.
 */
interface PrimaryButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  children: React.ReactNode;
  onClick?: () => void;
  isLoading?: boolean;
}

/**
 * PrimaryButton component for standard calls to action.
 * Utilizes Tailwind CSS for all its styling.
 */
export const PrimaryButton: React.FC<PrimaryButtonProps> = ({
  children,
  onClick,
  isLoading = false,
  className = '', // Allow external classes to be merged
  ...rest
}) => {
  return (
    <button
      type="button" // Default to button type
      onClick={onClick}
      disabled={isLoading}
      className={`
        px-6 py-3
        bg-indigo-600 hover:bg-indigo-700
        text-white font-semibold text-lg
        rounded-lg shadow-md
        focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:ring-opacity-75
        transition duration-150 ease-in-out
        ${isLoading ? 'opacity-75 cursor-not-allowed' : ''}
        ${className}
      `}
      {...rest}
    >
      {isLoading ? (
        <span className="flex items-center justify-center">
          <svg className="animate-spin -ml-1 mr-3 h-5 w-5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
            <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
            <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
          </svg>
          Loading...
        </span>
      ) : (
        children
      )}
    </button>
  );
};
```
**Comments on Example:**
*   **Tailwind Only:** All visual styling (`px-6`, `bg-indigo-600`, `rounded-lg`, etc.) is applied directly via Tailwind classes.
*   **Conditional Styling:** The `isLoading` prop dynamically adds `opacity-75` and `cursor-not-allowed` classes.
*   **Class Merging:** The `className` prop allows consumers to add or override classes without breaking the component's core styling.
*   **No Custom CSS:** There are no `<style>` tags or separate `.css` files for this component.

---

### 3.3. React Component Architecture: Naming and Structure

Consistent naming and a logical file structure are paramount for navigating the codebase efficiently.

**Principle:** React components must be named using PascalCase, and their corresponding files should reflect this naming convention and reside in a logical directory structure.

**Checklist Item:**
`[ ] React component names are PascalCase (e.g., `UserProfileCard`). File names match (e.g., `UserProfileCard.tsx`).`

**Explanation:**
*   **PascalCase for Components:** This is a universally accepted convention for React components, distinguishing them from regular HTML elements or JavaScript functions.
*   **File Naming:** Matching file names (e.g., `UserProfileCard.tsx`) makes it easy to locate components and ensures consistency.
*   **Folder Structure:** Organizing components into logical directories (e.g., `src/client/components/features/auth/LoginForm.tsx` or `src/client/components/ui/Button.tsx`) improves discoverability and maintainability.

**Step-by-Step Example (React 19 Component Structure):**

```
// src/client/
// └── components/
//     ├── features/                  // Feature-specific components
//     │   ├── auth/
//     │   │   ├── LoginForm.tsx
//     │   │   └── RegisterForm.tsx
//     │   └── settings/
//     │       └── UserProfileSettings.tsx
//     └── ui/                        // Reusable, generic UI components
//         ├── Button.tsx
//         ├── InputField.tsx
//         └── Modal.tsx

// src/client/components/features/auth/LoginForm.tsx
import React, { useState } from 'react';
import { InputField } from '../../ui/InputField'; // Relative import for UI component
import { PrimaryButton } from '../../ui/PrimaryButton'; // Relative import for UI component

/**
 * @interface LoginFormProps
 * Defines the props for the LoginForm component.
 */
interface LoginFormProps {
  onLoginSuccess: () => void;
}

/**
 * LoginForm component handles user authentication.
 */
export const LoginForm: React.FC<LoginFormProps> = ({ onLoginSuccess }) => {
  const [email, setEmail] = useState<string>('');
  const [password, setPassword] = useState<string>('');
  const [isLoading, setIsLoading] = useState<boolean>(false);

  const handleSubmit = async (event: React.FormEvent) => {
    event.preventDefault();
    setIsLoading(true);
    try {
      // Simulate API call
      await new Promise(resolve => setTimeout(resolve, 1500));
      console.log('Attempting login with:', { email, password });
      // In a real app, this would involve an actual API call and error handling
      onLoginSuccess();
    } catch (error) {
      console.error('Login failed:', error);
      // Handle error display
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="p-8 space-y-6 bg-white rounded-lg shadow-xl max-w-sm mx-auto">
      <h2 className="text-3xl font-bold text-center text-gray-800 mb-6">Login to Valtheron</h2>
      <InputField
        id="email"
        label="Email"
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        required
      />
      <InputField
        id="password"
        label="Password"
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        required
      />
      <PrimaryButton type="submit" isLoading={isLoading} className="w-full">
        {isLoading ? 'Signing In...' : 'Sign In'}
      </PrimaryButton>
    </form>
  );
};
```
**Comments on Example:**
*   **PascalCase:** `LoginForm`, `InputField`, `PrimaryButton` all follow PascalCase.
*   **File Naming:** `LoginForm.tsx` directly matches the component name.
*   **Logical Grouping:** Components are organized into `features` (for specific application functionalities) and `ui` (for generic, reusable UI elements).
*   **Props Typing:** `LoginFormProps` interface ensures clear and type-safe prop definitions.

---

### 3.4. Module Imports & Exports: Named Imports

Valtheron strictly enforces named imports and exports for all modules.

**Principle:** Always use named imports and exports. Default imports/exports are prohibited.

**Checklist Item:**
`[ ] All module imports and exports are named (e.g., `import { func } from './module';` and `export const func = ...;`). Default imports are not used.`

**Explanation:**
*   **Tree-Shaking:** Named exports allow bundlers (like Webpack or Rollup) to perform more effective tree-shaking, removing unused code and reducing bundle sizes.
*   **Clarity and Readability:** It's immediately clear what is being imported from a module. There's no ambiguity about the "default" export.
*   **Refactoring Safety:** Renaming an exported member requires updates at all import sites, which TypeScript and IDEs can easily track, preventing silent breakage.
*   **Consistency:** Enforces a single, predictable way to manage module dependencies across the entire project.

**Step-by-Step Example (TypeScript Module):**

```typescript
// src/server/services/userService.ts

import { db } from '../config/database'; // Named import for the database client
import { UserInput } from '../utils/validation'; // Named import for the interface
import { generateAuditTrail } from '../utils/audit'; // Named import for audit utility

/**
 * @interface User
 * Represents a user entity in the database.
 */
export interface User {
  id: string;
  username: string;
  email: string;
  passwordHash: string;
  createdAt: string;
  updatedAt: string;
}

/**
 * Creates a new user in the database.
 * @param userData The validated user input.
 * @returns The newly created user.
 */
export const createUser = async (userData: UserInput): Promise<User> => {
  const { username, email, passwordHash } = userData;
  const id = crypto.randomUUID(); // Node.js crypto module for UUID generation
  const now = new Date().toISOString();

  await db.run(
    `INSERT INTO users (id, username, email, passwordHash, createdAt, updatedAt)
     VALUES (?, ?, ?, ?, ?, ?)`,
    id, username, email, passwordHash, now, now
  );

  await generateAuditTrail('user_created', `User ${username} (${id}) created.`, 'SYSTEM'); // Example audit trail

  return { id, username, email, passwordHash, createdAt: now, updatedAt: now };
};

/**
 * Finds a user by their ID.
 * @param id The user's ID.
 * @returns The user if found, otherwise undefined.
 */
export const findUserById = async (id: string): Promise<User | undefined> => {
  const user = await db.get<User>(`SELECT * FROM users WHERE id = ?`, id);
  return user;
};

// More named exports for other user-related operations...
// export const updateUser = ...
// export const deleteUser = ...
```

```typescript
// src/server/routes/userRoutes.ts (Express 5.1 example)

import { Router, Request, Response, NextFunction } from 'express';
import { createUser, findUserById } from '../services/userService'; // Named imports for service functions
import { validateUserInput } from '../utils/validation'; // Named import for validation utility
import { authenticateToken } from '../middleware/authMiddleware'; // Named import for middleware

export const userRouter = Router(); // Named export for the router instance

// POST /api/users - Create a new user
userRouter.post('/', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const validatedData = validateUserInput(req.body);
    const newUser = await createUser(validatedData);
    res.status(201).json({ message: 'User created successfully', user: newUser });
  } catch (error) {
    next(error); // Pass error to global error handler
  }
});

// GET /api/users/:id - Get user by ID (requires authentication)
userRouter.get('/:id', authenticateToken, async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { id } = req.params;
    const user = await findUserById(id);
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    res.status(200).json(user);
  } catch (error) {
    next(error);
  }
});
```
**Comments on Example:**
*   **Named Exports:** `export interface User`, `export const createUser`, `export const findUserById` are all named exports.
*   **Named Imports:** `import { db } }`, `import { UserInput }` and `import { createUser, findUserById }` all use curly braces to explicitly import named members.
*   **Router Export:** The `userRouter` itself is a named export, allowing it to be imported and used by the main Express app.

---

### 3.5. TypeScript Type Management: Enums vs. Union Types

Understanding the nuances of TypeScript's type system is crucial for optimal bundle size and type safety.

**Principle:** Prefer string literal union types or `const` assertions over TypeScript `enum`s for simple sets of string or number constants, especially when only type-checking is needed. Avoid importing `enum`s solely for their type when a more efficient alternative exists.

**Checklist Item:**
`[ ] TypeScript `enum`s are used judiciously, primarily when runtime object generation is required. String literal union types are preferred for static type definitions.`

**Explanation:**
*   **TypeScript `enum`s:** These create a JavaScript object at runtime, which can increase bundle size. They are useful when you need a runtime lookup (e.g., `LogLevel[0]` gives `"INFO"`) or two-way mapping.
*   **String Literal Union Types:** `type Status = 'PENDING' | 'APPROVED' | 'REJECTED';` These are purely compile-time constructs. They are tree-shakeable, have no runtime footprint, and provide excellent type safety.
*   **`const` Assertions:** `export const UserRoles = { ADMIN: 'ADMIN', USER: 'USER' } as const;` allows you to create a runtime object whose *values* are also used as literal types, offering a good balance.

**Step-by-Step Example (TypeScript Type Management):**

**Scenario 1: Prefer Union Types for Static Type Checking**

If you only need to ensure a variable's value is one of a predefined set of strings, a union type is superior.

```typescript
// src/shared/types/common.ts

// ❌ Avoid for simple string sets (creates runtime JS object)
// export enum UserRoleEnum {
//   ADMIN = 'ADMIN',
//   EDITOR = 'EDITOR',
//   VIEWER = 'VIEWER',
// }

// ✅ Prefer string literal union type (no runtime footprint, excellent type safety)
export type UserRole = 'ADMIN' | 'EDITOR' | 'VIEWER';

// ✅ Alternative: const object with 'as const' (runtime object, but types are literals)
export const FeatureFlags = {
  NEW_DASHBOARD: 'newDashboard',
  ADVANCED_SEARCH: 'advancedSearch',
  MFA_ENFORCEMENT: 'mfaEnforcement',
} as const;

// Infer the type from the const object
export type FeatureFlag = typeof FeatureFlags[keyof typeof FeatureFlags];

// Example usage:
const currentUserRole: UserRole = 'ADMIN'; // Valid
// const invalidRole: UserRole = 'GUEST'; // Type error!

function checkFeature(flag: FeatureFlag): boolean {
  return flag === FeatureFlags.NEW_DASHBOARD; // Runtime usage
}

// checkFeature('nonExistentFlag'); // Type error!
```

**Scenario 2: When `enum` might be appropriate (rare in Valtheron, consider alternatives first)**

If you genuinely need a reverse mapping at runtime or specific numeric enum behaviors, then `enum` might be considered. However, for most cases, union types or `const` objects are better.

```typescript
// src/server/constants/httpStatusCodes.ts

// Example where enum might be considered if you need reverse mapping (e.g., HttpStatusCode[404] gives "NOT_FOUND")
// However, even here, a simple const object or map is often clearer and more explicit.
export enum HttpStatusCode {
  OK = 200,
  CREATED = 201,
  BAD_REQUEST = 400,
  UNAUTHORIZED = 401,
  FORBIDDEN = 403,
  NOT_FOUND = 404,
  INTERNAL_SERVER_ERROR = 500,
}

// Example usage in an Express handler:
// import { HttpStatusCode } from '../constants/httpStatusCodes';

// res.status(HttpStatusCode.NOT_FOUND).json({ message: 'Resource not found' });
// console.log(HttpStatusCode[404]); // Outputs "NOT_FOUND" - this is the runtime utility.
```
**Comments on Example:**
*   **Union Type Preference:** `UserRole` demonstrates the cleaner, more efficient way to define a set of allowed string values for type checking.
*   **`as const` for Runtime and Type:** `FeatureFlags` shows how to get both a runtime object and literal types from it, which is often a good alternative to `enum`.
*   **`enum` Rationale:** The `HttpStatusCode` example highlights the specific (and less common) scenario where an `enum`'s runtime reverse mapping might be useful. For Valtheron, such cases should be carefully justified.

---

### 3.6. Robust Data Handling: SQLite and API Interactions

SQLite is Valtheron's primary database. Proper interaction, especially regarding concurrency and transactions, is vital.

**Principle:** All SQLite interactions must be robust, handle potential locking issues gracefully, and utilize transactions for atomic operations. Consider security and auditability in every database interaction.

**Checklist Item:**
`[ ] SQLite interactions use transactions for atomic operations. Concurrency and locking issues are handled (e.g., retries, proper connection management). All sensitive data access is audited.`

**Explanation:**
*   **SQLite Locking:** SQLite is a file-based database. Concurrent writes from multiple processes or even multiple connections within the same process can lead to database locking, causing `SQLITE_BUSY` errors.
*   **Transactions:** Grouping related database operations into a single transaction (e.g., `BEGIN TRANSACTION; ... COMMIT;` or `ROLLBACK;`) ensures atomicity. If any part of the transaction fails, the entire transaction is rolled back, preventing data corruption.
*   **Concurrency Management:** In an Express.js context, each request might initiate a database operation. A dedicated database service layer can help manage connections, queue writes, or implement retry logic for transient locking issues.
*   **Security & Auditability:** All database write operations, especially those involving sensitive data or user actions, must be accompanied by audit trail entries. Data at rest (e.g., user secrets) must be encrypted (AES-256-GCM).

**Step-by-Step Example (Express 5.1 & SQLite Interaction):**

Assuming `src/server/config/database.ts` provides a `db` instance configured for `sqlite3` in `async`/`await` mode.

```typescript
// src/server/config/database.ts
import sqlite3 from 'sqlite3';
import { open, Database } from 'sqlite';

let db: Database;

export const initializeDatabase = async (): Promise<Database> => {
  if (db) return db; // Return existing connection if already initialized

  db = await open({
    filename: './valtheron.sqlite',
    driver: sqlite3.Database,
  });

  // Enable WAL mode for better concurrency (Write-Ahead Logging)
  await db.exec('PRAGMA journal_mode = WAL;');
  // Enable foreign key constraints
  await db.exec('PRAGMA foreign_keys = ON;');

  // Example schema creation (should be handled by migrations in production)
  await db.exec(`
    CREATE TABLE IF NOT EXISTS users (
      id TEXT PRIMARY KEY,
      username TEXT UNIQUE NOT NULL,
      email TEXT UNIQUE NOT NULL,
      passwordHash TEXT NOT NULL,
      mfaSecret TEXT, -- Encrypted MFA secret
      createdAt TEXT NOT NULL,
      updatedAt TEXT NOT NULL
    );

    CREATE TABLE IF NOT EXISTS audit_logs (
      id TEXT PRIMARY KEY,
      action TEXT NOT NULL,
      details TEXT NOT NULL,
      actor TEXT NOT NULL, -- User ID or 'SYSTEM'
      timestamp TEXT NOT NULL
    );
  `);
  console.log('SQLite database initialized and schemas checked.');
  return db;
};

// Named export for the database instance
export const getDb = (): Database => {
  if (!db) {
    throw new Error('Database not initialized. Call initializeDatabase() first.');
  }
  return db;
};
```

```typescript
// src/server/services/transactionService.ts
import { getDb } from '../config/database';
import { generateAuditTrail } from '../utils/audit';
import { encrypt, decrypt } from '../utils/encryption'; // AES-256-GCM utilities

// Define a type for a database transaction callback
type TransactionCallback<T> = (tx: Database) => Promise<T>;

/**
 * Executes a series of database operations within a single transaction.
 * Implements basic retry logic for SQLITE_BUSY errors.
 * @param callback The function containing database operations.
 * @param operationName A descriptive name for the operation for auditing.
 * @param actor The ID of the user performing the action, or 'SYSTEM'.
 * @param retries The number of times to retry on SQLITE_BUSY errors.
 * @returns The result of the callback function.
 */
export const runInTransaction = async <T>(
  callback: TransactionCallback<T>,
  operationName: string,
  actor: string,
  retries: number = 3
): Promise<T> => {
  const db = getDb();
  let attempts = 0;

  while (attempts <= retries) {
    try {
      await db.exec('BEGIN TRANSACTION;');
      const result = await callback(db);
      await db.exec('COMMIT;');
      await generateAuditTrail(operationName, `Transaction for ${operationName} succeeded.`, actor);
      return result;
    } catch (error: any) {
      await db.exec('ROLLBACK;'); // Always rollback on error
      if (error.code === 'SQLITE_BUSY' && attempts < retries) {
        console.warn(`SQLITE_BUSY encountered for ${operationName}. Retrying... (Attempt ${attempts + 1}/${retries})`);
        attempts++;
        await new Promise(resolve => setTimeout(resolve, 100 * attempts)); // Exponential backoff
        continue;
      }
      await generateAuditTrail(operationName, `Transaction for ${operationName} failed: ${error.message}.`, actor);
      throw error; // Re-throw if not SQLITE_BUSY or out of retries
    }
  }
  throw new Error(`Failed to complete transaction for ${operationName} after ${retries} retries due to SQLITE_BUSY.`);
};

// Example usage:
// src/server/services/authService.ts
// import { runInTransaction } from './transactionService';
// import { getDb } from '../config/database';
// import { UserInput } from '../utils/validation';
// import { User } from './userService';
// import { encrypt } from '../utils/encryption'; // For MFA secret

// export const registerUserWithMFA = async (userData: UserInput, mfaSecret: string, actorId: string): Promise<User> => {
//   return runInTransaction(async (tx) => {
//     const db = tx; // Use the transaction-specific database instance

//     const encryptedMfaSecret = encrypt(mfaSecret); // Encrypt MFA secret before storing

//     const id = crypto.randomUUID();
//     const now = new Date().toISOString();

//     await db.run(
//       `INSERT INTO users (id, username, email, passwordHash, mfaSecret, createdAt, updatedAt)
//        VALUES (?, ?, ?, ?, ?, ?, ?)`,
//       id, userData.username, userData.email, userData.passwordHash, encryptedMfaSecret, now, now
//     );

//     const newUser = await db.get<User>(`SELECT * FROM users WHERE id = ?`, id);
//     if (!newUser) throw new Error('Failed to retrieve new user after insertion.');

//     return newUser;
//   }, 'register_user_with_mfa', actorId);
// };
```
**Comments on Example:**
*   **WAL Mode:** `PRAGMA journal_mode = WAL;` significantly improves SQLite's concurrency for read-heavy workloads and reduces writer contention.
*   **Transaction Wrapper:** `runInTransaction` encapsulates the `BEGIN/COMMIT/ROLLBACK` logic, ensuring atomicity.
*   **Retry Logic:** Includes a basic retry mechanism with exponential backoff for `SQLITE_BUSY` errors, making the system more resilient.
*   **Audit Trailing:** `generateAuditTrail` is called on both success and failure within the transaction, demonstrating a commitment to auditability.
*   **Encryption:** The `mfaSecret` is explicitly encrypted using `encrypt()` before being stored, adhering to AES-256-GCM requirements for sensitive data.
*   **Dedicated `getDb`:** Ensures database is initialized and provides a single point of access.
*   **Foreign Keys:** `PRAGMA foreign_keys = ON;` is critical for data integrity.

---

### 3.7. Security & Auditability: General Principles

Valtheron's core mission includes robust security and transparency. These principles must permeate all contributions.

**Principle:** Every contribution must consider its impact on security (data protection, authentication, authorization) and ensure that all significant actions are appropriately audited.

**Checklist Item:**
`[ ] All new features or changes consider AES-256-GCM encryption for sensitive data, MFA implications, and generate appropriate audit trail entries for critical actions.`

**Explanation:**
*   **AES-256-GCM Encryption:** Any data deemed sensitive (e.g., API keys, MFA secrets, PII) must be encrypted at rest using AES-256-GCM. Ensure proper key management.
*   **Multi-Factor Authentication (MFA):** Changes to authentication flows or user settings must be compatible with or enhance our MFA implementation. Consider if an action requires re-authentication or a second factor.
*   **Audit Trailing:** Critical actions (user login/logout, data modification, security setting changes, failed authentication attempts, system events) must generate clear, immutable audit logs. These logs are essential for compliance, security monitoring, and incident response.

**Step-by-Step Example (Audit Trail Utility):**

```typescript
// src/server/utils/audit.ts
import { getDb } from '../config/database'; // Using named import
import { Database } from 'sqlite'; // For type hinting

/**
 * Generates a unique ID for audit logs.
 * In a production environment, consider a more robust ID generation strategy.
 */
const generateAuditId = (): string => crypto.randomUUID();

/**
 * Creates an audit trail entry in the database.
 * @param action The type of action performed (e.g., 'user_login', 'data_update').
 * @param details A detailed description of the action.
 * @param actor The ID of the user performing the action, or 'SYSTEM' for automated actions.
 */
export const generateAuditTrail = async (
  action: string,
  details: string,
  actor: string
): Promise<void> => {
  try {
    const db = getDb();
    const timestamp = new Date().toISOString();
    const id = generateAuditId();

    await db.run(
      `INSERT INTO audit_logs (id, action, details, actor, timestamp)
       VALUES (?, ?, ?, ?, ?)`,
      id, action, details, actor, timestamp
    );
  } catch (error) {
    // IMPORTANT: If audit logging fails, log to console and consider an emergency fallback
    console.error('CRITICAL: Failed to write audit log entry!', { action, details, actor, error });
    // In a real system, you might send an alert or write to a fallback file system log.
  }
};
```
**Comments on Example:**
*   **Named Export:** `export const generateAuditTrail` for clear import.
*   **Immutability:** Audit logs are append-only.
*   **Critical Error Handling:** Explicitly handles failures in audit logging, recognizing its critical nature.
*   **Contextual Details:** Requires `action`, `details`, and `actor` to provide a comprehensive record.

## 4. Key Best Practices Checklist for Pull Requests

Before submitting any Pull Request, please ensure all the following items have been thoroughly addressed. This checklist is designed to be directly integrated into our PR template.

*   [ ] **Code Quality & Type Safety:**
    *   [ ] All new and modified code passes `npm run lint` without errors.
    *   [ ] All new and modified TypeScript code compiles without `tsc --noEmit` errors.
    *   [ ] Type definitions are precise, and `any` is avoided unless absolutely necessary with a clear justification.
*   [ ] **Styling Conventions (React):**
    *   [ ] All React component styling is implemented exclusively using Tailwind CSS utility classes.
    *   [ ] No custom CSS files or inline `<style>` tags are used within components.
    *   [ ] Global CSS files (if any) only contain the `@import "tailwindcss";` directive and necessary base styles.
*   [ ] **React Component Architecture:**
    *   [ ] All React components are named using `PascalCase` (e.g., `UserProfileCard`).
    *   [ ] Component file names match their component name (e.g., `UserProfileCard.tsx`).
    *   [ ] Components are organized into logical directories (e.g., `features/`, `ui/`).
    *   [ ] Component props are explicitly typed using TypeScript interfaces.
*   [ ] **Module Imports & Exports:**
    *   [ ] All module imports are named (e.g., `import { func } from './module';`).
    *   [ ] All module exports are named (e.g., `export const func = ...;`).
    *   [ ] Default imports and exports are **not** used.
*   [ ] **TypeScript Type Management:**
    *   [ ] String literal union types (e.g., `type Status = 'ACTIVE' | 'INACTIVE';`) are preferred over `enum`s for simple sets of string or number constants.
    *   [ ] `const` objects with `as const` assertions are used when both runtime values and literal types are needed.
    *   [ ] `enum`s are used only when their runtime features (like reverse mapping) are explicitly required and justified.
*   [ ] **Robust Data Handling (SQLite & Express):**
    *   [ ] All database write operations are wrapped in transactions using `runInTransaction` or similar atomic constructs.
    *   [ ] Concurrency considerations for SQLite (e.g., `SQLITE_BUSY` errors) are handled with appropriate retry logic and `WAL` mode enabled.
    *   [ ] Sensitive data stored in SQLite is encrypted using AES-256-GCM before persistence and decrypted upon retrieval.
*   [ ] **Security & Auditability:**
    *   [ ] Any new sensitive data storage or transfer adheres to Valtheron's encryption standards (AES-256-GCM).
    *   [ ] Changes impacting user authentication or authorization are compatible with or enhance our MFA implementation.
    *   [ ] All critical user actions, system events, and security-relevant operations generate appropriate audit trail entries using `generateAuditTrail()`.
    *   [ ] Input validation (e.g., using Zod) is implemented on all API endpoints accepting user input.
*   [ ] **Documentation:**
    *   [ ] New components, functions, and complex logic are documented with JSDoc comments.
    *   [ ] Any significant architectural decisions or complex algorithms are explained in relevant `docs/` or code comments.

---

Thank you for your commitment to the Valtheron Agentic Workspace. Your adherence to these guidelines ensures we collectively build a secure, stable, and exceptional platform.