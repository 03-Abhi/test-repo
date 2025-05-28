# GitHub Copilot Instructions for qa-components

This document provides best practices and guidelines for developing within the `qa-components` repository, with a particular focus on leveraging GitHub Copilot effectively. Our goal is to maintain a high-quality, consistent, and maintainable codebase.

## General Principles

*   **Universality:** Strive to create universal solutions. If a logic or component developed for "Mothership" (legacy products) can be applied to "Speedboat" (new products), adapt and reuse it. Aim for consistent internal methods and data flows across different product lines. This reduces redundancy and ensures a cohesive user experience.
*   **Single Source of Truth:** Critical data, mappings, or configurations that are used in multiple places (especially between frontend and backend) should be defined once. Preferably, define them in backend constants (`qa_ops/config/constants.js`) and expose them to the frontend via an API. This minimizes duplication, prevents inconsistencies, and ensures maintainability.
*   **Modularity and Reusability:** Design components (React) and modules (Node.js) to be highly reusable and have a single, well-defined responsibility. This makes the codebase easier to understand, test, and scale.
*   **Readability and Maintainability:** Write clean, well-commented, and consistently formatted code. Copilot can assist with generating JSDoc comments, explaining code blocks, and suggesting refactorings for clarity.

## Project Structure and Development Guidelines

### 1. `qa_ops` (Backend - Node.js API)

The `qa_ops` service is a Node.js application using the Express framework. It serves as the backend API, handling business logic, data processing, interactions with databases (e.g., PostgreSQL with Sequelize), and communication with other internal or external services.

**Key Directory Structure:**

```
qa_ops/
├── README.md                 # Overview of the qa_ops service, setup instructions, and key information.
├── app.js                    # Main application setup: Initializes Express, mounts middleware (e.g., body-parser, cors), error handlers, and basic security configurations.
├── branch_recreation_scheduler/ # Contains scripts and logic related to automated branch recreation processes, likely scheduled tasks.
├── config/                   # Configuration files for various aspects of the application.
│   ├── constants.js          # CRITICAL: Defines application-level constants, frequently used static data, mappings (e.g., status codes, type definitions), and non-sensitive configurations. This is the single source of truth for such data.
│   ├── database.js           # (Example, actual name might vary) Sequelize or other database connection configurations for different environments (development, staging, production).
│   └── environment.js        # (Example) Manages environment-specific variables and settings.
├── controllers/              # Request handlers (Express route handlers). Each controller typically groups related actions for a specific resource (e.g., `userController.js`, `reportController.js`). Controllers are responsible for:
│                             #   - Receiving and validating request data (often using Joi schemas from `params/`).
│                             #   - Calling appropriate helper functions or service modules for business logic.
│                             #   - Formatting and sending HTTP responses.
├── ecosystem.config.js       # PM2 process manager configuration file. Defines how the application should be run, monitored, and managed in production (e.g., clustering, environment variables, logging).
├── helpers/                  # Utility functions and helper modules that encapsulate reusable business logic, data transformations, or interactions with external services. (e.g., `dateFormatter.js`, `apiClientHelper.js`).
├── index.js                  # Entry point of the application. Typically imports `app.js` and starts the HTTP server (e.g., `app.listen()`).
├── jenkins_metrics/          # Modules and scripts specifically for collecting, processing, or exposing Jenkins-related metrics.
├── migrations/               # Database migration scripts (e.g., using Sequelize CLI). Each file defines changes to the database schema (creating tables, adding columns, etc.) and allows for version control of the database structure.
├── models/                   # Database schemas and models, typically defined using an ORM like Sequelize. Each file represents a table in the database and defines its columns, data types, relationships, and any associated instance/static methods. (e.g., `User.js`, `Report.js`).
├── package.json              # Node.js project manifest: lists dependencies, development dependencies, scripts (e.g., for starting, testing, linting), and project metadata.
├── params/                   # Contains validation schemas, often using Joi. These schemas define the expected structure and data types for incoming request parameters (body, query, path params) and are used in controllers or middleware for validation.
├── public/                   # Static assets that can be served directly by Express (e.g., images, documentation files). Less common for pure APIs.
├── routes/                   # API route definitions. Routes are typically grouped by resource or feature into separate files (e.g., `userRoutes.js`, `reportRoutes.js`). Each file defines URL patterns and maps them to controller actions.
├── routes.js                 # Main router aggregation file. Imports all individual route files from the `routes/` directory and mounts them onto the main Express application router.
├── scripts/                  # Utility or maintenance scripts that are not part of the main application flow but are useful for development, deployment, or administrative tasks (e.g., data seeding, cleanup tasks).
└── views/                    # Server-side rendered views (e.g., using EJS, Pug). Less common if `qa_ops` is a pure JSON API, but might be used for simple status pages or admin interfaces.
```

**Best Practices for `qa_ops`:**

*   **Constants (`config/constants.js`):**
    *   This is the **single source of truth**. Any data, mapping, or configuration that might be used in multiple places or is critical for consistency (e.g., product types, status enums, role definitions) MUST be defined here.
    *   If the frontend (`qa_ops_dashboard`) needs access to these constants, create specific, lean API endpoints in `qa_ops` to expose only the necessary constants. **Do NOT duplicate backend constants directly in the frontend codebase.**
    *   **Example:**
        ```javascript
        // qa_ops/config/constants.js
        const PRODUCT_LINES = Object.freeze({
          MOTHERSHIP: 'mothership',
          SPEEDBOAT: 'speedboat'
        });

        const USER_ROLES = Object.freeze({
          ADMIN: 'admin',
          USER: 'user',
          GUEST: 'guest'
        });

        module.exports = { PRODUCT_LINES, USER_ROLES };

        // qa_ops/controllers/constantsController.js
        const { PRODUCT_LINES, USER_ROLES } = require('../config/constants');

        exports.getProductLines = (req, res) => {
          res.json(PRODUCT_LINES);
        };

        exports.getUserRoles = (req, res) => {
          res.json(USER_ROLES); // Expose only what's needed
        };
        ```
*   **Controllers:** Keep controllers lean and focused on request/response handling. Delegate complex business logic to `helpers/` or dedicated service modules. Ensure proper error handling and consistent response formats.
*   **Routes:** Group routes logically in the `routes/` directory. Use descriptive naming for route files (e.g., `featureNameRoutes.js`) and handlers. Version your APIs if significant breaking changes are introduced (e.g., `/api/v1/resource`, `/api/v2/resource`).
*   **Models:** Define clear, well-documented database schemas using Sequelize. Utilize model associations (hasOne, hasMany, belongsTo, belongsToMany) to represent relationships. Add instance or static methods to models for common operations related to that data.
*   **Error Handling:** Implement robust, centralized error handling middleware. This middleware should catch errors from route handlers and services, log them appropriately, and send user-friendly error responses with correct HTTP status codes.
*   **API Design:** Adhere to RESTful principles. Use standard HTTP verbs (GET, POST, PUT, DELETE, PATCH) correctly. Design clear, consistent, and predictable URL structures.
*   **Validation:** Use Joi schemas in the `params/` directory for all incoming request data validation. Validate early in the controller or via middleware.

**Copilot Usage for `qa_ops`:**

*   "Generate an Express route for `POST /api/v1/items` that uses the `itemController.createItem` handler and includes Joi validation from `params/itemSchemas.js`."
*   "Write a Sequelize model for 'Product' with fields: `name` (string, not null), `description` (text), `productLine` (enum from `config/constants.js PRODUCT_LINES`), `isActive` (boolean, default true)."
*   "Create a Joi schema in `params/itemSchemas.js` for validating a request body to create an item. It should have `name` (string, required, min 3 chars), `quantity` (number, integer, min 1), and `productId` (UUID, required)."
*   "Refactor this controller action to use async/await and improve error handling."
*   "Write a helper function in `qa_ops/helpers/dataTransformer.js` to convert an array of database objects into a specific DTO format."
*   "Add JSDoc comments to this Sequelize model, explaining its fields and associations."

### 2. `qa_ops_dashboard` (Frontend - React)

The `qa_ops_dashboard` is a React application, likely built with Vite, providing the user interface for interacting with the `qa_ops` API and visualizing data. It enables users to manage QA operations, view reports, and monitor various metrics.

**Key Directory Structure (within `ops_dashboard/src/`):**

```
qa_ops_dashboard/ops_dashboard/
├── README.md                 # Overview of the dashboard, setup, and development guidelines.
├── index.html                # Main HTML file, entry point for the Vite/React application.
├── package.json              # Lists frontend dependencies (React, Vite, Axios, etc.), dev dependencies, and scripts (start, build, test, lint).
├── vite.config.js            # Vite build configuration: defines plugins, server options, build output, aliases, etc.
├── public/                   # Static assets (images, fonts, manifest.json) that are copied directly to the build output.
└── src/
    ├── App.jsx                   # Main application component: sets up routing, global context providers, and overall layout structure.
    ├── main.jsx                  # Entry point for the React application: renders the root `App` component into the DOM.
    ├── assets/                   # Static assets like images (png, svg), fonts, or other media files imported by components.
    ├── components/               # Reusable UI components, organized by feature or commonality.
    │   ├── common/               # General-purpose, highly reusable components used across multiple features (e.g., `Button.jsx`, `Modal.jsx`, `Spinner.jsx`, `Table.jsx`).
    │   ├── reporting/            # Components specifically for the "Reporting" feature. This structure is a good model for other features.
    │   │   ├── styles/           # CSS Modules or styled-components for `reporting` components.
    │   │   │   └── ReportView.module.css # Example: CSS Module for ReportView, ensuring scoped styles.
    │   │   ├── utils/            # Utility functions specific to the reporting feature (e.g., `reportDataFormatter.js`, `chartHelper.js`).
    │   │   │   └── reportingUtils.js
    │   │   ├── views/            # Larger, container-like components that represent a specific view or page within the reporting feature. They compose smaller components and manage feature-specific state.
    │   │   │   └── ReportView.jsx
    │   │   ├── api/              # Functions dedicated to making API calls related to the reporting feature. These functions typically use a centralized API client.
    │   │   │   └── reportingAPI.js
    │   │   └── ReportSpecificComponent.jsx # A component with a single, specific use case within the reporting feature.
    │   └── gtg/                  # Components for the GTG (Good To Go) dashboard. Follow a similar sub-structure as `reporting/` (api/, styles/, utils/, views/).
    │       ├── styles/
    │       ├── utils/
    │       ├── views/
    │       └── api/
    ├── constants/                # Frontend-specific constants.
    │   └── appConstants.js       # UI text, theme configurations (colors, fonts), animation timings, local storage keys, etc. **Avoid duplicating backend constants here; fetch them via API.**
    ├── contexts/                 # React Context API providers and consumers for managing global or shared state across different parts of the application (e.g., `AuthContext.jsx`, `ThemeContext.jsx`).
    ├── hooks/                    # Custom React hooks that encapsulate reusable stateful logic (e.g., `useForm.js`, `useFetch.js`, `useLocalStorage.js`).
    ├── layouts/                  # Layout components that define the overall structure of pages (e.g., `MainLayout.jsx` with Header, Sidebar, Footer; `AuthLayout.jsx` for login/signup pages).
    ├── pages/                    # Top-level page components that are directly mapped to routes. These components often compose layout components and feature-specific `views/` components. (e.g., `HomePage.jsx`, `ReportingPage.jsx`, `GtgDashboardPage.jsx`).
    ├── services/                 # API service layers.
    │   └── apiClient.js          # Centralized API client setup (e.g., an Axios instance) with base URL, default headers, interceptors for request/response handling (e.g., adding auth tokens, global error handling).
    ├── styles/                   # Global styles, theme definitions, CSS resets, or utility CSS classes.
    │   └── global.css
    ├── utils/                    # Global utility functions that are not specific to any single feature or component (e.g., `dateFormatter.js`, `validationUtils.js`, `localStorageHelper.js`).
    └── router/                   # React Router configuration.
        └── index.js              # Defines application routes, including protected routes, nested routes, and lazy loading of page components.
```

**Best Practices for `qa_ops_dashboard`:**

*   **Component Structure (inspired by `src/components/reporting/`):**
    *   **Feature-Based Modularity:** Group components, API calls, utils, and styles by feature (e.g., `reporting/`, `gtg/`, `userManagement/`).
    *   **`views/`**: These are smart/container components. They fetch data (often via functions from the feature's `api/` directory), manage state for that feature view, and compose smaller, dumber UI components.
    *   **`styles/`**: Use CSS Modules (`*.module.css`) for component-level styling to prevent class name collisions and ensure encapsulation. For global styles, use `src/styles/`.
    *   **`utils/`**: Place utility functions specific to a feature within its directory. Global utilities go into `src/utils/`.
    *   **`api/`**: Isolate all API call logic for a feature here. These functions should use the central `apiClient.js` from `src/services/`. This makes it easy to find and manage where and how backend calls are made.
    *   **Single Use Case Components:** Strive to create components with a single, clear responsibility. This enhances reusability, testability, and maintainability.
*   **Constants (`src/constants/appConstants.js`):**
    *   Use for UI-specific text, theme settings, etc.
    *   **Crucially:** For any data that needs to be consistent with the backend (e.g., product types, status codes, role definitions), **fetch it from `qa_ops` API endpoints.** Do not hardcode or duplicate backend constants in the frontend.
    *   **Example (consuming backend constants):**
        ```javascript
        // qa_ops_dashboard/ops_dashboard/src/components/reporting/api/reportingAPI.js
        import apiClient from '../../../services/apiClient';

        export const fetchProductLines = async () => {
          try {
            // Assuming qa_ops has an endpoint like /api/constants/product-lines
            const response = await apiClient.get('/constants/product-lines');
            return response.data; // e.g., { MOTHERSHIP: 'mothership', SPEEDBOAT: 'speedboat' }
          } catch (error) {
            console.error("Error fetching product lines:", error);
            // Handle error appropriately, maybe return a default or throw
            return {};
          }
        };

        // In a component:
        // import { fetchProductLines } from '../api/reportingAPI';
        // const [productLines, setProductLines] = useState({});
        // useEffect(() => {
        //   fetchProductLines().then(data => setProductLines(data));
        // }, []);
        ```
*   **State Management:**
    *   For simple local component state, use `useState` and `useReducer`.
    *   For cross-component state or global state, use React Context (`src/contexts/`) for moderately complex scenarios.
    *   For more complex global state management, consider libraries like Zustand or Redux Toolkit if the need arises (evaluate based on application complexity).
*   **API Calls:**
    *   Centralize all API interaction logic. Use the `apiClient.js` in `src/services/` for base configuration (Axios instance).
    *   Feature-specific API functions should reside in the feature's `api/` directory (e.g., `src/components/reporting/api/reportingAPI.js`).
    *   Implement consistent error handling for API calls, perhaps using interceptors in `apiClient.js` or a custom hook.
*   **Styling:**
    *   Prefer CSS Modules for component-scoped styles.
    *   Use global styles in `src/styles/` for base styling, resets, and theme variables.
    *   Consider a utility-first CSS framework (like Tailwind CSS) if it aligns with team preference and project needs, but ensure it's configured correctly with Vite.
*   **GTG Dashboard Extension (for Speedboat teams):**
    *   **Identify Reusability:** Before creating new GTG components for Speedboat, thoroughly examine existing Mothership GTG components in `src/components/gtg/`.
    *   **Adapt and Parameterize:** Adapt existing components to handle data from both Mothership and Speedboat. This might involve:
        *   Passing a `productLine` prop to components.
        *   Making API calls in `src/components/gtg/api/` conditional or parameterized based on `productLine`.
        *   Using conditional rendering within components to display Speedboat-specific information or UI elements.
    *   **New Components:** Only create new components if Speedboat requirements are fundamentally different and cannot be met by adapting existing ones. Place them within the `src/components/gtg/` structure.
    *   **Universal Data Models:** Ensure the data structures returned by the backend for GTG (for both Mothership and Speedboat) are as consistent as possible to simplify frontend logic.

**Copilot Usage for `qa_ops_dashboard`:**

*   "Create a React functional component `ProductSelectorDropdown` in `src/components/common/` that takes an array of `products` (each with `id` and `name`) and an `onSelectProduct` callback prop. It should render a dropdown/select element. Use CSS Modules for styling."
*   "Write a function for `src/components/settings/api/settingsAPI.js` named `updateUserSettings` that makes a PUT request to `/api/v1/users/settings` with the provided settings object, using the global `apiClient`."
*   "Generate CSS Module styles in `src/components/common/Button.module.css` for a button component with primary, secondary, and disabled states."
*   "How do I implement a custom React hook `useAppContext()` that provides easy access to multiple contexts (e.g., AuthContext, ThemeContext)?"
*   "Show me how to adapt the `DashboardCard.jsx` component to display a 'New' badge if `props.isNewFeature` is true and `props.productLine` is 'speedboat'."
*   "Generate a basic React Router setup in `src/router/index.js` with a public route for `/login` and a private route for `/dashboard` that requires authentication (assume an `AuthContext` provides `isAuthenticated`). Lazy load the page components."

## Mothership vs. Speedboat

*   **Mothership:** Refers to our legacy product lines. The codebase for these is generally more mature and established.
*   **Speedboat:** Refers to newly added product lines. Development for Speedboat should prioritize leveraging and extending the existing `qa-components` infrastructure rather than building from scratch.

**Key Strategy:**

*   **Maximize Code Reuse:** Before writing any new code for a Speedboat feature, **always** investigate if similar functionality exists for Mothership (in both `qa_ops` and `qa_ops_dashboard`). Adapt, extend, and reuse existing code, components, API endpoints, and backend logic whenever feasible. This is crucial for consistency, reducing development effort, and minimizing maintenance overhead.
*   **Universal Flow & Logic:** Strive to maintain the same or very similar internal methods, data structures, and user flows for both Mothership and Speedboat features. If a process for Mothership involves steps A, B, and C, the equivalent Speedboat process should ideally follow the same conceptual steps, even if the underlying data sources or specific implementation details differ slightly.
    *   **Example:** If a reporting workflow for Mothership involves fetching data, transforming it, and then displaying it in a chart, the Speedboat reporting workflow should follow this same pattern. The specific API endpoints or transformation rules might vary, but the overall flow should be consistent.
*   **Configuration-Driven Design:** Where differences between Mothership and Speedboat are unavoidable, try to make components and backend logic configuration-driven rather than creating entirely separate code paths.
    *   Pass parameters (e.g., `productLine`) to functions or components.
    *   Use conditional logic based on these parameters.
    *   Fetch configuration from the backend where appropriate.

**Example Scenario (Revisited):**

If `qa_ops_dashboard` has a reporting feature for Mothership products displaying metrics X, Y, and Z, and you need to add reporting for Speedboat products displaying metrics X, Y, and W:

1.  **Backend (`qa_ops`):**
    *   **Modify Existing Endpoints:** Adapt existing API endpoints (e.g., `/api/reports/summary`) to accept a `productLine` query parameter (e.g., `?productLine=mothership` or `?productLine=speedboat`).
    *   **Adapt Logic:** The controller and helper/service logic should use this `productLine` parameter to fetch and aggregate data accordingly. The core data aggregation logic might be largely reusable.
    *   **Consistent Response Structure:** Aim for a consistent response structure from the API, even if some fields are specific to a product line (they can be null or omitted for the other).
2.  **Frontend (`qa_ops_dashboard`):**
    *   **Reuse Components:** Reuse the existing reporting components from `src/components/reporting/`.
    *   **Parameterized API Calls:** Modify API calls in `src/components/reporting/api/reportingAPI.js` to include the `productLine` parameter when fetching data.
    *   **Conditional Rendering:** In the `views/` components (e.g., `ReportView.jsx`), use the fetched data and the `productLine` prop to conditionally render metric Z (for Mothership) or W (for Speedboat). The components displaying X and Y would remain largely unchanged.
    *   **Flexible Components:** Design components to be flexible. For instance, a chart component could accept an array of metric configurations to display, rather than hardcoding specific metrics.

## Copilot Prompts - Examples

**General:**

*   "Explain this block of [language] code: [paste code]"
*   "Refactor this function to be more readable and efficient."
*   "Add JSDoc/TSDoc comments to this function/class/module."
*   "What are the potential edge cases or error conditions for this logic?"
*   "Suggest a more descriptive name for this variable/function."
*   "Convert this Promise-based code to use async/await."

**`qa_ops` (Backend - Node.js/Express):**

*   "Generate an Express GET route for `/api/v1/users/:userId/profile` that uses the `userController.getUserProfile` handler. Ensure it includes a Joi validation middleware for the `userId` path parameter (must be a UUID)."
*   "Write a Sequelize model named 'FeatureToggle' with fields: `name` (string, unique, not null), `description` (text), `isEnabled` (boolean, default false), `productLine` (enum using `PRODUCT_LINES` from `config/constants.js`). Include timestamps."
*   "Create a Joi schema in `qa_ops/params/authSchemas.js` for a login request body with `email` (required, valid email format) and `password` (required, string, min 8 characters, max 128 characters)."
*   "How can I write a helper function in `qa_ops/helpers/authHelper.js` to generate a JWT token with a user ID and role?"
*   "Write a test case using Jest and Supertest for the `POST /api/v1/items` endpoint, mocking the database interaction."

**`qa_ops_dashboard` (Frontend - React/Vite):**

*   "Create a React functional component `ProductSelectorDropdown` in `src/components/common/` that takes an array of `products` (each with `id` and `name`) and an `onSelectProduct` callback prop. It should render a dropdown/select element. Use CSS Modules for styling."
*   "Write a function for `src/components/settings/api/settingsAPI.js` named `updateUserSettings` that makes a PUT request to `/api/v1/users/settings` with the provided settings object, using the global `apiClient`."
*   "Generate CSS Module styles in `src/components/common/Button.module.css` for a button component with primary, secondary, and disabled states."
*   "How do I implement a custom React hook `useAppContext()` that provides easy access to multiple contexts (e.g., AuthContext, ThemeContext)?"
*   "Show me how to adapt the `DashboardCard.jsx` component to display a 'New' badge if `props.isNewFeature` is true and `props.productLine` is 'speedboat'."
*   "Generate a basic React Router setup in `src/router/index.js` with a public route for `/login` and a private route for `/dashboard` that requires authentication (assume an `AuthContext` provides `isAuthenticated`). Lazy load the page components."

By following these detailed guidelines and leveraging GitHub Copilot effectively with targeted prompts, we can build robust, scalable, and maintainable solutions within the `qa-components` ecosystem. Always critically review and test Copilot's suggestions before committing them.
