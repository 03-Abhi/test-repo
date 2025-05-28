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
│   ├── database.js           # (Example, actual name might vary) Sequelize or other database connection configurations (connection strings, pooling options) for different environments (development, staging, production). Often uses environment variables for credentials.
│   └── environment.js        # (Example) Manages environment-specific variables and settings, often using a library like `dotenv` to load `.env` files.
├── controllers/              # Request handlers (Express route handlers). Each controller typically groups related actions for a specific resource (e.g., `userController.js`, `reportController.js`). Controllers are responsible for:
│                             #   - Receiving and validating request data (often using Joi schemas from `params/`).
│                             #   - Calling appropriate helper functions or service modules from `helpers/` for business logic.
│                             #   - Formatting and sending HTTP responses.
├── ecosystem.config.js       # PM2 process manager configuration file. Defines how the application should be run, monitored, and managed in production (e.g., clustering, environment variables, logging).
├── helpers/                  # Utility functions and helper modules that encapsulate reusable business logic, data transformations, or interactions with external services. (e.g., `dateFormatter.js`, `authHelper.js`, `dbQueryHelper.js`). These are crucial for keeping controllers lean.
├── index.js                  # Entry point of the application. Typically imports `app.js` and starts the HTTP server (e.g., `app.listen()`).
├── jenkins_metrics/          # Modules and scripts specifically for collecting, processing, or exposing Jenkins-related metrics.
├── migrations/               # Database migration scripts (e.g., using Sequelize CLI). Each file defines changes to the database schema (creating tables, adding columns, etc.) and allows for version control of the database structure.
├── models/                   # Database schemas and models, typically defined using an ORM like Sequelize. Each file (e.g., `User.js`, `Report.js`) represents a table and defines its columns, data types, associations (hasOne, hasMany, etc.), and any associated instance/static methods for custom queries or data manipulation.
├── package.json              # Node.js project manifest: lists dependencies, development dependencies, scripts (e.g., for starting, testing, linting), and project metadata.
├── params/                   # Contains validation schemas, often using Joi. These schemas (e.g., `userValidationSchemas.js`) define the expected structure, data types, and constraints for incoming request parameters (body, query, path params) and are used in controllers or as middleware for validation.
├── public/                   # Static assets that can be served directly by Express (e.g., images, documentation files). Less common for pure APIs.
├── routes/                   # API route definitions. Routes are typically grouped by resource or feature into separate files (e.g., `userRoutes.js`, `reportRoutes.js`). Each file defines URL patterns and maps them to controller actions.
├── routes.js                 # Main router aggregation file. Imports all individual route files from the `routes/` directory and mounts them onto the main Express application router.
├── scripts/                  # Utility or maintenance scripts that are not part of the main application flow but are useful for development, deployment, or administrative tasks (e.g., data seeding, cleanup tasks).
└── views/                    # Server-side rendered views (e.g., using EJS, Pug). Less common if `qa_ops` is a pure JSON API, but might be used for simple status pages or admin interfaces.
```

**Best Practices for `qa_ops`:**

*   **Constants (`config/constants.js`):**
    *   This is the **single source of truth**. Any data, mapping, or configuration that might be used in multiple places or is critical for consistency (e.g., product types, status enums, role definitions) MUST be defined here.
    *   If the frontend (`qa_ops_dashboard`) needs access to these constants, create specific, lean API endpoints in `qa_ops` (e.g., in a `constantsController.js`) to expose only the necessary constants. **Do NOT duplicate backend constants directly in the frontend codebase.**
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
*   **Routes:** Group routes logically in the `routes/` directory (e.g., `userRoutes.js`, `productRoutes.js`). Use descriptive naming. Version your APIs (e.g., `/api/v1/resource`, `/api/v2/resource`) if introducing breaking changes.
*   **Models:** Define clear, well-documented Sequelize models. Utilize associations effectively. Add instance or static methods for common data operations related to that model.
*   **Error Handling:** Implement robust, centralized error handling middleware in `app.js` or a dedicated error handling module. This middleware should catch errors, log them, and send user-friendly JSON error responses with correct HTTP status codes.
*   **API Design:** Adhere to RESTful principles. Use standard HTTP verbs correctly. Design clear, consistent, and predictable URL structures.
*   **Validation:** Use Joi schemas in `params/` for all incoming request data. Validate early in the controller or via dedicated validation middleware.

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
    ├── App.jsx                   # Main application component: sets up React Router, global context providers (e.g., `AuthContext`, `ThemeContext`), and overall layout structure.
    ├── main.jsx                  # Entry point for the React application: renders the root `App` component into the DOM.
    ├── assets/                   # Static assets like images (png, svg), fonts, or other media files imported by components.
    ├── components/               # Reusable UI components, organized by feature or commonality.
    │   ├── common/               # General-purpose, highly reusable components used across multiple features (e.g., `Button.jsx`, `Modal.jsx`, `Spinner.jsx`, `Table.jsx`, `Input.jsx`).
    │   ├── reporting/            # Components specifically for the "Reporting" feature. This structure is a good model for other features.
    │   │   ├── styles/           # CSS Modules (e.g., `ReportView.module.css`) or styled-components for `reporting` components, ensuring scoped styles.
    │   │   ├── utils/            # Utility functions specific to the reporting feature (e.g., `reportDataFormatter.js`, `chartHelper.js` in `reportingUtils.js`).
    │   │   ├── views/            # Larger, container-like components (smart components) that represent a specific view or page within the reporting feature (e.g., `ReportView.jsx`). They compose smaller components and manage feature-specific state and data fetching via `api/`.
    │   │   ├── api/              # Functions dedicated to making API calls related to the reporting feature (e.g., `reportingAPI.js`). These functions typically use the centralized `apiClient`.
    │   │   └── ReportSpecificComponent.jsx # A smaller, presentational component with a single, specific use case within the reporting feature.
    │   └── gtg/                  # Components for the GTG (Good To Go) dashboard. Follow a similar sub-structure as `reporting/` (api/, styles/, utils/, views/).
    │       ├── styles/
    │       ├── utils/
    │       ├── views/
    │       └── api/
    ├── constants/                # Frontend-specific constants.
    │   └── appConstants.js       # UI text, theme configurations (colors, fonts), animation timings, local storage keys, etc. **Avoid duplicating backend constants here; fetch them via API.**
    ├── contexts/                 # React Context API providers and consumers for managing global or shared state (e.g., `AuthContext.jsx`, `ThemeContext.jsx`, `NotificationContext.jsx`).
    ├── hooks/                    # Custom React hooks that encapsulate reusable stateful logic (e.g., `useForm.js`, `useFetch.js`, `useLocalStorage.js`, `useDebounce.js`).
    ├── layouts/                  # Layout components that define the overall structure of pages (e.g., `MainLayout.jsx` with Header, Sidebar, Footer; `AuthLayout.jsx` for login/signup pages).
    ├── pages/                    # Top-level page components that are directly mapped to routes. These components often compose layout components and feature-specific `views/` components. (e.g., `HomePage.jsx`, `ReportingPage.jsx`, `GtgDashboardPage.jsx`).
    ├── services/                 # API service layers.
    │   └── apiClient.js          # Centralized API client setup (e.g., an Axios instance) with base URL, default headers, and interceptors for request/response handling (e.g., adding auth tokens, global error handling like redirecting on 401s).
    ├── styles/                   # Global styles, theme definitions (e.g., CSS variables), CSS resets, or utility CSS classes (e.g., `global.css`).
    ├── utils/                    # Global utility functions that are not specific to any single feature or component (e.g., `dateFormatter.js`, `validationUtils.js`, `localStorageHelper.js`).
    └── router/                   # React Router configuration.
        └── index.js              # Defines application routes using React Router, including protected routes (e.g., checking `AuthContext`), nested routes, and lazy loading of page components.
```

**Best Practices for `qa_ops_dashboard`:**

*   **Component Structure (inspired by `src/components/reporting/`):**
    *   **Feature-Based Modularity:** Group components, API calls, utils, and styles by feature (e.g., `reporting/`, `gtg/`, `userManagement/`).
    *   **`views/`**: These are smart/container components. They fetch data (via functions from the feature's `api/` directory like `reportingAPI.js`), manage state for that feature view, and compose smaller, dumber UI components.
    *   **`styles/`**: Use CSS Modules (`*.module.css`) for component-level styling to prevent class name collisions and ensure encapsulation. For global styles, use `src/styles/global.css`.
    *   **`utils/`**: Place utility functions specific to a feature (e.g., `reportingUtils.js`) within its directory. Global utilities go into `src/utils/`.
    *   **`api/`**: Isolate all API call logic for a feature here (e.g., `reportingAPI.js`). These functions should use the central `apiClient.js` from `src/services/`.
    *   **Single Use Case Components:** Strive to create components with a single, clear responsibility. This enhances reusability, testability, and maintainability.
*   **Constants (`src/constants/appConstants.js`):**
    *   Use for UI-specific text, theme settings, local storage keys, etc.
    *   **Crucially:** For any data that needs to be consistent with the backend (e.g., product types, status codes, role definitions), **fetch it from `qa_ops` API endpoints.** Do not hardcode or duplicate backend constants.
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
            // Handle error appropriately, maybe show a notification via NotificationContext
            return {};
          }
        };

        // In a component (e.g., src/components/reporting/views/ReportFilters.jsx):
        // import { fetchProductLines } from '../api/reportingAPI';
        // const [productLines, setProductLines] = useState({});
        // useEffect(() => {
        //   fetchProductLines().then(data => setProductLines(data));
        // }, []);
        ```
*   **State Management:**
    *   For simple local component state, use `useState` and `useReducer`.
    *   For cross-component state or global state, use React Context (`src/contexts/`) (e.g., `AuthContext`, `ThemeContext`) for moderately complex scenarios.
    *   For more complex global state management, consider libraries like Zustand or Redux Toolkit if the need arises.
*   **API Calls:**
    *   Centralize API interaction logic. Use the `apiClient.js` in `src/services/` for base Axios configuration, including interceptors for adding auth tokens to requests and handling global errors (e.g., logging, showing notifications, or redirecting on 401/403 errors).
    *   Feature-specific API functions should reside in the feature's `api/` directory.
*   **Styling:**
    *   Prefer CSS Modules for component-scoped styles.
    *   Use global styles in `src/styles/global.css` for base styling, resets, and CSS theme variables.
*   **GTG Dashboard Extension (for Speedboat teams):**
    *   **Identify Reusability:** Before creating new GTG components for Speedboat, thoroughly examine existing Mothership GTG components in `src/components/gtg/`.
    *   **Adapt and Parameterize:** Adapt existing components (e.g., in `src/components/gtg/views/GtgStatusView.jsx`) to handle data from both Mothership and Speedboat. This might involve:
        *   Passing a `productLine` prop to components.
        *   Making API calls in `src/components/gtg/api/gtgAPI.js` conditional or parameterized based on `productLine`.
        *   Using conditional rendering within components to display Speedboat-specific information or UI elements.
    *   **New Components:** Only create new components if Speedboat requirements are fundamentally different. Place them within the `src/components/gtg/` structure.
    *   **Universal Data Models:** Ensure backend data structures for GTG are consistent for both product lines to simplify frontend logic.

## Mothership vs. Speedboat

*   **Mothership:** Refers to our legacy product lines. The codebase for these is generally more mature and established.
*   **Speedboat:** Refers to newly added product lines. Development for Speedboat should prioritize leveraging and extending the existing `qa-components` infrastructure rather than building from scratch.

**Key Strategy:**

*   **Maximize Code Reuse:** Before writing any new code for a Speedboat feature, **always** investigate if similar functionality exists for Mothership (in both `qa_ops` and `qa_ops_dashboard`). Adapt, extend, and reuse existing code, components, API endpoints, and backend logic whenever feasible.
*   **Universal Flow & Logic:** Strive to maintain the same or very similar internal methods, data structures, and user flows for both Mothership and Speedboat features.
    *   **Example:** If a reporting workflow for Mothership (e.g., in `qa_ops_dashboard/src/components/reporting/views/MainReportView.jsx`) involves fetching data via `reportingAPI.js`, transforming it with `reportingUtils.js`, and then displaying it, the Speedboat reporting workflow should follow this same pattern.
*   **Configuration-Driven Design:** Where differences are unavoidable, make components and backend logic configuration-driven.
    *   Pass parameters (e.g., `productLine`) to functions or components.
    *   Use conditional logic based on these parameters.

**Example Scenario (Revisited):**

If `qa_ops_dashboard` has a reporting feature for Mothership products displaying metrics X, Y, and Z, and you need to add reporting for Speedboat products displaying metrics X, Y, and W:

1.  **Backend (`qa_ops`):**
    *   **Modify Existing Endpoints:** Adapt existing API endpoints in `qa_ops/routes/reportRoutes.js` (e.g., `/api/reports/summary`) to accept a `productLine` query parameter.
    *   **Adapt Logic:** The corresponding controller in `qa_ops/controllers/reportController.js` and helpers in `qa_ops/helpers/reportHelpers.js` should use this `productLine` to fetch and aggregate data.
    *   **Consistent Response Structure:** Aim for a consistent JSON response structure.
2.  **Frontend (`qa_ops_dashboard`):**
    *   **Reuse Components:** Reuse existing components from `src/components/reporting/`.
    *   **Parameterized API Calls:** Modify API calls in `src/components/reporting/api/reportingAPI.js` to include the `productLine`.
    *   **Conditional Rendering:** In `src/components/reporting/views/ReportView.jsx`, use the fetched data and `productLine` prop to conditionally render metric Z or W.

By following these detailed guidelines and leveraging GitHub Copilot effectively with targeted prompts, we can build robust, scalable, and maintainable solutions within the `qa-components` ecosystem. Always critically review and test Copilot's suggestions before committing them.
