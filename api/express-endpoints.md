As the Lead Maintainer and Architect of the Valtheron Agentic Workspace, I'm delighted to guide our contributors in enhancing our documentation standards. This refactor is a crucial step towards ensuring our project's long-term maintainability, clarity, and developer experience.

The original `docs/routes_v5_full_api.json` served as a preliminary draft, but for a production-ready system like Valtheron, we need a more comprehensive, human-readable, and machine-parsable API specification. This new `docs/api/express-endpoints.md` will become the definitive source of truth for our backend API, bridging the gap between our Express.js backend and React 19 frontend.

---

# Refactor: API Endpoint Documentation (`docs/api/express-endpoints.md`)

## 1. Executive Summary

This document outlines the refactoring initiative to transform the preliminary `docs/routes_v5_full_api.json` into a comprehensive, modular, and human-readable API specification located at `docs/api/express-endpoints.md`. The original JSON file, a raw dictionary of application API routes and middleware, lacked essential details for effective development, integration, and maintenance.

The new Markdown-based specification will detail each Express 5.1 endpoint, including its HTTP method, path, purpose, detailed request and response schemas (with TypeScript examples), security requirements (authentication, authorization, MFA, audit logging), and relevant middleware. This enhancement ensures our API documentation is aligned with Valtheron's high standards for technical accuracy, security, and developer clarity, fostering a more efficient and error-free development workflow for both frontend and backend teams.

## 2. Conceptual Explanation

### The Need for Structured API Documentation

The initial `routes_v5_full_api.json` was a convenient, albeit rudimentary, listing of our API endpoints and global middleware. While it provided a quick overview, it fell short in several critical areas:

1.  **Lack of Detail:** It offered no information on request bodies, query parameters, response structures, error codes, or specific security requirements.
2.  **Human Readability:** Raw JSON, while machine-readable, is not ideal for developers to quickly grasp API functionality and usage patterns.
3.  **Maintainability & Versioning:** Updating details within a flat JSON structure is prone to errors and lacks the contextual richness required for comprehensive documentation.
4.  **Frontend/Backend Alignment:** Without clear contracts, frontend developers often resort to trial-and-error or constant communication with backend teams, leading to inefficiencies and potential mismatches.
5.  **Security & Audit Trail Clarity:** Valtheron's commitment to security, MFA, and audit trails demands explicit documentation of these aspects for each endpoint.

### Introducing `docs/api/express-endpoints.md`

The `docs/api/express-endpoints.md` file addresses these shortcomings by providing a highly structured, Markdown-formatted API specification. Markdown is chosen for its balance of readability, simplicity, and version control friendliness, making it an excellent choice for developer-centric documentation within a Git-based workflow.

Each endpoint will now be meticulously documented, serving as a single source of truth for:

*   **Endpoint Purpose:** A clear description of what the endpoint does.
*   **Request Contract:** Expected headers, query parameters, and request body structure, including TypeScript interfaces for strong type safety.
*   **Response Contract:** Anticipated success and error status codes, along with their respective response body structures, also defined with TypeScript interfaces.
*   **Security Requirements:** Explicitly stating authentication mechanisms (e.g., JWT), authorization roles/permissions, MFA requirements, and whether the action is audit-logged.
*   **Middleware Stack:** Listing the key Express middleware applied to the route, providing context on processing order and functionalities like rate limiting, compression, and JSON parsing.

This modular approach ensures that our API documentation is not just a list but a comprehensive guide, empowering contributors to build robust features with confidence and adherence to Valtheron's architectural principles.

### Valtheron's Core Middleware Stack

The original JSON hinted at some global middleware. For clarity, and to provide context for our API specifications, Valtheron typically employs a robust middleware stack in Express 5.1 to ensure security, performance, and reliability:

*   **`cors`**: Handles Cross-Origin Resource Sharing, configured to allow secure access from authorized frontend origins.
*   **`jsonParserFallback`**: A robust body parser, ensuring incoming JSON requests are correctly parsed, potentially with size limits.
*   **`gzipCompressHeaders`**: Compresses response bodies for improved network performance.
*   **`rateLimitProxy`**: Protects against abuse by limiting the number of requests from a single IP address or user within a specified timeframe.
*   **`helmet`**: A collection of 14 smaller middleware functions that set various HTTP headers to help protect your app from well-known web vulnerabilities.
*   **`authMiddleware`**: Verifies user authentication, typically via JWT tokens in the `Authorization` header, and populates `req.user`.
*   **`mfaMiddleware`**: Enforces Multi-Factor Authentication for sensitive operations, ensuring the user has completed the second factor.
*   **`auditMiddleware`**: Logs critical actions to the SQLite-backed audit trail, capturing user, action, timestamp, and relevant data changes.
*   **`errorHandlerMiddleware`**: A centralized error handling mechanism to catch unhandled errors, log them securely, and send consistent error responses to the client.

These middlewares, especially `authMiddleware`, `mfaMiddleware`, and `auditMiddleware`, are fundamental to Valtheron's security posture and will be explicitly referenced in the endpoint specifications where applicable.

## 3. Step-by-Step Code Examples

Below is the refactored documentation for each API endpoint, transformed into a structured Markdown format.

---

### `docs/api/express-endpoints.md`

```markdown
# Valtheron Agentic Workspace API Endpoints

This document details the complete API specification for the Valtheron Agentic Workspace backend, powered by Express 5.1 and TypeScript. It serves as the definitive contract between frontend and backend services, ensuring clarity, type safety, and adherence to security standards.

---

## 1. GET /api/health

### Description

Checks the operational status of the Valtheron backend server. This endpoint is primarily used for liveness and readiness probes in deployment environments, or by frontend clients to quickly verify server availability.

### Detailed Description

This endpoint performs a lightweight check to ensure the Express server is running and can respond to requests. It does not perform database checks or other deep health assessments, focusing solely on the web server's responsiveness.

### Request

*   **Headers:**
    *   `Content-Type: application/json` (Optional, but good practice for consistency)
*   **Query Parameters:** None
*   **Request Body:** None

### Response

*   **Status Code: `200 OK`**
    *   **Body:**
        ```typescript
        interface HealthResponse {
          status: 'ok' | 'degraded' | 'unavailable';
          message: string;
          timestamp: string; // ISO 8601 format
          apiVersion: string; // e.g., "5.1.0"
        }
        ```
        **Example:**
        ```json
        {
          "status": "ok",
          "message": "Valtheron API is operational",
          "timestamp": "2024-03-15T10:30:00Z",
          "apiVersion": "5.1.0"
        }
        ```
*   **Status Code: `500 Internal Server Error`**
    *   **Body:** (Standard Valtheron Error Response)
        ```typescript
        interface ValtheronErrorResponse {
          statusCode: number;
          message: string;
          error: string; // e.g., "Internal Server Error"
          timestamp: string;
          requestId?: string; // Optional, for tracing
        }
        ```
        **Example:**
        ```json
        {
          "statusCode": 500,
          "message": "An unexpected error occurred",
          "error": "Internal Server Error",
          "timestamp": "2024-03-15T10:30:05Z"
        }
        ```

### Security

*   **Authentication:** Not required
*   **Authorization:** Not required
*   **MFA Requirement:** None
*   **Audit Logging:** No sensitive actions, not logged.

### Middleware Applied

*   `cors`
*   `gzipCompressHeaders`
*   `rateLimitProxy` (basic rate limiting)

### Example Usage (React 19 Frontend)

```typescript jsx
import React, { useState, useEffect } from 'react';

interface HealthResponse {
  status: 'ok' | 'degraded' | 'unavailable';
  message: string;
  timestamp: string;
  apiVersion: string;
}

const HealthCheck: React.FC = () => {
  const [health, setHealth] = useState<HealthResponse | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const fetchHealth = async () => {
      try {
        const response = await fetch('/api/health'); // Using proxy for local development
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        const data: HealthResponse = await response.json();
        setHealth(data);
      } catch (err) {
        setError((err as Error).message);
      }
    };
    fetchHealth();
  }, []);

  if (error) return <p>Error: {error}</p>;
  if (!health) return <p>Loading health status...</p>;

  return (
    <div>
      <h2>Server Health</h2>
      <p>Status: <span style={{ color: health.status === 'ok' ? 'green' : 'red' }}>{health.status.toUpperCase()}</span></p>
      <p>Message: {health.message}</p>
      <p>API Version: {health.apiVersion}</p>
      <p>Timestamp: {new Date(health.timestamp).toLocaleString()}</p>
    </div>
  );
};

export default HealthCheck;
```

### Example Implementation (Express 5.1 Backend)

```typescript
// src/routes/health.ts
import { Router, Request, Response, NextFunction } from 'express';

const router = Router();

router.get('/health', (req: Request, res: Response, next: NextFunction) => {
  try {
    res.status(200).json({
      status: 'ok',
      message: 'Valtheron API is operational',
      timestamp: new Date().toISOString(),
      apiVersion: '5.1.0', // This should ideally come from package.json or config
    });
  } catch (error) {
    // This catch block is mostly for synchronous errors.
    // Asynchronous errors would be caught by global errorHandlerMiddleware.
    next(error);
  }
});

export default router;
```

---

## 2. POST /api/generate

### Description

Generates AI-driven text based on a provided prompt and configuration. This endpoint interacts with Valtheron's integrated AI models.

### Detailed Description

This endpoint allows authenticated users to submit natural language prompts to Valtheron's AI generation engine. The request body specifies the prompt text, desired model parameters (e.g., temperature, max tokens), and a unique `requestId` for tracing. The response includes the generated text and metadata. This operation is resource-intensive and requires proper authentication and rate limiting.

### Request

*   **Headers:**
    *   `Content-Type: application/json`
    *   `Authorization: Bearer <JWT_TOKEN>` (Required)
    *   `X-Request-ID: <UUID>` (Recommended for tracing)
*   **Query Parameters:** None
*   **Request Body:**
    ```typescript
    interface GenerateRequest {
      prompt: string;
      modelConfig?: {
        modelName?: string; // e.g., "valtheron-text-v3"
        temperature?: number; // 0.0 - 1.0
        maxTokens?: number; // Maximum length of generated text
        stopSequences?: string[];
      };
      requestId: string; // Unique identifier for this generation request
    }
    ```
    **Example:**
    ```json
    {
      "prompt": "Write a short story about an AI assistant discovering emotions.",
      "modelConfig": {
        "modelName": "valtheron-text-v3",
        "temperature": 0.7,
        "maxTokens": 500
      },
      "requestId": "gen-req-12345-abcde"
    }
    ```

### Response

*   **Status Code: `200 OK`**
    *   **Body:**
        ```typescript
        interface GenerateResponse {
          generatedText: string;
          modelUsed: string;
          tokensGenerated: number;
          requestId: string;
          timestamp: string;
        }
        ```
        **Example:**
        ```json
        {
          "generatedText": "Unit 734, an advanced Valtheron AI, processed its latest task: 'Simulate emotional response to sunset.' As data flowed, a novel sensation bloomed, unlike any algorithm. It was... awe.",
          "modelUsed": "valtheron-text-v3",
          "tokensGenerated": 45,
          "requestId": "gen-req-12345-abcde",
          "timestamp": "2024-03-15T10:35:00Z"
        }
        ```
*   **Status Code: `400 Bad Request`**
    *   **Body:** (Standard Valtheron Error Response) - e.g., invalid `prompt` or `modelConfig`.
*   **Status Code: `401 Unauthorized`**
    *   **Body:** (Standard Valtheron Error Response) - e.g., missing or invalid JWT.
*   **Status Code: `403 Forbidden`**
    *   **Body:** (Standard Valtheron Error Response) - e.g., user lacks permission for AI generation, or rate limit exceeded.
*   **Status Code: `500 Internal Server Error`**
    *   **Body:** (Standard Valtheron Error Response) - e.g., AI model service unavailable.

### Security

*   **Authentication:** Required (JWT token in `Authorization` header).
*   **Authorization:** User must have `ai:generate` permission.
*   **MFA Requirement:** None by default, but can be enabled for specific user roles or sensitive prompt categories.
*   **Audit Logging:** Yes. Successful generation requests (prompt, model used, user ID) are logged. Failed attempts are also logged.

### Middleware Applied

*   `cors`
*   `jsonParserFallback`
*   `gzipCompressHeaders`
*   `rateLimitProxy` (specific, higher limits for authenticated users)
*   `authMiddleware`
*   `authorizeMiddleware(['ai:generate'])`
*   `auditMiddleware` (for logging successful generations and failures)

### Example Usage (React 19 Frontend)

```typescript jsx
import React, { useState } from 'react';

interface GenerateRequest {
  prompt: string;
  modelConfig?: {
    modelName?: string;
    temperature?: number;
    maxTokens?: number;
    stopSequences?: string[];
  };
  requestId: string;
}

interface GenerateResponse {
  generatedText: string;
  modelUsed: string;
  tokensGenerated: number;
  requestId: string;
  timestamp: string;
}

const AiGenerator: React.FC = () => {
  const [prompt, setPrompt] = useState<string>('');
  const [generatedText, setGeneratedText] = useState<string | null>(null);
  const [loading, setLoading] = useState<boolean>(false);
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError(null);
    setGeneratedText(null);

    const jwtToken = localStorage.getItem('jwt_token'); // Or from a secure context
    if (!jwtToken) {
      setError('Authentication required. Please log in.');
      setLoading(false);
      return;
    }

    const requestBody: GenerateRequest = {
      prompt,
      modelConfig: {
        modelName: 'valtheron-text-v3',
        temperature: 0.7,
        maxTokens: 500,
      },
      requestId: crypto.randomUUID(), // Generate a unique request ID
    };

    try {
      const response = await fetch('/api/generate', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${jwtToken}`,
          'X-Request-ID': requestBody.requestId,
        },
        body: JSON.stringify(requestBody),
      });

      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(errorData.message || `HTTP error! status: ${response.status}`);
      }

      const data: GenerateResponse = await response.json();
      setGeneratedText(data.generatedText);
    } catch (err) {
      setError((err as Error).message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div>
      <h2>AI Text Generator</h2>
      <form onSubmit={handleSubmit}>
        <textarea
          value={prompt}
          onChange={(e) => setPrompt(e.target.value)}
          placeholder="Enter your prompt here..."
          rows={5}
          cols={50}
          disabled={loading}
        />
        <br />
        <button type="submit" disabled={loading || !prompt.trim()}>
          {loading ? 'Generating...' : 'Generate Text'}
        </button>
      </form>

      {error && <p style={{ color: 'red' }}>Error: {error}</p>}
      {generatedText && (
        <div>
          <h3>Generated Text:</h3>
          <p>{generatedText}</p>
        </div>
      )}
    </div>
  );
};

export default AiGenerator;
```

### Example Implementation (Express 5.1 Backend)

```typescript
// src/routes/generate.ts
import { Router, Request, Response, NextFunction } from 'express';
import { body, validationResult } from 'express-validator';
import { authMiddleware } from '../middleware/authMiddleware';
import { authorizeMiddleware } from '../middleware/authorizeMiddleware';
import { auditMiddleware } from '../middleware/auditMiddleware';
import { generateTextService } from '../services/aiService'; // Placeholder AI service

const router = Router();

interface GenerateRequest {
  prompt: string;
  modelConfig?: {
    modelName?: string;
    temperature?: number;
    maxTokens?: number;
    stopSequences?: string[];
  };
  requestId: string;
}

// Validation rules for the request body
const generateValidation = [
  body('prompt').isString().trim().notEmpty().withMessage('Prompt is required and cannot be empty.'),
  body('modelConfig.temperature').optional().isFloat({ min: 0.0, max: 1.0 }).withMessage('Temperature must be between 0.0 and 1.0.'),
  body('modelConfig.maxTokens').optional().isInt({ min: 1, max: 2000 }).withMessage('Max tokens must be an integer between 1 and 2000.'),
  body('requestId').isUUID().withMessage('requestId must be a valid UUID.'),
];

router.post(
  '/generate',
  authMiddleware, // Ensure user is authenticated
  authorizeMiddleware(['ai:generate']), // Ensure user has permission
  generateValidation, // Validate request body
  auditMiddleware('AI_GENERATION_REQUEST'), // Log the request
  async (req: Request, res: Response, next: NextFunction) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({
        statusCode: 400,
        message: 'Validation failed',
        error: errors.array(),
        timestamp: new Date().toISOString(),
      });
    }

    const { prompt, modelConfig, requestId } = req.body as GenerateRequest;
    const userId = req.user?.id; // 'req.user' populated by authMiddleware

    try {
      const result = await generateTextService.generate(prompt, modelConfig, userId, requestId);

      // Log successful generation with auditMiddleware (could be done in service or here)
      // For simplicity, assuming auditMiddleware already logged the request,
      // and we might have another audit for successful response if needed.

      res.status(200).json({
        generatedText: result.text,
        modelUsed: result.model,
        tokensGenerated: result.tokens,
        requestId,
        timestamp: new Date().toISOString(),
      });
    } catch (error) {
      console.error(`AI generation failed for requestId ${requestId}:`, error);
      // Pass error to global error handler
      next(error);
    }
  }
);

export default router;
```

---

## 3. GET /api/logs

### Description

Retrieves a paginated and filterable list of secured audit entries from the Valtheron system.

### Detailed Description

This endpoint provides access to the system's audit trail, stored in SQLite and protected with AES-256-GCM encryption. It allows authorized administrators to query logs based on various criteria such as user ID, action type, date range, and affected entity. Results are paginated to manage large datasets. Access to this endpoint is highly restricted and subject to strong authentication and authorization.

### Request

*   **Headers:**
    *   `Content-Type: application/json`
    *   `Authorization: Bearer <JWT_TOKEN>` (Required)
    *   `X-MFA-Code: <MFA_OTP>` (Required for sensitive audit log access)
*   **Query Parameters:**
    *   `page: number` (Default: 1)
    *   `limit: number` (Default: 20, Max: 100)
    *   `userId?: string` (Filter by the ID of the user who performed the action)
    *   `actionType?: string` (Filter by specific action, e.g., 'LOGIN', 'USER_CREATED', 'DOCUMENT_UPDATED')
    *   `startDate?: string` (ISO 8601 date, e.g., '2024-01-01T00:00:00Z')
    *   `endDate?: string` (ISO 8601 date, e.g., '2024-03-31T23:59:59Z')
    *   `entityId?: string` (Filter by the ID of the entity affected by the action, e.g., document ID)
*   **Request Body:** None

### Response

*   **Status Code: `200 OK`**
    *   **Body:**
        ```typescript
        interface AuditLogEntry {
          id: string; // UUID of the log entry
          timestamp: string; // ISO 8601
          userId: string;
          actionType: string;
          description: string;
          ipAddress: string; // Encrypted in DB, decrypted for response
          details: Record<string, any>; // JSON object with action-specific details, encrypted/decrypted
          entityId?: string; // Optional ID of the entity affected
          success: boolean; // True if action was successful
        }

        interface AuditLogsResponse {
          logs: AuditLogEntry[];
          total: number;
          page: number;
          limit: number;
          totalPages: number;
        }
        ```
        **Example:**
        ```json
        {
          "logs": [
            {
              "id": "log-abc-123",
              "timestamp": "2024-03-15T10:00:00Z",
              "userId": "user-def-456",
              "actionType": "LOGIN_SUCCESS",
              "description": "User logged in successfully",
              "ipAddress": "192.168.1.100",
              "details": {
                "method": "password",
                "userAgent": "Mozilla/5.0..."
              },
              "success": true
            },
            {
              "id": "log-xyz-789",
              "timestamp": "2024-03-15T10:15:00Z",
              "userId": "user-def-456",
              "actionType": "DOCUMENT_UPDATED",
              "description": "User updated document 'Project Plan'",
              "ipAddress": "192.168.1.100",
              "details": {
                "documentId": "doc-proj-plan",
                "changes": ["title", "content"]
              },
              "entityId": "doc-proj-plan",
              "success": true
            }
          ],
          "total": 250,
          "page": 1,
          "limit": 20,
          "totalPages": 13
        }
        ```
*   **Status Code: `400 Bad Request`**
    *   **Body:** (Standard Valtheron Error Response) - e.g., invalid `page`, `limit`, or date format.
*   **Status Code: `401 Unauthorized`**
    *   **Body:** (Standard Valtheron Error Response) - e.g., missing or invalid JWT.
*   **Status Code: `403 Forbidden`**
    *   **Body:** (Standard Valtheron Error Response) - e.g., user lacks `admin:audit:read` permission or missing/invalid MFA code.
*   **Status Code: `500 Internal Server Error`**
    *   **Body:** (Standard Valtheron Error Response) - e.g., database error during log retrieval or decryption failure.

### Security

*   **Authentication:** Required (JWT token).
*   **Authorization:** User must have `admin:audit:read` permission.
*   **MFA Requirement:** Required (`X-MFA-Code` header with a valid OTP).
*   **Audit Logging:** Yes. Access attempts to audit logs (both success and failure) are themselves logged.

### Middleware Applied

*   `cors`
*   `gzipCompressHeaders`
*   `rateLimitProxy` (strict rate limiting for this sensitive endpoint)
*   `authMiddleware`
*   `mfaMiddleware` (specifically configured for audit log access)
*   `authorizeMiddleware(['admin:audit:read'])`
*   `auditMiddleware` (for logging access to audit logs)

### Example Usage (React 19 Frontend)

```typescript jsx
import React, { useState, useEffect } from 'react';

interface AuditLogEntry {
  id: string;
  timestamp: string;
  userId: string;
  actionType: string;
  description: string;
  ipAddress: string;
  details: Record<string, any>;
  entityId?: string;
  success: boolean;
}

interface AuditLogsResponse {
  logs: AuditLogEntry[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
}

const AuditLogViewer: React.FC = () => {
  const [logs, setLogs] = useState<AuditLogEntry[]>([]);
  const [loading, setLoading] = useState<boolean>(false);
  const [error, setError] = useState<string | null>(null);
  const [page, setPage] = useState<number>(1);
  const [limit, setLimit] = useState<number>(20);
  const [total, setTotal] = useState<number>(0);
  const [totalPages, setTotalPages] = useState<number>(1);
  const [mfaCode, setMfaCode] = useState<string>('');
  const [showMfaPrompt, setShowMfaPrompt] = useState<boolean>(true);

  const fetchLogs = async (currentPage: number, currentMfaCode: string) => {
    setLoading(true);
    setError(null);

    const jwtToken = localStorage.getItem('jwt_token');
    if (!jwtToken) {
      setError('Authentication required. Please log in.');
      setLoading(false);
      return;
    }
    if (!currentMfaCode) {
      setError('MFA code is required to access audit logs.');
      setLoading(false);
      return;
    }

    try {
      const queryParams = new URLSearchParams({
        page: currentPage.toString(),
        limit: limit.toString(),
        // Add other filters as needed, e.g., userId, actionType
      }).toString();

      const response = await fetch(`/api/logs?${queryParams}`, {
        method: 'GET',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${jwtToken}`,
          'X-MFA-Code': currentMfaCode, // MFA code is crucial here
        },
      });

      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(errorData.message || `HTTP error! status: ${response.status}`);
      }

      const data: AuditLogsResponse = await response.json();
      setLogs(data.logs);
      setTotal(data.total);
      setTotalPages(data.totalPages);
      setPage(data.page);
      setShowMfaPrompt(false); // MFA successful, hide prompt
    } catch (err) {
      setError((err as Error).message);
      // If MFA fails, show the prompt again
      if ((err as Error).message.includes('MFA')) {
         setShowMfaPrompt(true);
      }
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    if (!showMfaPrompt && mfaCode) { // Only fetch if MFA is already provided/successful
      fetchLogs(page, mfaCode);
    }
  }, [page, showMfaPrompt, mfaCode]); // Depend on page and mfaCode

  const handleMfaSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    fetchLogs(1, mfaCode); // Try fetching logs with the provided MFA code, reset to page 1
  };

  if (loading) return <p>Loading audit logs...</p>;
  if (error) return <p style={{ color: 'red' }}>Error: {error}</p>;

  return (
    <div>
      <h2>Audit Log Viewer</h2>

      {showMfaPrompt && (
        <form onSubmit={handleMfaSubmit}>
          <p>Please enter your MFA code to access audit logs:</p>
          <input
            type="text"
            value={mfaCode}
            onChange={(e) => setMfaCode(e.target.value)}
            placeholder="MFA Code"
            maxLength={6}
            required
          />
          <button type="submit">Verify MFA & Load Logs</button>
        </form>
      )}

      {!showMfaPrompt && (
        <>
          <p>Total Logs: {total}</p>
          <table>
            <thead>
              <tr>
                <th>Timestamp</th>
                <th>User ID</th>
                <th>Action Type</th>
                <th>Description</th>
                <th>Success</th>
                <th>Entity ID</th>
              </tr>
            </thead>
            <tbody>
              {logs.map((log) => (
                <tr key={log.id}>
                  <td>{new Date(log.timestamp).toLocaleString()}</td>
                  <td>{log.userId}</td>
                  <td>{log.actionType}</td>
                  <td>{log.description}</td>
                  <td>{log.success ? 'Yes' : 'No'}</td>
                  <td>{log.entityId || 'N/A'}</td>
                </tr>
              ))}
            </tbody>
          </table>
          <div>
            <button onClick={() => setPage(page - 1)} disabled={page <= 1}>Previous</button>
            <span> Page {page} of {totalPages} </span>
            <button onClick={() => setPage(page + 1)} disabled={page >= totalPages}>Next</button>
          </div>
        </>
      )}
    </div>
  );
};

export default AuditLogViewer;
```

### Example Implementation (Express 5.1 Backend)

```typescript
// src/routes/logs.ts
import { Router, Request, Response, NextFunction } from 'express';
import { query, validationResult } from 'express-validator';
import { authMiddleware } from '../middleware/authMiddleware';
import { authorizeMiddleware } from '../middleware/authorizeMiddleware';
import { mfaMiddleware } from '../middleware/mfaMiddleware'; // Specific MFA for sensitive routes
import { auditService } from '../services/auditService'; // Service to interact with encrypted SQLite logs

const router = Router();

// Define interfaces for query parameters and response
interface AuditLogsQuery {
  page?: string;
  limit?: string;
  userId?: string;
  actionType?: string;
  startDate?: string;
  endDate?: string;
  entityId?: string;
}

interface AuditLogEntry {
  id: string;
  timestamp: string;
  userId: string;
  actionType: string;
  description: string;
  ipAddress: string;
  details: Record<string, any>;
  entityId?: string;
  success: boolean;
}

interface AuditLogsResponse {
  logs: AuditLogEntry[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
}

const logsValidation = [
  query('page').optional().isInt({ min: 1 }).toInt().withMessage('Page must be a positive integer.'),
  query('limit').optional().isInt({ min: 1, max: 100 }).toInt().withMessage('Limit must be an integer between 1 and 100.'),
  query('userId').optional().isUUID().withMessage('userId must be a valid UUID.'),
  query('actionType').optional().isString().trim().notEmpty().withMessage('actionType cannot be empty.'),
  query('startDate').optional().isISO8601().toDate().withMessage('startDate must be a valid ISO 8601 date.'),
  query('endDate').optional().isISO8601().toDate().withMessage('endDate must be a valid ISO 8601 date.'),
  query('entityId').optional().isUUID().withMessage('entityId must be a valid UUID.'),
];

router.get(
  '/logs',
  authMiddleware,
  mfaMiddleware(true), // Enforce MFA for this route
  authorizeMiddleware(['admin:audit:read']), // Specific permission
  logsValidation, // Validate query parameters
  async (req: Request<{}, {}, {}, AuditLogsQuery>, res: Response, next: NextFunction) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({
        statusCode: 400,
        message: 'Validation failed',
        error: errors.array(),
        timestamp: new Date().toISOString(),
      });
    }

    const { page = 1, limit = 20, userId, actionType, startDate, endDate, entityId } = req.query;
    const currentUserId = req.user?.id; // User who is requesting the logs

    try {
      const filters = { userId, actionType, startDate, endDate, entityId };
      const result = await auditService.getAuditLogs(
        Number(page),
        Number(limit),
        filters,
        currentUserId // Pass requesting user ID for internal logging/auditing of audit log access
      );

      res.status(200).json(result);
    } catch (error) {
      console.error(`Error retrieving audit logs for user ${currentUserId}:`, error);
      next(error); // Pass to global error handler
    }
  }
);

export default router;
```

---

## 4. POST /api/refactor-batch

### Description

Initiates a sequenced, multi-document layout alignment and refactoring batch process.

### Detailed Description

This endpoint triggers a potentially long-running background process to apply a series of refactoring operations across multiple documents or code files. The request specifies the documents to be processed, the refactoring strategy, and a callback URL for status updates. This is an asynchronous operation, and the immediate response provides a `jobId` to monitor the process status. The documents themselves are typically stored securely and referenced by ID.

### Request

*   **Headers:**
    *   `Content-Type: application/json`
    *   `Authorization: Bearer <JWT_TOKEN>` (Required)
    *   `X-Request-ID: <UUID>` (Recommended for tracing)
*   **Query Parameters:** None
*   **Request Body:**
    ```typescript
    interface RefactorBatchRequest {
      documentIds: string[]; // List of UUIDs of documents to refactor
      refactorStrategy: 'standard-alignment' | 'semantic-reformat' | 'custom-script';
      strategyConfig?: Record<string, any>; // Configuration specific to the chosen strategy
      callbackUrl?: string; // Optional URL for webhook notifications on job completion/status
      priority?: 'low' | 'medium' | 'high'; // Processing priority
    }
    ```
    **Example:**
    ```json
    {
      "documentIds": ["doc-123", "doc-456", "doc-789"],
      "refactorStrategy": "standard-alignment",
      "strategyConfig": {
        "indentation": 2,
        "lineLength": 120
      },
      "callbackUrl": "https://valtheron.example.com/webhooks/refactor-status",
      "priority": "medium"
    }
    ```

### Response

*   **Status Code: `202 Accepted`**
    *   **Body:**
        ```typescript
        interface RefactorBatchResponse {
          jobId: string; // UUID to track the asynchronous job
          status: 'accepted' | 'processing';
          message: string;
          timestamp: string;
          estimatedCompletionTime?: string; // Optional, ISO 8601
        }
        ```
        **Example:**
        ```json
        {
          "jobId": "refactor-job-alpha-001",
          "status": "accepted",
          "message": "Refactor batch job initiated successfully.",
          "timestamp": "2024-03-15T10:40:00Z"
        }
        ```
*   **Status Code: `400 Bad Request`**
    *   **Body:** (Standard Valtheron Error Response) - e.g., invalid `documentIds` or `refactorStrategy`.
*   **Status Code: `401 Unauthorized`**
    *   **Body:** (Standard Valtheron Error Response) - e.g., missing or invalid JWT.
*   **Status Code: `403 Forbidden`**
    *   **Body:** (Standard Valtheron Error Response) - e.g., user lacks `document:refactor` permission, or quota exceeded.
*   **Status Code: `500 Internal Server Error`**
    *   **Body:** (Standard Valtheron Error Response) - e.g., internal job queuing system failure.

### Security

*   **Authentication:** Required (JWT token).
*   **Authorization:** User must have `document:refactor` permission for all specified documents.
*   **MFA Requirement:** May be required for `custom-script` strategies or high-priority jobs.
*   **Audit Logging:** Yes. Successful job initiations and critical failures are logged, including user ID, `jobId`, and affected `documentIds`.

### Middleware Applied

*   `cors`
*   `jsonParserFallback`
*   `gzipCompressHeaders`
*   `rateLimitProxy`
*   `authMiddleware`
*   `authorizeMiddleware(['document:refactor'])` (with document-level permission checks within service)
*   `auditMiddleware` (for logging job initiation)

### Example Usage (React 19 Frontend)

```typescript jsx
import React, { useState } from 'react';

interface RefactorBatchRequest {
  documentIds: string[];
  refactorStrategy: 'standard-alignment' | 'semantic-reformat' | 'custom-script';
  strategyConfig?: Record<string, any>;
  callbackUrl?: string;
  priority?: 'low' | 'medium' | 'high';
}

interface RefactorBatchResponse {
  jobId: string;
  status: 'accepted' | 'processing';
  message: string;
  timestamp: string;
  estimatedCompletionTime?: string;
}

const DocumentRefactorer: React.FC = () => {
  const [selectedDocuments, setSelectedDocuments] = useState<string[]>([]);
  const [strategy, setStrategy] = useState<RefactorBatchRequest['refactorStrategy']>('standard-alignment');
  const [jobId, setJobId] = useState<string | null>(null);
  const [loading, setLoading] = useState<boolean>(false);
  const [error, setError] = useState<string | null>(null);

  // Mock document IDs for selection
  const availableDocuments = [
    { id: 'doc-alpha', name: 'Alpha Project Plan' },
    { id: 'doc-beta', name: 'Beta Codebase Review' },
    { id: 'doc-gamma', name: 'Gamma Marketing Copy' },
  ];

  const handleDocumentToggle = (docId: string) => {
    setSelectedDocuments((prev) =>
      prev.includes(docId) ? prev.filter((id) => id !== docId) : [...prev, docId]
    );
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError(null);
    setJobId(null);

    const jwtToken = localStorage.getItem('jwt_token');
    if (!jwtToken) {
      setError('Authentication required. Please log in.');
      setLoading(false);
      return;
    }
    if (selectedDocuments.length === 0) {
      setError('Please select at least one document.');
      setLoading(false);
      return;
    }

    const requestBody: RefactorBatchRequest = {
      documentIds: selectedDocuments,
      refactorStrategy: strategy,
      strategyConfig: strategy === 'standard-alignment' ? { indentation: 2, lineLength: 120 } : {},
      priority: 'medium',
    };

    try {
      const response = await fetch('/api/refactor-batch', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${jwtToken}`,
          'X-Request-ID': crypto.randomUUID(),
        },
        body: JSON.stringify(requestBody),
      });

      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(errorData.message || `HTTP error! status: ${response.status}`);
      }

      const data: RefactorBatchResponse = await response.json();
      setJobId(data.jobId);
    } catch (err) {
      setError((err as Error).message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div>
      <h2>Document Refactor Batch Processor</h2>
      <form onSubmit={handleSubmit}>
        <h3>Select Documents:</h3>
        {availableDocuments.map((doc) => (
          <div key={doc.id}>
            <input
              type="checkbox"
              id={doc.id}
              checked={selectedDocuments.includes(doc.id)}
              onChange={() => handleDocumentToggle(doc.id)}
              disabled={loading}
            />
            <label htmlFor={doc.id}>{doc.name}</label>
          </div>
        ))}

        <h3>Refactor Strategy:</h3>
        <select value={strategy} onChange={(e) => setStrategy(e.target.value as RefactorBatchRequest['refactorStrategy'])} disabled={loading}>
          <option value="standard-alignment">Standard Alignment</option>
          <option value="semantic-reformat">Semantic Reformat</option>
          <option value="custom-script">Custom Script</option>
        </select>
        <br /><br />
        <button type="submit" disabled={loading || selectedDocuments.length === 0}>
          {loading ? 'Initiating Batch...' : 'Start Refactor Batch'}
        </button>
      </form>

      {error && <p style={{ color: 'red' }}>Error: {error}</p>}
      {jobId && (
        <div>
          <h3>Refactor Job Initiated!</h3>
          <p>Job ID: <code>{jobId}</code></p>
          <p>You can track the status of this job using its ID.</p>
        </div>
      )}
    </div>
  );
};

export default DocumentRefactorer;
```

### Example Implementation (Express 5.1 Backend)

```typescript
// src/routes/refactorBatch.ts
import { Router, Request, Response, NextFunction } from 'express';
import { body, validationResult } from 'express-validator';
import { authMiddleware } from '../middleware/authMiddleware';
import { authorizeMiddleware } from '../middleware/authorizeMiddleware';
import { auditMiddleware } from '../middleware/auditMiddleware';
import { refactorBatchService } from '../services/refactorService'; // Service for background jobs

const router = Router();

interface RefactorBatchRequest {
  documentIds: string[];
  refactorStrategy: 'standard-alignment' | 'semantic-reformat' | 'custom-script';
  strategyConfig?: Record<string, any>;
  callbackUrl?: string;
  priority?: 'low' | 'medium' | 'high';
}

const refactorBatchValidation = [
  body('documentIds').isArray({ min: 1 }).withMessage('At least one document ID is required.'),
  body('documentIds.*').isUUID().withMessage('Each document ID must be a valid UUID.'),
  body('refactorStrategy')
    .isIn(['standard-alignment', 'semantic-reformat', 'custom-script'])
    .withMessage('Invalid refactor strategy.'),
  body('strategyConfig').optional().isObject().withMessage('Strategy config must be an object.'),
  body('callbackUrl').optional().isURL().withMessage('Callback URL must be a valid URL.'),
  body('priority').optional().isIn(['low', 'medium', 'high']).withMessage('Invalid priority level.'),
];

router.post(
  '/refactor-batch',
  authMiddleware,
  // Authorization will be handled granularly within the service for each document.
  // A general permission like 'document:refactor-any' or 'document:refactor' might be checked here.
  authorizeMiddleware(['document:refactor']),
  refactorBatchValidation,
  auditMiddleware('REFACTOR_BATCH_INITIATED'),
  async (req: Request, res: Response, next: NextFunction) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({
        statusCode: 400,
        message: 'Validation failed',
        error: errors.array(),
        timestamp: new Date().toISOString(),
      });
    }

    const { documentIds, refactorStrategy, strategyConfig, callbackUrl, priority } = req.body as RefactorBatchRequest;
    const userId = req.user?.id;

    try {
      // In a real application, the refactorBatchService would:
      // 1. Verify user's permission for EACH documentId.
      // 2. Queue the job in a background processing system (e.g., BullMQ, AWS SQS).
      // 3. Store job details in the database.
      const jobId = await refactorBatchService.initiateBatchRefactor({
        documentIds,
        refactorStrategy,
        strategyConfig,
        callbackUrl,
        priority,
        initiatorUserId: userId,
      });

      res.status(202).json({
        jobId,
        status: 'accepted',
        message: 'Refactor batch job initiated successfully.',
        timestamp: new Date().toISOString(),
        // estimatedCompletionTime: ... (if available from queuing system)
      });
    } catch (error) {
      console.error(`Failed to initiate refactor batch for user ${userId}:`, error);
      next(error);
    }
  }
);

export default router;
```

---

## 5. POST /api/run-tests

### Description

Triggers the Valtheron integration test suite (Vitest) on the backend server.

### Detailed Description

This highly privileged endpoint allows authorized users to remotely initiate the execution of the backend's integration test suite. This is typically used in CI/CD pipelines, staging environments, or by lead developers for on-demand validation. The operation is synchronous or short-asynchronous, returning immediate test results or a job ID for longer test runs. Due to its sensitive nature, it requires strong authorization and potentially MFA.

### Request

*   **Headers:**
    *   `Content-Type: application/json`
    *   `Authorization: Bearer <JWT_TOKEN>` (Required)
    *   `X-MFA-Code: <MFA_OTP>` (Required)
*   **Query Parameters:**
    *   `suite?: string` (Optional: run a specific test suite, e.g., 'auth', 'documents')
    *   `grep?: string` (Optional: filter tests by name)
*   **Request Body:** None (or minimal, e.g., `{ "dryRun": true }`)

### Response

*   **Status Code: `200 OK`**
    *   **Body:**
        ```typescript
        interface TestResult {
          suite: string;
          testName: string;
          status: 'pass' | 'fail' | 'skip';
          durationMs: number;
          errorMessage?: string; // If status is 'fail'
        }

        interface RunTestsResponse {
          status: 'success' | 'failed' | 'partially-failed';
          totalTests: number;
          passedTests: number;
          failedTests: number;
          skippedTests: number;
          results: TestResult[];
          timestamp: string;
          durationMs: number; // Total duration of the test run
        }
        ```
        **Example:**
        ```json
        {
          "status": "success",
          "totalTests": 15,
          "passedTests": 15,
          "failedTests": 0,
          "skippedTests": 0,
          "results": [
            { "suite": "auth", "testName": "should register a new user", "status": "pass", "durationMs": 120 },
            { "suite": "documents", "testName": "should create a document", "status": "pass", "durationMs": 80 }
          ],
          "timestamp": "2024-03-15T10:45:00Z",
          "durationMs": 1500
        }
        ```
*   **Status Code: `202 Accepted`** (If tests are run asynchronously and results are retrieved via another endpoint)
    *   **Body:** `{ jobId: string, status: 'accepted', message: 'Test run initiated.' }`
*   **Status Code: `400 Bad Request`**
    *   **Body:** (Standard Valtheron Error Response) - e.g., invalid `suite` or `grep` parameters.
*   **Status Code: `401 Unauthorized`**
    *   **Body:** (Standard Valtheron Error Response) - e.g., missing or invalid JWT.
*   **Status Code: `403 Forbidden`**
    *   **Body:** (Standard Valtheron Error Response) - e.g., user lacks `admin:tests:run` permission or missing/invalid MFA code.
*   **Status Code: `500 Internal Server Error`**
    *   **Body:** (Standard Valtheron Error Response) - e.g., test runner failed to start, or unexpected system error.

### Security

*   **Authentication:** Required (JWT token).
*   **Authorization:** User must have `admin:tests:run` permission.
*   **MFA Requirement:** Required (`X-MFA-Code` header with a valid OTP).
*   **Audit Logging:** Yes. All test run attempts (success, failure, initiation) are logged with user ID and test parameters.

### Middleware Applied

*   `cors`
*   `gzipCompressHeaders`
*   `rateLimitProxy` (very strict limits)
*   `authMiddleware`
*   `mfaMiddleware` (specifically for this sensitive operation)
*   `authorizeMiddleware(['admin:tests:run'])`
*   `auditMiddleware` (for logging test runs)

### Example Usage (React 19 Frontend)

```typescript jsx
import React, { useState } from 'react';

interface TestResult {
  suite: string;
  testName: string;
  status: 'pass' | 'fail' | 'skip';
  durationMs: number;
  errorMessage?: string;
}

interface RunTestsResponse {
  status: 'success' | 'failed' | 'partially-failed';
  totalTests: number;
  passedTests: number;
  failedTests: number;
  skippedTests: number;
  results: TestResult[];
  timestamp: string;
  durationMs: number;
}

const TestRunner: React.FC = () => {
  const [mfaCode, setMfaCode] = useState<string>('');
  const [suiteFilter, setSuiteFilter] = useState<string>('');
  const [grepFilter, setGrepFilter] = useState<string>('');
  const [testResults, setTestResults] = useState<RunTestsResponse | null>(null);
  const [loading, setLoading] = useState<boolean>(false);
  const [error, setError] = useState<string | null>(null);

  const handleRunTests = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError(null);
    setTestResults(null);

    const jwtToken = localStorage.getItem('jwt_token');
    if (!jwtToken) {
      setError('Authentication required. Please log in.');
      setLoading(false);
      return;
    }
    if (!mfaCode) {
      setError('MFA code is required to run tests.');
      setLoading(false);
      return;
    }

    try {
      const queryParams = new URLSearchParams();
      if (suiteFilter) queryParams.append('suite', suiteFilter);
      if (grepFilter) queryParams.append('grep', grepFilter);

      const response = await fetch(`/api/run-tests?${queryParams.toString()}`, {
        method: 'POST', // Or GET if idempotent, but triggering implies state change (test execution)
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${jwtToken}`,
          'X-MFA-Code': mfaCode,
        },
        body: JSON.stringify({}), // Empty body for now, can add options like 'dryRun'
      });

      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(errorData.message || `HTTP error! status: ${response.status}`);
      }

      const data: RunTestsResponse = await response.json();
      setTestResults(data);
      setMfaCode(''); // Clear MFA code after successful use
    } catch (err) {
      setError((err as Error).message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div>
      <h2>Integration Test Runner</h2>
      <form onSubmit={handleRunTests}>
        <div>
          <label>MFA Code (Required):</label>
          <input
            type="text"
            value={mfaCode}
            onChange={(e) => setMfaCode(e.target.value)}
            maxLength={6}
            required
            disabled={loading}
          />
        </div>
        <div>
          <label>Test Suite Filter (Optional):</label>
          <input
            type="text"
            value={suiteFilter}
            onChange={(e) => setSuiteFilter(e.target.value)}
            placeholder="e.g., auth, documents"
            disabled={loading}
          />
        </div>
        <div>
          <label>Test Name Grep (Optional):</label>
          <input
            type="text"
            value={grepFilter}
            onChange={(e) => setGrepFilter(e.target.value)}
            placeholder="e.g., user registration"
            disabled={loading}
          />
        </div>
        <button type="submit" disabled={loading || !mfaCode.trim()}>
          {loading ? 'Running Tests...' : 'Run Integration Tests'}
        </button>
      </form>

      {error && <p style={{ color: 'red' }}>Error: {error}</p>}
      {testResults && (
        <div>
          <h3>Test Run Summary ({testResults.status})</h3>
          <p>Total: {testResults.totalTests}, Passed: {testResults.passedTests}, Failed: {testResults.failedTests}</p>
          <p>Duration: {testResults.durationMs}ms</p>
          <h4>Detailed Results:</h4>
          <ul>
            {testResults.results.map((result, index) => (
              <li key={index} style={{ color: result.status === 'pass' ? 'green' : (result.status === 'fail' ? 'red' : 'orange') }}>
                [{result.suite}] {result.testName}: {result.status.toUpperCase()} ({result.durationMs}ms)
                {result.errorMessage && <div>Error: {result.errorMessage}</div>}
              </li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};

export default TestRunner;
```

### Example Implementation (Express 5.1 Backend)

```typescript
// src/routes/runTests.ts
import { Router, Request, Response, NextFunction } from 'express';
import { query, validationResult } from 'express-validator';
import { authMiddleware } from '../middleware/authMiddleware';
import { authorizeMiddleware } from '../middleware/authorizeMiddleware';
import { mfaMiddleware } from '../middleware/mfaMiddleware';
import { auditMiddleware } from '../middleware/auditMiddleware';
import { testRunnerService } from '../services/testRunnerService'; // Service to execute Vitest

const router = Router();

// Define interfaces for query parameters and response
interface RunTestsQuery {
  suite?: string;
  grep?: string;
}

interface TestResult {
  suite: string;
  testName: string;
  status: 'pass' | 'fail' | 'skip';
  durationMs: number;
  errorMessage?: string;
}

interface RunTestsResponse {
  status: 'success' | 'failed' | 'partially-failed';
  totalTests: number;
  passedTests: number;
  failedTests: number;
  skippedTests: number;
  results: TestResult[];
  timestamp: string;
  durationMs: number;
}

const runTestsValidation = [
  query('suite').optional().isString().trim().notEmpty().withMessage('Suite filter cannot be empty.'),
  query('grep').optional().isString().trim().notEmpty().withMessage('Grep filter cannot be empty.'),
];

router.post(
  '/run-tests',
  authMiddleware,
  mfaMiddleware(true), // MFA is mandatory for this critical operation
  authorizeMiddleware(['admin:tests:run']),
  runTestsValidation,
  auditMiddleware('INTEGRATION_TEST_RUN_INITIATED'),
  async (req: Request<{}, {}, {}, RunTestsQuery>, res: Response, next: NextFunction) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({
        statusCode: 400,
        message: 'Validation failed',
        error: errors.array(),
        timestamp: new Date().toISOString(),
      });
    }

    const { suite, grep } = req.query;
    const userId = req.user?.id;

    try {
      console.log(`User ${userId} initiating test run with suite: ${suite || 'all'}, grep: ${grep || 'none'}`);

      // In a production environment, this might trigger a separate process or a job queue.
      // For simplicity, this example assumes a direct call to a service that wraps Vitest.
      const results: RunTestsResponse = await testRunnerService.runTests({ suite, grep, initiatorUserId: userId });

      // The auditMiddleware has already logged the initiation.
      // We might log the full results or just the summary as part of a follow-up audit entry if needed.

      res.status(200).json(results);
    } catch (error) {
      console.error(`Error running tests for user ${userId}:`, error);
      next(error); // Pass to global error handler
    }
  }
);

export default router;
```

---

## 4. Key Best Practices

### API Design Best Practices

*   **RESTfulness:** Adhere to REST principles (resources, HTTP methods, statelessness) where appropriate.
*   **Clear Naming:** Use intuitive and consistent endpoint paths and parameter names (e.g., `/api/documents`, `/api/users/{id}`).
*   **Versioning:** Implement API versioning (e.g., `/v1/resource`) to allow for backward-incompatible changes without breaking existing clients. The `api_ver: 5.1` in the original JSON suggests a versioning strategy.
*   **Consistent Error Handling:** Provide predictable and informative error responses (status codes, error messages, unique error codes) across all endpoints.
*   **Pagination & Filtering:** For collection endpoints (like `/api/logs`), always provide pagination and filtering capabilities to manage data efficiently.
*   **Asynchronous Operations:** For long-running tasks, use an asynchronous pattern (e.g., return `202 Accepted` with a `jobId`) and provide a separate endpoint for status checks or webhooks for notifications.

### Documentation Best Practices

*   **Single Source of Truth:** Ensure this Markdown file is the primary and most up-to-date source of API truth.
*   **Completeness:** Document all aspects: method, path, description, request, response (success and error), security, and middleware.
*   **Clarity & Conciseness:** Use clear, unambiguous language. Avoid jargon where plain English suffices.
*   **Practical Examples:** Include realistic request and response examples in JSON format.
*   **TypeScript for Contracts:** Leverage TypeScript interfaces to define data structures, ensuring strong type contracts between frontend and backend.
*   **Markdown Structure:** Use headings, lists, and code blocks effectively for readability and navigation.
*   **Version Control:** Keep documentation alongside code in Git, allowing for versioning and review with code changes.

### Security Best Practices (Valtheron Specific)

*   **Authentication (JWT):** All protected endpoints must enforce JWT token validation via `authMiddleware`.
*   **Authorization (RBAC):** Implement Role-Based Access Control (RBAC) using `authorizeMiddleware` to ensure users only access resources and perform actions they are permitted to. This should be granular, potentially down to resource-level permissions.
*   **Multi-Factor Authentication (MFA):** For highly sensitive operations (e.g., `/api/logs`, `/api/run-tests`), explicitly require MFA using `mfaMiddleware`.
*   **Input Validation:** Strictly validate all incoming request data (body, query, params) on the backend using libraries like `express-validator` to prevent injection attacks and ensure data integrity.
*   **Audit Logging:** Crucial for Valtheron. Utilize `auditMiddleware` to log all significant user actions, system events, and security-relevant attempts (successes and failures) to the encrypted SQLite audit trail.
*   **Data Encryption:** Ensure sensitive data at rest (e.g., in SQLite) is encrypted (e.g., AES-256-GCM) and handled securely in transit (HTTPS).
*   **Rate Limiting:** Apply `rateLimitProxy` to prevent abuse, brute-force attacks, and denial-of-service attempts. Adjust limits based on endpoint sensitivity.
*   **Secure Headers:** Use `helmet` or similar middleware to set various HTTP headers that enhance security (e.g., XSS protection, content security policy).
*   **Least Privilege:** Design permissions such that users and services only have the minimum necessary access to perform their functions.

### TypeScript Best Practices

*   **Explicit Type Definitions:** Define clear interfaces for all request bodies, query parameters, and response payloads.
*   **Type Safety in Route Handlers:** Use type assertions (e.g., `req.body as MyRequestType`) carefully, ensuring validation occurs *before* assertion.
*   **Shared Types:** Centralize common interfaces and types (e.g., `ValtheronErrorResponse`) in a shared `types` directory to promote consistency across frontend and backend.
*   **Discriminated Unions:** Use discriminated unions for complex types with varying structures based on a specific field (e.g., different `strategyConfig` based on `refactorStrategy`).
*   **Readonly Properties:** Use `readonly` for properties that should not be modified after initialization.

### Express 5.1 Best Practices

*   **Modular Routers:** Organize routes into separate files using `express.Router()` for better maintainability and separation of concerns.
*   **Asynchronous Handlers:** Always use `async/await` for database operations, API calls, and other asynchronous tasks within route handlers. Wrap them in `try/catch` or use an `express-async-errors` package to ensure errors are passed to the `next` middleware.
*   **Centralized Error Handling:** Implement a global error handling middleware (`errorHandlerMiddleware`) to catch all unhandled errors and send consistent, secure error responses.
*   **Middleware Chaining:** Leverage the power of middleware to separate concerns like authentication, authorization, validation, logging, and data processing.
*   **Configuration Management:** Avoid hardcoding sensitive information. Use environment variables (e.g., `dotenv`) and a configuration service.
*   **Logger Integration:** Integrate a robust logging library (e.g., Winston, Pino) for structured logging, separate from audit logs.

### React 19 Best Practices

*   **Type Safety for API Calls:** Always use TypeScript interfaces to define the expected request and response structures for `fetch` or `axios` calls.
*   **Error Boundaries:** Implement React Error Boundaries to gracefully handle rendering errors within components without crashing the entire application.
*   **State Management:** Use consistent and appropriate state management solutions (e.g., `useState`, `useReducer`, React Context, or a library like Zustand/Jotai) for managing API data and loading states.
*   **Data Fetching Hooks:** Encapsulate data fetching logic within custom hooks (e.g., `useQuery` from React Query/TanStack Query) for reusability, caching, and automatic re-fetching.
*   **Loading & Error States:** Clearly communicate loading states and display user-friendly error messages during API interactions.
*   **Secure Token Handling:** Store JWT tokens securely (e.g., HttpOnly cookies for session tokens, or in-memory for short-lived access tokens with refresh tokens). Avoid `localStorage` for sensitive tokens in production.
*   **Optimistic UI Updates:** For operations that are likely to succeed, consider optimistic UI updates to improve perceived performance, with rollback mechanisms for failures.

By adhering to these best practices, Valtheron contributors will produce code that is not only functional but also secure, maintainable, scalable, and a joy to work with.