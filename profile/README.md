# **MPC-UI Embedding Protocol Specification (Draft 7)**

**Version:** 1.0.0
**Date:** 5/20/2025

## **1. Introduction**

This specification describes a protocol for embedding interactive UI components into AI chat interfaces, specifically when the AI agent is orchestrated via a Multi-Party Computation (MPC) server or similar system that registers and manages available tools. The protocol enables service providers to register embeddable UI endpoints and defines a standardized message-passing interface between the chat host and embedded UI components.

**Goal:** To foster a rich, interactive, and secure ecosystem where AI agents can intelligently select and seamlessly leverage specialized UIs, enhancing user experience, task completion, and overall agent capability. This document aims to provide a clear, robust, and adoptable standard.

## **2. Relationship to MPC and Tooling Specifications**

This MPC-UI Embedding Protocol Specification is designed to be **distinct from but compatible with** broader specifications that define core Multi-Party Computation (MPC) mechanisms, tool registration, discovery, and invocation.

-   **Separate Concerns:**
    -   Core MPC/Tooling specifications typically focus on the backend aspects of how tools are defined (inputs, outputs, execution logic), secured, discovered, and orchestrated by an AI agent.
    -   This UI Embedding Specification focuses on the *presentation layer* – how a visual component associated with a tool or data can be embedded, interact with the chat host, handle user consent for UI-specific actions, and manage its lifecycle securely.
-   **Point of Integration:**
    -   This specification defines a "UI Registration Payload" (see Section 5.1). This payload is designed to be included *within* a tool's metadata as defined by a broader MPC or tooling specification.
    -   An MPC tool definition would reference this UI spec for the structure and interpretation of its UI-related metadata. For example, an MPC tool might have an `embeddable_ui_definition` field, the content of which must conform to Section 5.1 of this document.
-   **Modularity:** This separation allows for independent evolution of the UI embedding capabilities and the core tool orchestration mechanisms, promoting flexibility and reducing complexity in each respective domain.

## **3. Background and Motivation**

AI chat interfaces are increasingly agentic, leveraging tool-use via MPC servers to perform complex tasks. However, the user experience is often limited by the chat interface’s ability to present rich, interactive content or contextually appropriate visual aids. By allowing service providers to register embeddable UIs with descriptive metadata, the AI agent can make informed decisions about when and how to present contextually relevant, interactive experiences (e.g., product galleries, data visualizations, checkout forms) directly within the chat, improving usability, understanding, and conversion.

## **4. Protocol Overview**

-   **MPC Server / Tool Registry**: Registers tools and their associated embeddable UI definitions (as per Section 5.1).
-   **AI Agent**: The core intelligence that processes user requests, invokes tools via the MPC server, analyzes results, and uses UI metadata to select and parameterize appropriate UIs for presentation.
-   **Chat Host**: The AI chat interface that embeds UI components (typically as iframes) based on the agent's decisions.
-   **Embedded UI**: A web application (served at a registered URL) that communicates with the host via `postMessage`, adhering to the defined security and communication protocols.

## **5. Registration of Embeddable UIs**

Service providers register embeddable UI definitions with the MPC server or tool registry. This registration includes metadata crucial for the AI agent to discover, understand, and appropriately utilize the UI.

### **5.1. UI Registration Fields**

The following fields constitute the UI Registration Payload, intended to be part of a tool's broader definition within an MPC system:

-   **`ui_name`** (string, required): A human-friendly name for the UI component (e.g., "Detailed Product Viewer," "Flight Comparison Chart"). Used for logging and potentially for attribution in user-facing prompts.
-   **`ui_url_template`** (string, required): The URL template for embedding the UI. May contain placeholders (e.g., `{product_id}`, `{document_id}`) to be filled by the agent. **Placeholders in this template are intended for essential data parameters required for the UI's initial server-side rendering (SSR). The agent is responsible for populating these placeholders before the host embeds the URL.**
-   **`description`** (string, required): A natural language description of the UI's purpose, ideal use cases, the type of information it best presents, and key interactions it supports. This is a primary input for the AI agent's UI selection logic.
-   **`capabilities`** (array of strings, required): An array indicating key functional aspects, content types, or interaction patterns the UI is optimized for. Examples: `image_gallery`, `video_player`, `interactive_map`, `data_chart`, `form_input`, `comparison_view`, `detailed_specification_view`, `booking_form`.
-   **`tool_association`** (string, optional): The name or ID of a primary tool this UI is designed to complement. This provides a direct hint for UI selection post-tool execution.
-   **`data_type_handled`** (string, optional): A semantic descriptor for the primary data type or object structure the UI expects (e.g., `ecommerce_product_v1`, `flight_itinerary_data_v2`, `user_profile_data`). This allows for more generic UI usage across different tools returning similar data.
-   **`permissions`** (object, required): Defines the scopes the UI might request.
    -   `required_scopes` (array of strings): Permissions essential for the UI's core functionality.
    -   `optional_scopes` (array of strings): Permissions for enhanced features.
-   **`protocol_support`** (object, required): Specifies the protocol versions supported by the UI.
    -   `min_version` (string): Minimum protocol version the UI can work with (e.g., "1.0.0").
    -   `target_version` (string): Protocol version the UI was primarily designed and tested against (e.g., "1.0.0").

### **5.1.1. Example Registration Payload**
```json
{
  "ui_name": "Interactive Product Explorer",
  "ui_url_template": "https://shop.example.com/embed/product-explorer?pid={product_id}&theme={theme_base}",
  "description": "Provides an interactive exploration of a product, featuring a zoomable image gallery, 3D model viewer if available, variant selection, and real-time stock availability. Best used when a user shows strong interest in a specific product's visual details and options.",
  "capabilities": [
    "image_gallery_zoomable",
    "3d_model_viewer",
    "variant_selection_ui",
    "real_time_availability"
  ],
  "tool_association": "getProductDetailsTool",
  "data_type_handled": "detailed_product_feed_v2.1",
  "permissions": {
    "required_scopes": ["read:product_info_basic"],
    "optional_scopes": ["read:user_reviews", "read:user_location_for_shipping"]
  },
  "protocol_support": {
    "min_version": "1.0.0",
    "target_version": "1.0.0"
  }
}
```

### **5.2. Discovery by AI Agent**
The AI agent queries the MPC server/tool registry for available tools and their associated UI definitions based on its current task or the user's query.

## **6. Agent Logic for UI Selection and Presentation**

A key aspect of this protocol is empowering the AI agent to make intelligent decisions about when and which UI to present. The agent should use a combination of factors:

### **6.1. Triggering Conditions**
The agent may decide to embed a UI:
-   Post-tool execution: After a tool successfully returns data suitable for visual presentation.
-   User intent: When the user explicitly asks for a visual representation or to perform a task best suited for a UI.
-   Contextual relevance: When the conversation implies a UI would significantly aid understanding or task completion.

### **6.2. Selection Criteria**
When multiple UIs could potentially be used, or when selecting a UI for a given data set, the agent should consider:
1.  Direct `tool_association`.
2.  `data_type_handled` Match.
3.  `description` Analysis (leveraging LLM capabilities).
4.  `capabilities` Matching.
5.  User Preferences (Future).
6.  Conversation History.

### **6.3. No-UI Decision**
The agent must also retain the option *not* to embed a UI if a textual or spoken response is more appropriate.

### **6.4. Parameterization and Data Passing**
Once a UI is selected, the agent is responsible for constructing the full UI URL by populating any placeholders in its registered `ui_url_template` with the necessary parameters (e.g., resource identifiers like `product_id`, language preferences) derived from tool outputs or the conversational context. This fully-formed URL is then provided to the chat host for iframe embedding, enabling Server-Side Rendering (SSR) of the initial UI state.

Supplementary or more complex contextual data not suitable for URL parameters, or data intended for client-side enhancement after initial render, can be passed via the `context` object in the `init` message (see Section 10.2).

### **6.5. Updating UI with New Core Data**
When the core identifying data for an embedded UI changes (e.g., viewing a different product), the recommended approach for UIs supporting SSR is for the agent to instruct the host to re-embed the UI component by providing a new, fully parameterized URL. This typically involves replacing the `src` of the existing iframe or creating a new iframe instance. The `update_context` message (Section 10.2) is more suitable for changes to supplementary data or state within an already loaded UI instance.

## **7. Embedding UI in the Chat Host**

### **7.1. Embedding Mechanism**
-   The chat host embeds the UI using an `<iframe>` element.
-   The `src` attribute is the fully parameterized URL constructed by the agent.
-   The iframe **MUST** be sandboxed with appropriate permissions (see Section 9.1).

### **7.2. UI Lifecycle**
-   The host is responsible for mounting, updating (e.g., `src` changes for new core data), and unmounting the iframe.
-   The host should provide loading states and handle errors (e.g., failed UI load).

### **7.3. Performance Considerations for Hosts**
-   Implement lazy loading for iframes.
-   Consider strategies for monitoring/limiting UI resources.
-   Cache non-sensitive, static aspects like JWKS responses (respecting cache-control).

### **7.4. UI/UX Cohesion and Host Control**
-   The `theme` message (or `theme_settings` in `init`) allows hosts to communicate styling preferences.
-   Hosts dictate iframe dimensions; UIs must be responsive.

## **8. Authentication Protocol (Host as IDP with JWK)**

### **8.1. Token Issuance and Delivery**
-   The host acts as an IDP, issuing short-lived, signed JWTs.
-   Signatures are validated using the host's public key published in JWK format at a well-known URL (e.g., `https://host.example.com/.well-known/jwks.json`).
-   The token (containing user ID, session ID, expiry, scopes, nonce, issuer, audience) and `jwks_url` are delivered in the `init` message.

### **8.2. Token Validation by UI**
-   The UI fetches the JWK set from `jwks_url` and verifies the JWT's signature and claims.
-   Tokens should include a `kid` (key ID) header for key selection.

### **8.3. Key Rotation**
-   The host may rotate keys by updating the JWK set.

### **8.4. Token Refresh and Revocation**
-   The host can send new tokens (`auth_update`) or revoke tokens (`auth_revoke`).

## **9. Security Considerations**

### **9.1. Iframe Sandboxing**
The host **MUST** embed UIs using a restrictive `sandbox` attribute:
`sandbox="allow-scripts allow-forms allow-same-origin"`
-   `allow-same-origin` is only if the UI needs access to cookies/localStorage for its *own* domain. Omit if not needed.
-   **Do NOT** use `allow-top-navigation` or `allow-popups` unless absolutely necessary and understood.

### **9.2. Data Minimization**
Only send minimum necessary user/context data. Sensitive data (payment info, PII) must not be sent via `postMessage` or exposed to the iframe unless strictly required, authorized, and secured.

### **9.3. Origin Validation**
Both host and UI **MUST** validate the `origin` of all incoming `postMessage` events.

### **9.4. Token Security**
-   Tokens: short-lived, scoped to minimum permissions, signed.
-   UI: store tokens in memory only (not localStorage/sessionStorage).

### **9.5. Clickjacking and UI Redress**
Hosts should ensure embedded UIs cannot overlay critical chat elements. Consider CSP.

### **9.6. Logging and Auditing**
Authentication events should be logged by the host.

### **9.7. Clear Responsibility Demarcation**
-   **Host:** Secure embedding, iframe lifecycle, token generation/delivery, `postMessage` origin validation.
-   **UI:** Adhering to sandbox, `postMessage` origin validation, secure token handling/validation.

## **10. Host-UI Communication Protocol**

### **10.1. Transport**
Communication uses `window.postMessage`. Messages are JSON objects with a `type` field.

### **10.2. Standard Message Types**
(See Appendix 17 for full table and illustrative payloads)

#### **From Host to UI**
-   `init`: Initialize UI. Includes `protocol_version`, `user`, `auth`, `context`, `theme_settings`.
-   `update_context`: Push supplementary context updates.
-   `theme`: Inform UI of dynamic theme changes.
-   `auth_update`: New authentication token issued.
-   `auth_revoke`: Current authentication token is revoked.
-   `permission_granted`: Inform UI of permission grant status.
-   `permission_revoked`: Inform UI that a permission has been revoked.

#### **From UI to Host**
-   `ready`: UI loaded and ready.
-   `action`: User performed a significant action.
-   `error`: UI encountered an unrecoverable error.
-   `resize`: UI requests iframe resize.
-   `request_permission`: UI requests an optional scope.

## **11. User Consent and Permission Management**

### **11.1. Scope Definition and Declaration**
-   Scopes: clearly named (e.g., `read:user_profile`, `write:cart`).
-   Registration: UI providers MUST declare `required_scopes` and `optional_scopes` (Section 5.1).

### **11.2. Host-Mediated Consent Flow**
-   **Attribution:** Prompts MUST identify the UI (name and origin).
-   **Required Scopes:** Prompt on first load or if new required scopes are added. Denial may prevent UI load or lead to degraded state.
-   **Optional Scopes (Just-In-Time):** Requested by UI via `request_permission` in response to user interaction. Host presents prompt.
-   **Token Scoping:** JWT `scope` claim reflects only currently granted permissions.

### **11.3. Permission Revocation**
-   Host MUST provide UI for users to review/revoke permissions (per-origin, per-scope).
-   Revocation effects: Host sends `permission_revoked` or `auth_revoke`; ensures UI can no longer exercise permission; updates token for future.

## **12. Protocol Versioning**

### **12.1. Versioning Scheme**
Adheres to Semantic Versioning 2.0.0 (SemVer: `MAJOR.MINOR.PATCH`).

### **12.2. Version Declaration and Discovery**
-   **Specification Document:** Clearly states its version.
-   **UI Registration (`protocol_support` field):** UI declares `min_version` and `target_version` it supports (Section 5.1).
-   **Host `init` Message:** Host includes its `protocol_version`.

### **12.3. Handling Version Compatibility**
-   **Backward Compatibility (Host newer):** Host `1.1.0` should support UI `1.0.0`.
-   **Forward Compatibility (UI newer):** UI `1.1.0` should gracefully degrade on Host `1.0.0`. UI checks `protocol_version` from `init`. Hosts should ignore unknown message types/fields from newer UIs if core structure is intact.
-   **Major Version Mismatches:** If `MAJOR` versions don't align, interaction should not proceed or result in a clear error.

### **12.4. Deprecation Policy**
Features/versions marked deprecated with ample notice and migration paths.

### **12.5. Extensibility**
Implementers should tolerate unexpected new fields in messages.

## **13. Example Flow: Ecommerce Product Discovery**
(Illustrative - simplified from Draft 6 for brevity here, but assumes the detailed logic)

1.  **User**: "Show details for product ID XYZ789."
2.  **AI Agent**: Selects `InteractiveProductExplorer` (template: `.../product-explorer?pid={product_id}`). Parameterizes URL: `.../product-explorer?pid=XYZ789`.
3.  **Chat Host**: Embeds iframe with this URL. Sends `init` message.
4.  **Embedded UI**: Server-renders using `pid`. Client-side loads, gets `init`, sends `ready`.
5.  User interacts (e.g., "Add to Wishlist"). UI requests `write:wishlist` if needed. Host handles consent. UI sends `action`.
6.  Agent/Host processes action.

## **14. Extensibility**
-   Additional message types can be proposed in future protocol versions.
-   Custom messages for specific integrations fall outside the standard.

## **15. Implementation and Adoption Guidance**

-   **Reference Implementations & SDKs:** Community encouraged to develop open-source tools.
-   **Phased Rollout:** Implement core features first.
-   **Security First:** Adherence to Section 9 is paramount.
-   **Performance Best Practices:** UIs should be lightweight and efficient.
-   **Open Standard & Community:** Encourage feedback and contributions.
-   **Demonstrate Value:** Highlight UX benefits for hosts and UI providers.

## **16. Open Questions / Areas for Feedback**

-   Should `permissions` in UI registration also include human-readable `reasoning` for each scope to aid host consent prompts?
-   How should hosts handle `MAJOR` version mismatches discovered post-load?
-   Reasonable timeframe for deprecation support?
-   For `data_type_handled`, recommend specific ontology (e.g., schema.org)?
-   Should `init` include a list of specific *optional features/messages* host supports beyond global version?
-   Common URL parameters (e.g., `locale`, `theme_base`) to recommend as standard placeholders in `ui_url_template`?

---

## **17. Appendix: Message Type Reference (Illustrative for v1.0.0)**

| Type                 | Direction      | Description                                                                                                |
|----------------------|----------------|------------------------------------------------------------------------------------------------------------|
| `init`               | Host → UI      | Initialize UI. Payload: `{ protocol_version: string, user?: User, auth?: Auth, context?: object, theme_settings?: object }` |
| `update_context`     | Host → UI      | Push supplementary context updates to UI. Payload: `{ context: object }`                                     |
| `theme`              | Host → UI      | Inform UI of theme changes. Payload: `{ theme_settings: object }`                                          |
| `auth_update`        | Host → UI      | New authentication token issued. Payload: `{ auth: Auth }`                                                   |
| `auth_revoke`        | Host → UI      | Current authentication token is revoked.                                                                     |
| `permission_granted` | Host → UI      | Host informs UI of permission grant status. Payload: `{ scope: string, granted: boolean, auth?: Auth }`      |
| `permission_revoked` | Host → UI      | Host informs UI that a previously granted permission has been revoked. Payload: `{ scope: string }`          |
| `ready`              | UI → Host      | UI has loaded and is ready.                                                                                |
| `action`             | UI → Host      | User performed an action. Payload: `{ action_name: string, payload?: object }`                               |
| `error`              | UI → Host      | UI encountered an error. Payload: `{ code: string, message: string }`                                        |
| `resize`             | UI → Host      | UI requests resize. Payload: `{ width?: string, height?: string }`                                           |
| `request_permission` | UI → Host      | UI requests an optional scope. Payload: `{ scope: string, reasoning?: string }`                              |

*(Detailed structures for `User`, `Auth`, `ThemeSettings` objects would be defined in a full specification appendix or separate schema files.)*

## **18. Appendix: Example JWK Set, JWT Header, JWT Payload**

### **Example JWK Set**

```json
{
  "keys": [
    {
      "kty": "RSA",
      "kid": "2025-05-20-key1",
      "use": "sig",
      "alg": "RS256",
      "n": "....",
      "e": "AQAB"
    }
  ]
}
```

---

### **Example JWT Header**

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "2025-05-20-key1"
}
```

---

### **Example JWT Payload**

```json
{
  "iss": "https://host.com",
  "sub": "user-123",
  "aud": "https://store.com/embed",
  "exp": 1753058400,
  "scope": ["view_product", "checkout"],
  "nonce": "random-string"
}
```

