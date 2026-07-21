As the Lead Maintainer and Architect of the Valtheron Agentic Workspace, I'm delighted to guide you through this important refactoring. Our goal is to ensure all documentation is not only technically accurate but also highly polished, structured, and immediately useful for contributing to a production-ready system.

This guide addresses the refactoring of our "HTTP Adapter" documentation. Previously located at `docs/adapters/http.md`, it will now reside at `docs/guides/http.md`. This change reflects our commitment to a clear, modular documentation structure, where "guides" provide practical, step-by-step instructions on *how to use* core functionalities, while "adapters" would detail the *implementation specifics* of integrating with external systems.

---

# Valtheron Core HTTP Client: A Guide to Secure & Compliant HTTP Communication

## 1. Executive Summary

In the Valtheron Agentic Workspace, robust, secure, and auditable HTTP communication is paramount. This guide introduces the **Valtheron Core HTTP Client**, our standardized utility for handling all HTTP requests, both on the frontend (React 19) and the backend (Express 5.1).

This client is designed to encapsulate Valtheron's stringent requirements for enterprise compliance, security (including AES-256-GCM encryption and Multi-Factor Authentication awareness), and comprehensive audit trailing. By adhering to this guide and utilizing the provided client, contributors ensure their code seamlessly integrates with Valtheron's security and operational frameworks, promoting consistency, reducing boilerplate, and mitigating common vulnerabilities.

This document serves as a practical tutorial, demonstrating how to effectively use the Valtheron Core HTTP Client for various scenarios, from making authenticated API calls to handling sensitive data securely.

## 2. Conceptual Explanation

### What is the Valtheron Core HTTP Client?

The Valtheron Core HTTP Client is not a single "adapter" in the traditional sense, but rather a set of opinionated utilities and conventions built upon standard HTTP clients (like `fetch` or a lightweight wrapper around it). It provides a unified interface for making HTTP requests throughout the Valtheron ecosystem. Its primary responsibilities include:

1.  **Standardized Request & Response Handling:** Ensures consistent headers, error formats, and data serialization/deserialization across the platform.
2.  **Automated Security Integration:**
    *   **Authentication:** Automatically injects authentication tokens (e.g., JWTs) from the user's session or environment variables.
    *   **Authorization:** Facilitates the inclusion of necessary authorization headers.
    *   **Encryption (AES-256-GCM):** Integrates with `ValtheronCryptoService` to encrypt sensitive outgoing request payloads and decrypt incoming response payloads on the backend, ensuring data confidentiality in transit and at rest within logs.
    *   **MFA Awareness:** Supports scenarios where requests require an additional Multi-Factor Authentication (MFA) code.
3.  **Comprehensive Audit Trailing:** Automatically logs significant HTTP interactions (requests and responses, with sensitive data redacted or encrypted) to our audit service, ensuring traceability and compliance.
4.  **Robust Error Management:** Provides standardized error structures and mechanisms for graceful error handling.

### Why a Dedicated Client and Guide?

*   **Security & Compliance:** Centralizing HTTP communication allows us to enforce security policies (like encryption, token handling) and compliance requirements (like audit logging) consistently without relying on individual developer implementations.
*   **Consistency & Maintainability:** Reduces variation in how HTTP requests are made, leading to more readable and maintainable code.
*   **Developer Experience:** Abstracts away complex security and compliance concerns, allowing contributors to focus on business logic.
*   **Refactored Documentation Structure:** The move from `docs/adapters/http.md` to `docs/guides/http.md` signifies a shift. The original "adapter" documentation might have described the internal workings of an HTTP client. This "guide" focuses on *how to effectively use* the pre-built Valtheron Core HTTP Client utilities in your features, aligning with our modular documentation strategy where guides provide practical "how-to" information.

## 3. Step-by-Step Code Examples

### 3.1 Backend (Express 5.1): Secure External API Communication

On the backend, the Valtheron Core HTTP Client is crucial for securely interacting with third-party services or other internal microservices. It ensures that outgoing sensitive data is encrypted, and all interactions are properly audited.

#### Scenario: Processing Sensitive Data with an External Service

Let's imagine we need to send user-provided sensitive data to an external AI service for processing, and then potentially receive sensitive results back.

```typescript
// src/services/externalDataProcessor.ts
import { Request, Response, Router } from 'express';
import { ValtheronHttpClient, ValtheronHttpRequestConfig, ValtheronHttpError, ValtheronHttpResponse } from '../core/http/ValtheronHttpClient';
import { ValtheronCryptoService } from '../core/security/ValtheronCryptoService';
import { AuditService } from '../core/audit/AuditService'; // Assuming an AuditService for logging
import { AppError } from '../shared/errors/AppError'; // Custom application error
import { logger } from '../shared/utils/logger'; // Centralized logger

// Define interfaces for request and response payloads
interface ExternalServiceRequestData {
    encryptedPayload: string; // Sensitive data always encrypted
    correlationId: string;
}

interface ExternalServiceResponseData {
    processedResult: string; // Could be encrypted or plain depending on sensitivity
    externalTransactionId: string;
    status: 'SUCCESS' | 'FAILURE';
}

// Initialize the ValtheronHttpClient for external service communication
// This instance is configured once, typically at application startup or within a service module.
const externalHttpClient = new ValtheronHttpClient({
    baseURL: process.env.EXTERNAL_AI_SERVICE_URL || 'https://api.external-ai.com',
    headers: {
        'X-API-Key': process.env.EXTERNAL_AI_SERVICE_API_KEY!, // API key for authentication
        'Content-Type': 'application/json',
        'Accept': 'application/json'
    },
    timeout: 10000 // 10-second timeout
});

const router = Router();

router.post('/process-data', async (req: Request, res: Response) => {
    // Assuming 'req.user' is populated by an authentication middleware
    const userId = req.user?.id;
    const { sensitiveInput, identifier } = req.body; // sensitiveInput is raw, needs encryption

    if (!sensitiveInput || !identifier) {
        throw new AppError('VALIDATION_ERROR', 'Sensitive input and identifier are required.', 400);
    }

    try {
        // --- Step 1: Encrypt sensitive outgoing data using ValtheronCryptoService ---
        logger.debug(`Encrypting sensitive input for identifier: ${identifier}`);
        const encryptedSensitiveInput = await ValtheronCryptoService.encrypt(sensitiveInput);

        const requestPayload: ExternalServiceRequestData = {
            encryptedPayload: encryptedSensitiveInput,
            correlationId: identifier
        };

        // --- Step 2: Make the HTTP request using ValtheronHttpClient ---
        // The client automatically handles connection pooling, retries (if configured),
        // and initial audit logging for the request.
        logger.info(`Sending request to external AI service for identifier: ${identifier}`);
        const response: ValtheronHttpResponse<ExternalServiceResponseData> = await externalHttpClient.post(
            '/analyze',
            requestPayload,
            {
                // Optional: Provide additional audit context specific to this request
                auditContext: {
                    userId: userId,
                    action: 'EXTERNAL_AI_DATA_ANALYSIS_REQUEST',
                    // Log a hash of the payload, not the raw payload, for audit trail integrity
                    payloadHash: ValtheronCryptoService.hash(JSON.stringify(requestPayload))
                }
            } as ValtheronHttpRequestConfig // Type assertion for auditContext
        );

        // --- Step 3: Handle and process the response ---
        if (response.data.status === 'FAILURE') {
            throw new AppError(
                'EXTERNAL_SERVICE_FAILURE',
                `External AI service reported failure for transaction ${response.data.externalTransactionId}`,
                502
            );
        }

        // Assuming the processed result itself might be sensitive or needs further processing
        // If 'processedResult' is encrypted, decrypt it here. For simplicity, we assume it's not.
        const finalProcessedResult = response.data.processedResult;

        // --- Step 4: Audit the successful operation ---
        await AuditService.log({
            userId: userId,
            action: 'EXTERNAL_AI_DATA_ANALYSIS_SUCCESS',
            details: `Successfully analyzed data with external AI service. External Transaction ID: ${response.data.externalTransactionId}`,
            outcome: 'SUCCESS',
            relatedEntity: { type: 'ExternalAIResponse', id: response.data.externalTransactionId },
            metadata: { identifier }
        });

        res.status(200).json({
            message: 'Data processed by external AI service successfully.',
            result: finalProcessedResult,
            externalTransactionId: response.data.externalTransactionId
        });

    } catch (error: any) {
        // --- Step 5: Robust Error Handling and Audit Failures ---
        logger.error(`Error processing external data for identifier ${identifier}:`, error);

        // Audit the failure
        await AuditService.log({
            userId: userId,
            action: 'EXTERNAL_AI_DATA_ANALYSIS_FAILURE',
            details: `Failed to process data with external AI service. Error: ${error.message}`,
            outcome: 'FAILURE',
            error: { name: error.name, message: error.message, stack: error.stack },
            metadata: { identifier }
        });

        if (error instanceof ValtheronHttpError) {
            // Handle specific HTTP errors from the external service
            throw new AppError(
                'EXTERNAL_SERVICE_HTTP_ERROR',
                `External service responded with status ${error.statusCode}: ${error.message}`,
                error.statusCode,
                error.originalError // Preserve original error for debugging
            );
        } else if (error instanceof AppError) {
            // Re-throw our custom application errors
            throw error;
        } else {
            // Catch any other unexpected errors
            throw new AppError(
                'UNEXPECTED_EXTERNAL_SERVICE_ERROR',
                'An unexpected error occurred while communicating with the external AI service.',
                500,
                error
            );
        }
    }
});

export default router;
```

#### Key Backend Takeaways:

*   **`ValtheronHttpClient`:** Centralized client for external requests, handling base URL, default headers, and timeouts.
*   **`ValtheronCryptoService.encrypt/decrypt`:** Essential for handling sensitive data payloads with AES-256-GCM. Always encrypt sensitive data *before* sending it over the wire, even to trusted external services, and decrypt only when necessary.
*   **`AuditService.log`:** Crucial for comprehensive audit trails, capturing successes and failures with rich context. Remember to redact or hash sensitive data within audit logs.
*   **`ValtheronHttpError`:** Custom error type provided by the client for standardized error handling from HTTP responses.
*   **Robust Error Handling:** Always wrap external calls in `try-catch` blocks and use custom error types (`AppError`) for consistent error reporting.

### 3.2 Frontend (React 19): Interacting with Valtheron Backend APIs

On the frontend, the Valtheron Core HTTP Client facilitates secure and authenticated communication with the Valtheron backend API. It automatically manages authentication tokens and provides a consistent interface for data fetching and submission.

#### Scenario: Updating a User Profile with MFA Requirement

Let's demonstrate a React component that allows a user to update their profile, with an optional MFA code for sensitive changes (like email).

```typescript
// src/features/user/components/UserProfileEditor.tsx
import React, { useState, useEffect, useCallback } from 'react';
import { ValtheronFrontendHttpClient, ValtheronHttpError } from '../../../core/http/ValtheronFrontendHttpClient';
import { useAuth } from '../../../core/auth/AuthContext'; // Assuming an AuthContext for managing authentication state and tokens
import { UserProfile } from '../../../shared/types/user'; // Shared type definition for UserProfile
import { ValtheronLoadingSpinner } from '../../shared/components/ValtheronLoadingSpinner'; // Custom loading component
import { ValtheronAlert } from '../../shared/components/ValtheronAlert'; // Custom alert component

// Initialize the frontend HTTP client.
// It is typically configured once at the application's root or within a dedicated API service.
// The `tokenProvider` ensures that the latest authentication token is always included.
const valtheronApi = new ValtheronFrontendHttpClient({
    baseURL: '/api', // All frontend API calls are proxied to the backend via /api
    // The tokenProvider function is called by the client for each request
    // to dynamically get the current authentication token.
    tokenProvider: () => useAuth().getToken(), // Example of integrating with AuthContext
});

// Define interfaces for request and response payloads
interface UpdateProfileRequest {
    name: string;
    email: string;
    mfaCode?: string; // Optional, required for sensitive updates
}

const UserProfileEditor: React.FC = () => {
    const { isAuthenticated, user, getToken } = useAuth(); // Access authentication state and token provider
    const [profile, setProfile] = useState<UserProfile | null>(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState<string | null>(null);
    const [isUpdating, setIsUpdating] = useState(false);
    const [newName, setNewName] = useState('');
    const [newEmail, setNewEmail] = useState('');
    const [mfaCode, setMfaCode] = useState('');
    const [showMfaPrompt, setShowMfaPrompt] = useState(false);

    // useCallback to memoize the fetch function and prevent unnecessary re-renders
    const fetchProfile = useCallback(async () => {
        if (!isAuthenticated) {
            setError('User not authenticated. Please log in.');
            setLoading(false);
            return;
        }
        setLoading(true);
        setError(null);
        try {
            // --- Step 1: Make an authenticated GET request ---
            // The ValtheronFrontendHttpClient automatically injects the Authorization token
            // using the configured `tokenProvider`.
            const response = await valtheronApi.get<UserProfile>('/user/profile');
            setProfile(response.data);
            setNewName(response.data.name);
            setNewEmail(response.data.email);
        } catch (err) {
            // --- Step 2: Handle HTTP errors from the backend ---
            if (err instanceof ValtheronHttpError) {
                setError(`Failed to fetch profile: ${err.message} (Code: ${err.code || 'N/A'})`);
            } else {
                setError('An unexpected error occurred while fetching profile.');
            }
            console.error('Error fetching profile:', err);
        } finally {
            setLoading(false);
        }
    }, [isAuthenticated]); // Dependency on isAuthenticated

    useEffect(() => {
        fetchProfile();
    }, [fetchProfile]); // Re-run when fetchProfile changes

    const handleUpdateProfile = async (event: React.FormEvent) => {
        event.preventDefault();
        setIsUpdating(true);
        setError(null);
        setShowMfaPrompt(false);

        const requestBody: UpdateProfileRequest = {
            name: newName,
            email: newEmail,
            ...(mfaCode && { mfaCode }) // Conditionally include MFA code
        };

        try {
            // --- Step 3: Make an authenticated PUT request with potential MFA ---
            const response = await valtheronApi.put<UserProfile>('/user/profile', requestBody);
            setProfile(response.data);
            ValtheronAlert.success('Profile updated successfully!');
            setMfaCode(''); // Clear MFA code on success
            setShowMfaPrompt(false); // Hide MFA prompt
        } catch (err) {
            // --- Step 4: Handle specific backend errors, including MFA requirements ---
            if (err instanceof ValtheronHttpError) {
                if (err.statusCode === 403 && err.code === 'MFA_REQUIRED') {
                    // Backend explicitly requested MFA for this sensitive operation
                    setError('MFA code is required for this update. Please enter your code.');
                    setShowMfaPrompt(true);
                } else if (err.statusCode === 401 || err.statusCode === 403) {
                    setError(`Authentication/Authorization error: ${err.message}`);
                    // Potentially redirect to login or refresh token
                } else {
                    setError(`Failed to update profile: ${err.message} (Code: ${err.code || 'N/A'})`);
                }
            } else {
                setError('An unexpected error occurred while updating profile.');
            }
            console.error('Error updating profile:', err);
        } finally {
            setIsUpdating(false);
        }
    };

    if (loading) return <ValtheronLoadingSpinner message="Loading profile..." />;
    if (error && !showMfaPrompt) return <ValtheronAlert type="error" message={error} />;
    if (!profile) return <ValtheronAlert type="info" message="No profile data available." />;

    return (
        <div className="valtheron-card profile-editor">
            <h2>Edit User Profile</h2>
            <form onSubmit={handleUpdateProfile}>
                {error && <ValtheronAlert type="error" message={error} />}
                <div>
                    <label htmlFor="name">Name:</label>
                    <input
                        id="name"
                        type="text"
                        value={newName}
                        onChange={(e) => setNewName(e.target.value)}
                        disabled={isUpdating}
                        className="valtheron-input"
                    />
                </div>
                <div>
                    <label htmlFor="email">Email:</label>
                    <input
                        id="email"
                        type="email"
                        value={newEmail}
                        onChange={(e) => setNewEmail(e.target.value)}
                        disabled={isUpdating}
                        className="valtheron-input"
                    />
                </div>
                {(showMfaPrompt || (mfaCode && error && error.includes('MFA_REQUIRED'))) && (
                    <div>
                        <label htmlFor="mfaCode">MFA Code:</label>
                        <input
                            id="mfaCode"
                            type="text"
                            value={mfaCode}
                            onChange={(e) => setMfaCode(e.target.value)}
                            disabled={isUpdating}
                            placeholder="Enter your MFA code"
                            className="valtheron-input"
                        />
                        <p className="valtheron-text-muted">An MFA code may be required for sensitive changes.</p>
                    </div>
                )}
                <button type="submit" disabled={isUpdating} className="valtheron-button valtheron-button-primary">
                    {isUpdating ? 'Updating...' : 'Update Profile'}
                </button>
            </form>
        </div>
    );
};

export default UserProfileEditor;
```

#### Key Frontend Takeaways:

*   **`ValtheronFrontendHttpClient`:** The dedicated client for frontend-to-backend communication, automatically handling token injection via `tokenProvider`.
*   **`useAuth().getToken()`:** Integration with Valtheron's authentication context to retrieve the current user's session token.
*   **`ValtheronHttpError`:** Catch specific errors returned by the Valtheron backend, enabling tailored UI feedback (e.g., prompting for MFA).
*   **React 19 Conventions:** Uses functional components, `useState`, `useEffect`, `useCallback` for managing component state and lifecycle.
*   **User Feedback:** Provides clear loading states, error messages, and success notifications to the user.
*   **MFA Integration:** Demonstrates how to conditionally prompt for and send an MFA code when required by the backend for sensitive operations.

## 4. Key Best Practices

Adhering to these best practices when using the Valtheron Core HTTP Client ensures your contributions are secure, compliant, and maintainable:

1.  **Prioritize Security:**
    *   **Always use HTTPS:** Ensure all communication, internal and external, uses secure TLS/SSL. The Valtheron environment enforces this by default.
    *   **Encrypt Sensitive Payloads (Backend):** Use `ValtheronCryptoService.encrypt` for all sensitive data sent to external services or stored in audit logs. Decrypt only at the point of consumption.
    *   **Token Management:** Never expose authentication tokens on the frontend via URL parameters or console logs. Rely on the `ValtheronFrontendHttpClient`'s `tokenProvider` for secure injection.
    *   **Validate Certificates:** Ensure server-side HTTP clients are configured to validate SSL/TLS certificates of external services.

2.  **Robust Error Handling:**
    *   **Catch and Handle:** Always wrap HTTP calls in `try-catch` blocks.
    *   **Use `ValtheronHttpError`:** Leverage the custom error types provided by the Valtheron HTTP Clients to differentiate between network issues, client-side errors, and specific backend/external service errors.
    *   **Standardized Error Responses:** Ensure your backend API consistently returns standardized error structures that the frontend client can easily parse and display.
    *   **Meaningful User Feedback:** On the frontend, translate backend errors into user-friendly messages.

3.  **Comprehensive Auditability:**
    *   **Log All Critical Interactions:** Utilize `AuditService.log` on the backend for every significant HTTP request and response, especially those involving sensitive data or system-altering actions.
    *   **Redact/Encrypt Sensitive Data in Logs:** Never log raw sensitive data. Either encrypt it before logging or log only hashes/metadata. The Valtheron HTTP Client and Audit Service are designed to facilitate this.
    *   **Include Context:** Provide sufficient `auditContext` (e.g., `userId`, `action`, `correlationId`) to make audit trails meaningful.

4.  **Consistency and Reusability:**
    *   **Leverage Valtheron Clients:** Always use the provided `ValtheronHttpClient` (backend) and `ValtheronFrontendHttpClient` (frontend) rather than implementing raw `fetch` or Axios calls directly.
    *   **Shared Types:** Define and use shared TypeScript interfaces for request and response payloads (`src/shared/types/`). This ensures type safety across frontend and backend.
    *   **Centralized Configuration:** Configure `ValtheronHttpClient` instances centrally (e.g., in `src/core/http/`) and export them for reuse.

5.  **Data Validation:**
    *   **Input Validation:** Validate all incoming request data (e.g., `req.body` in Express) on the backend before processing or forwarding.
    *   **Output Validation:** Consider validating outgoing request data and incoming response data against expected schemas to catch unexpected formats early.

6.  **Performance Considerations:**
    *   **Avoid Redundant Requests:** Implement caching strategies where appropriate to reduce unnecessary HTTP calls.
    *   **Efficient Data Transfer:** Optimize payload sizes and consider compression for large data transfers.

7.  **Thorough Testing:**
    *   **Unit Tests:** Write unit tests for services and components that make HTTP requests, mocking the `ValtheronHttpClient` to isolate logic.
    *   **Integration Tests:** Implement integration tests to verify the end-to-end flow of HTTP communication, including authentication, encryption, and audit logging.

By following this guide and these best practices, you contribute to building a secure, reliable, and compliant Valtheron Agentic Workspace. Your adherence to these standards is deeply appreciated and critical for the platform's success.