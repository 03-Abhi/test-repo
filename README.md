# GitHub Copilot Instructions for qa-components

This document provides best practices and guidelines for developing within the `qa-components` repository, with a particular focus on leveraging GitHub Copilot effectively. Our goal is to maintain a high-quality, consistent, and maintainable codebase.

## General Principles

*   **Universality:** Strive to create universal solutions. If a logic or component developed for "Mothership" (legacy products) can be applied to "Speedboat" (new products), adapt and reuse it. Aim for consistent internal methods and data flows across different product lines. This reduces redundancy and ensures a cohesive user experience.
*   **Single Source of Truth:** Critical data, mappings, or configurations that are used in multiple places (especially between frontend and backend) should be defined once. Preferably, define them in backend constants (`qa_ops/config/constants.js`) and expose them to the frontend via an API. This minimizes duplication, prevents inconsistencies, and ensures maintainability.
*   **Modularity and Reusability:** Design components (React) and modules (Node.js) to be highly reusable and have a single, well-defined responsibility. This makes the codebase easier to understand, test, and scale.
*   **Readability and Maintainability:** Write clean, well-commented, and consistently formatted code. Copilot can assist with generating JSDoc comments, explaining code blocks, and suggesting refactorings for clarity.

## Project Structure and Development Guidelines

**Note:** The filenames and specific component names mentioned in the examples below are for illustrative purposes. Actual filenames and structures may vary within the project. Always refer to the current codebase for the most accurate information.

### 1. `qa_ops` (Backend - Node.js API)

The `qa_ops` service is a Node.js application using the Express framework. It serves as the backend API, handling business logic, data processing, interactions with databases and communication with other internal or external services.

**Key Directory Structure:**

```
qa_ops/
├── README.md                 # Overview of the qa_ops service, setup instructions, and key information.
├── app.js                    # Main application setup: Initializes Express, mounts middleware (e.g., body-parser, cors), error handlers, and basic security configurations.
├── branch_recreation_scheduler/ # Contains scripts and logic related to automated branch recreation processes, likely scheduled tasks.
├── config/                   # Configuration files for various aspects of the application.
│   ├── constants.js          # CRITICAL: Defines application-level constants, frequently used static data, mappings (e.g., status codes, type definitions), and non-sensitive configurations. This is the single source of truth for such data.
│   ├── database.js           # Database connection configurations.
├── controllers/              # Request handlers (Express route handlers). Controllers orchestrate application logic by processing incoming requests and coordinating actions. They typically group related functionalities (e.g., `userController.js` for user management, `reportController.js` for reporting, `actions.js` for various automated operations). Responsibilities include:
│                             #   - Receiving and validating request data (often using Joi schemas from `params/`).
│                             #   - Invoking service modules or helper functions from `helpers/` to execute business logic.
│                             #   - Constructing and sending appropriate HTTP responses.
├── ecosystem.config.js       # PM2 process manager configuration file. Defines how the application should be run, monitored, and managed in production (e.g., clustering, environment variables, logging).
├── helpers/                  # Utility functions and helper modules that encapsulate reusable business logic, data transformations, or interactions with external services. (e.g., `dateFormatter.js`, `authHelper.js`, `dbQueryHelper.js`). These are crucial for keeping controllers lean.
├── index.js                  # Entry point of the application. 
├── jenkins_metrics/          # Modules and scripts specifically for collecting, processing, or exposing Jenkins-related metrics.
├── migrations/               # Database migration scripts (e.g., using Sequelize CLI). Each file defines changes to the database schema (creating tables, adding columns, etc.) and allows for version control of the database structure.
├── models/                   # Defines database schemas, particularly for MongoDB using Mongoose, or contains configurations for direct SQL/MySQL interactions. Each file might represent a collection/table schema (e.g., `User.js` for a 'users' collection) defining fields, types, and potentially Mongoose schema methods.
├── package.json              # Node.js project manifest: lists dependencies, development dependencies, scripts (e.g., for starting, testing, linting), and project metadata.
├── params/                   # Potentially for request parameter configurations or other specific parameters. (Note: Current usage might differ from typical validation schema storage, e.g., may hold connection strings or specific operational flags).
├── public/                   # Static assets that can be served directly by Express (e.g., images, documentation files). Less common for pure APIs.
├── routes/                   # API route definitions. Routes are typically grouped by resource or feature into separate files (e.g., `userRoutes.js`, `reportRoutes.js`). Each file defines URL patterns and maps them to controller actions.
├── routes.js                 # Main router aggregation file. Imports all individual route files from the `routes/` directory and mounts them onto the main Express application router.
├── scripts/                  # Utility or maintenance scripts that are not part of the main application flow but are useful for development, deployment, or administrative tasks (e.g., data seeding, cleanup tasks).
└── views/                    # Server-side rendered views (e.g., using EJS, Pug). Less common if `qa_ops` is a pure JSON API, but might be used for simple status pages or admin interfaces.
```

**Best Practices for `qa_ops`:**

*   **Constants (`config/constants.js`):**
    *   This is the **single source of truth**. Any data, mapping, or configuration that might be used in multiple places or is critical for consistency (e.g., product types, status enums, role definitions, default branch names, external service URLs) MUST be defined here.
    *   If the frontend (`qa_ops_dashboard`) needs access to these constants, create specific, lean API endpoints in `qa_ops` (e.g., in a `constantsController.js`) to expose only the necessary constants. **Do NOT duplicate backend constants directly in the frontend codebase.**
    *   **Example:**
        ```javascript
        // qa_ops/config/constants.js Example:
        // ...
        exports.BASE_BRANCH = 'pre_prod'; // Default base branch for pre-production
        exports.GITHUB_OWNER = 'browserstack';
        exports.SLACK = {
          channel: "team-qa-deploys",
          // ... other slack related constants
        };
        // ...

        // Example of internal usage (e.g., in qa_ops/controllers/actions.js):
        const constants = require('../config/constants'); // Or: const { BASE_BRANCH, GITHUB_OWNER } = require('../config/constants');
        const { send_slack_text_message } = require('../helpers/slack'); // Helper that might use SLACK constants

        async function recreatePreProd(repo) {
          const defaultBranch = constants.BASE_BRANCH;
          const owner = constants.GITHUB_OWNER;
          logger.info(`Recreating pre-prod for repo: ${owner}/${repo}, from branch: ${defaultBranch}`);
          // ... logic using defaultBranch and owner ...

          // Example of using another constant indirectly or directly
          await send_slack_text_message(constants.SLACK.channel, `Recreation of ${repo} ${defaultBranch} started.`);
        }
        // ...

        // For exposing constants to the frontend (e.g., in a qa_ops/controllers/constantsController.js):
        // (Retain this pattern for frontend needs)
        const { RELEVANT_CONSTANT_FOR_FRONTEND } = require('../config/constants');

        exports.getRelevantConstant = (req, res) => {
          res.json(RELEVANT_CONSTANT_FOR_FRONTEND);
        };
        ```
*   **Controllers:** Keep controllers lean and focused on request/response handling. Delegate complex business logic to `helpers/` or dedicated service modules. Ensure proper error handling and consistent response formats.
*   **Routes:** Group routes logically in the `routes/` directory (e.g., `userRoutes.js`, `productRoutes.js`). Use descriptive naming. Version your APIs (e.g., `/api/v1/resource`, `/api/v2/resource`) if introducing breaking changes.
*   **Models/Data Access:** When using Mongoose with MongoDB, define clear, well-documented schemas in the `models/` directory. For direct SQL/MySQL interactions (e.g., for GTG), ensure database interaction logic (often found in `helpers/` like `mysql.js` or `dbQueryHelper.js`) is well-organized, secure (e.g., using parameterized queries to prevent SQL injection), and efficient. Connection configurations are typically in `config/database.js` or `config/constants.js`.
*   **Error Handling:** Implement robust, centralized error handling middleware in `app.js` or a dedicated error handling module. This middleware should catch errors, log them, and send user-friendly JSON error responses with correct HTTP status codes.
*   **API Design:** Adhere to RESTful principles. Use standard HTTP verbs correctly. Design clear, consistent, and predictable URL structures.

### 2. `qa_ops_dashboard` (Frontend - React)

The `qa_ops_dashboard` is a React application, likely built with Vite, providing the user interface for interacting with the `qa_ops` API and visualizing data. It enables users to manage QA operations, view reports, and monitor various metrics.

**Key Directory Structure (within `ops_dashboard/src/`):**

```
qa_ops_dashboard/ops_dashboard/
├── README.md                 # Overview of the dashboard, setup, and development guidelines.
├── index.html                # Main HTML file, entry point for the Vite/React application.
├── package.json              # Lists frontend dependencies (React, Vite, Axios, etc.), dev dependencies, and scripts (start, build, test, lint).
├── vite.config.js            # Vite build configuration: defines plugins, server options, build output, aliases, etc.
├── public/                   # Static assets (images, fonts, manifest.json) that are copied directly to the build output folder.
└── src/                      # Contains all the source code for the React application.
    ├── App.jsx                   # Main application component: responsible for setting up React Router, global context providers (e.g., `AuthContext`, `ThemeContext`), and the overall layout structure of the application.
    ├── main.jsx                  # The entry point for the React application. This file renders the root `App` component into the DOM.
    ├── assets/                   # Stores static assets such as images (PNG, SVG, JPG), fonts (WOFF, TTF), icons, or other media files that are imported and used by various components across the application.
    ├── components/               # Houses all React components, organized by feature or commonality.
    │   ├── commonComponents      # General-purpose, highly reusable UI components used across multiple features (e.g., `Header.jsx`, `Home.jsx`, `Docs.jsx`, `Merge.jsx`).
    │   ├── reporting/            # Components specifically for the "Reporting" feature.
    │   │   ├── api/              # Functions for API calls related to reporting (e.g., `apis.js`).
    │   │   ├── styles/           # CSS Modules (e.g., `ReportView.module.css`) for reporting components.
    │   │   ├── utils/            # Utility functions for the reporting feature (e.g., `utils.js`).
    │   │   ├── views/            # Container components for reporting views (e.g., `ReportingHome.jsx`).
    │   │   └── ReportSpecificComponent.jsx # Smaller, presentational components for reporting.
    │   ├── new_gtg_dashboard/    # Components for the GTG (Good To Go) dashboard.
    │   │   ├── api/              # API calls for GTG feature (e.g., `gtgAPI.js`).
    │   │   ├── styles/           # Styles for GTG components (e.g., `GtgStatusViewcss`).
    │   │   ├── utils/            # Utility functions for GTG feature.
    │   │   ├── views/            # View components for GTG feature (e.g., `GtgStatusView.jsx`).
    │   │   └── GtgSpecificComponent.jsx # Smaller components for GTG.
    │   └── (other_features)/     # Additional feature-specific directories (e.g., `userManagement/`, `userProfile/`, `settings/`) follow a similar structure, each containing their own `api/`, `styles/`, `utils/`, `views/`, and specific component files as needed.
    ├── constants.js              # Holds frontend-specific constants.
    └── Router.jsx                # Defines all application routes using `react-router-dom`. It handles basic authentication by checking for a token (via `useToken` hook) and rendering a `Login` component if no token is present. It then defines various application paths and maps them to their respective components within a `<Switch>` statement. Also includes a default route.
```

**Best Practices for `qa_ops_dashboard`:**

*   **Component Structure (inspired by `src/components/reporting/`):**
    *   **Feature-Based Modularity:** Group components, API calls, utils, and styles by feature (e.g., `reporting/`, `gtg/`, `userManagement/`).
    *   **`styles/`**: Use CSS Modules (`*.module.css`) for component-level styling to prevent class name collisions and ensure encapsulation. For global styles, use `index.css` at the root of the `src/` folder.
    *   **`utils/`**: Place utility functions specific to a feature (e.g., `reportingUtils.js`) within its directory. Global utilities go into `src/utils/`.
    *   **`api/`**: Isolate all API call logic for a feature like this here (e.g., `gtgApis.js`).
    *   **Single Use Case Components:** Strive to create components with a single, clear responsibility. This enhances reusability, testability, and maintainability.
*   **Constants (`src/constants/appConstants.js`):**
    *   Use for UI-specific text, theme settings, local storage keys, etc.
    *   **Crucially:** For any data that needs to be consistent with the backend (e.g., product types, status codes, role definitions, `ALL_GTG_SQUADS`), **fetch it from `qa_ops` API endpoints.** Do not hardcode or duplicate backend constants.
    *   **Example (consuming backend constants):**
        ```javascript
        // qa_ops_dashboard/ops_dashboard/src/services/apiClient.js
        import axios from 'axios';
        import { CONSTANTS } from '../constants'; // Assuming frontend constants are here

        const apiClient = axios.create({
          baseURL: CONSTANTS.API_ENDPOINT, 
          withCredentials: true,
        });

        // Add interceptors for auth, error handling etc. if needed

        export default apiClient;

        // qa_ops_dashboard/ops_dashboard/src/components/gtg/api/gtgAPI.js
        import apiClient from '../../../services/apiClient';

        export const fetchAllGtgSquads = async () => {
          try {
            // that returns: { "squadsString": "accessibility,app_automate,app_live,..." }
            const response = await apiClient.get('/constants/all-gtg-squads');
            // The backend might return it as a string, so split if necessary
            return response.data.squadsString.split(','); // e.g., ['accessibility', 'app_automate', ...]
          } catch (error) {
            console.error("Error fetching all GTG squads:", error);
            // Handle error appropriately
            return [];
          }
        };

        ```
*   **State Management:**
    *   For simple local component state, use `useState`.
*   **API Calls:**
    *   Centralize API interaction logic. Use a configured Axios instance from `src/services/apiClient.js` for base URL, `withCredentials`, default headers, and interceptors (e.g., for adding auth tokens, global error handling).
        ```javascript
        // Example: src/services/apiClient.js
        import axios from 'axios';
        import { CONSTANTS } from '../constants/appConstants'; // Assuming API_ENDPOINT is in appConstants.js

        const apiClient = axios.create({
          baseURL: CONSTANTS.API_ENDPOINT, // Or your actual API base URL from your constants
          withCredentials: true,
          // You can add other default configurations here, like timeout
          // timeout: 10000, // 10 seconds
        });

        export default apiClient;
        ```
    *   Feature-specific API functions should reside in the feature\'s `api/` directory and use this `apiClient`.
*   **Styling:**
    *   Prefer CSS Modules (`*.module.css`) for component-scoped styles.
    *   Use `index.css` in the `src/` directory for base styling, resets, and any global CSS rules or variables.
*   **GTG Dashboard Extension (for Speedboat teams):**
    *   **Identify Reusability:** Before creating new GTG components for Speedboat, thoroughly examine existing Mothership GTG components in `src/components/new_gtg_dashboard/`.
    *   **Adapt and Parameterize:** Adapt existing components (e.g., in `src/components/new_gtg_dashboard/NewGtgDashboard.jsx`) to handle data from both Mothership and Speedboat. This might involve:
        *   Passing a `productLine` prop to components.
        *   Making API calls in `src/components/gtg/api/gtgAPI.js` conditional or parameterized based on `productLine`.
        *   Using conditional rendering within components to display Speedboat-specific information or UI elements.
    *   **New Components:** Only create new components if Speedboat requirements are fundamentally different. Place them within the `src/components/new_gtg_dashboard/` structure.
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

**Example Scenario ():**

If `qa_ops_dashboard` has a reporting feature for Mothership products displaying metrics X, Y, and Z, and you need to add reporting for Speedboat products displaying metrics X, Y, and W:

1.  **Backend (`qa_ops`):**
    *   **Modify Existing Endpoints:** Adapt existing API endpoints in `qa_ops/routes/reportRoutes.js` (e.g., `/api/reports/summary`) to accept a `productLine` query parameter.
    *   **Adapt Logic:** The corresponding controller in `qa_ops/controllers/reportController.js` and helpers in `qa_ops/helpers/reportHelpers.js` should use this `productLine` to fetch and aggregate data.
    *   **Consistent Response Structure:** Aim for a consistent JSON response structure.
2.  **Frontend (`qa_ops_dashboard`):**
    *   **Reuse Components:** Reuse existing components from `src/components/reporting/`.
    *   **Parameterized API Calls:** Modify API calls in `src/components/reporting/api/reportingAPI.js` to include the `productLine`.
    *   **Conditional Rendering:** In `src/components/reporting/views/ReportView.jsx`, use the fetched data and `productLine` prop to conditionally render metric Z or W.


