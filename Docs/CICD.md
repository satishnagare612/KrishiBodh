Here is the comprehensive, consolidated architecture for **KrishiBodh**, integrating all the technical decisions we have made so far. This blueprint covers the technology stack, the ground-up multilingual strategy, and the multi-tool CI/CD orchestration.

---

### 1. Technology Stack & System Components
Your app suite consists of five distinct components, each chosen for specific strengths:

| Component | Technology Stack | Key Technical Details |
| :--- | :--- | :--- |
| **Mobile Apps** | **React Native** (with Expo) | Chosen over Swift/Kotlin to leverage your team's React expertise, share code, and avoid double development. Expo is used for simplified native builds and OTA (Over-the-Air) updates. |
| **Web App** | **React** | Standard SPA (Single Page Application) architecture, aligning with the mobile team's skillset. |
| **Backend API** | **C# / ASP.NET Core** | Provides robust, strongly typed backend services. Deployed to Azure App Service or Container Apps. |
| **AI Service** | **Python** | Hosts the DeepSeek model interactions and RAG (Retrieval-Augmented Generation) pipelines. Deployed via Azure Container Apps or Azure ML. |
| **Database** | **Azure SQL** | Managed relational database. Requires a Unicode-compliant collation (`Latin1_General_100_CI_AS_SC_UTF8`) to support Devanagari and other scripts. |

---

### 2. Ground-Up Multilingual (i18n) Architecture
Because the app must be multilingual from day one, localization is treated as a first-class citizen across all layers.

*   **Single Source of Truth (TMS):** A Translation Management System (e.g., Crowdin, Phrase, Localazy) acts as the central hub for all translators.
*   **Frontend (React & React Native):** Uses the **`i18next`** ecosystem. Translation strings are stored in JSON files. React Native requires an adapter (like `AsyncStorage`) to persist the user's language choice.
*   **Backend (C#):** Uses **`.resx` resource files** and ASP.NET Core's built-in `RequestLocalizationMiddleware` to serve localized error messages or emails based on the `Accept-Language` header.
*   **AI Service (Python):** Prompts and system instructions are localized. The AI must handle mixed-language queries and code-switching (common in agricultural contexts).
*   **Database (Azure SQL):** Uses **Translation Tables** (e.g., a `CropTranslations` table linked to a base `Crops` table) to store dynamic content like crop names and disease descriptions in multiple languages.

---

### 3. CI/CD & Release Pipeline Architecture
Your team uses both GitHub and Azure DevOps. Instead of choosing one, you will use a **Component-Aware Hybrid Model**:

*   **GitHub Actions (CI):** Owns the code. It handles fast feedback on pull requests—linting, unit tests, and syncing translation strings to the TMS.
*   **EAS (Expo Application Services):** Owns the mobile binary. It handles complex native compilation and code signing (especially for iOS), bypassing the need for macOS build runners in your CI.
*   **Azure DevOps (CD & Orchestration):** Owns the release. It handles enterprise governance, deployment to Azure infrastructure, and database schema updates (`.dacpac`).

| Component | CI Tool | CD/Deployment Tool | How it works together |
| :--- | :--- | :--- | :--- |
| **React Web** | GitHub Actions | Azure DevOps -> Azure Static Web Apps | GH Actions runs tests and pulls translated JSON from TMS. AzDO deploys the build artifact. |
| **React Native** | GitHub Actions | **EAS** (triggered by GH Actions) | GH Actions runs tests, then uses `eas-cli` to trigger a cloud build. EAS signs the app and pushes to App/Play Store. |
| **C# API** | Azure DevOps | Azure DevOps -> Azure App Service | AzDO builds the `.NET` project, pulls `.resx` files from TMS, and deploys. |
| **Python AI** | Azure DevOps | Azure DevOps -> Azure ML/Containers | AzDO packages the Python app with localized prompt templates and deploys. |
| **Azure SQL** | Azure DevOps | Azure DevOps | Compiles a SQL Database Project into a `.dacpac` and deploys it using the `SqlAzureDacpacDeployment` task. |

---

### 4. The Master Release Workflow (Orchestration)
To manage the release of five components, you will use **Path-Filtered Triggers** (e.g., the C# pipeline only runs if files in `/api` change) combined with a **Master Orchestrator Pipeline** in Azure DevOps for full-stack releases.

**A typical release flow looks like this:**
1.  **Code Push**: Developer pushes to GitHub.
2.  **CI Phase**: GitHub Actions runs tests and syncs new English strings to the TMS.
3.  **Translation Phase**: Translators work in the TMS. 
4.  **Build Phase**: Azure DevOps pulls the latest translated files, builds the C# API and Python AI service, and triggers the EAS build for React Native.
5.  **Deployment Phase**: 
    *   Azure DevOps deploys the updated **SQL Database** schema.
    *   Azure DevOps deploys the **C# API** and **Python AI** to Azure.
    *   Azure DevOps deploys the **React Web App** to Static Web Apps.
    *   EAS submits the **React Native** binary to the app stores.
    *   *(Optional)* EAS pushes a translation-only update to mobile devices via OTA, bypassing app store review.

**Key Takeaway for the Team:** You are not forcing a single tool to do everything. GitHub Actions handles rapid code verification, EAS handles the heavy lifting of mobile binaries, and Azure DevOps acts as the secure, governed control tower for Azure deployments and database migrations.
