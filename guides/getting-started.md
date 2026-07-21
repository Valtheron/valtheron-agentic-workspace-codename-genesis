### 📖 Onboarding Setup: Local Sandbox Environment Installation

*Note: Refactored offline from disorganized setup txt instructions into markdown standards.*

#### 1. Executive Summary
This document outlines standard prerequisite steps, secret environment variable configurations, and seeding commands required to spin up the **Valtheron Workspace** for local engineering and evaluation.

#### 2. Installation & Prerequisites
Before executing local setup routines, verify that your machine environment complies with the following infrastructure baselines:
- **Node.js Environment:** Target Node.js Version **18 LTS** or higher.
- **Active NVM (Node Version Manager):** Run `nvm use 18` to align Node runtime versions.

#### 3. Standard Setup Sequence
```bash
# 1. Boot local package installation routines
npm install

# 2. Re-trigger database migrations and apply seed records
npm run seed

# 3. Spin up the high-concurrency local development server
npm run dev
```

#### 4. Required Secret Environment Guidelines
Create a `.env` file at your workspace root containing the following variables. Never commit this file to git:
```env
# .env Configuration Standard
PORT=3000
SECRET_KEY=highly_confidential_key
DATABASE_FILE=database.sqlite
```

#### 5. Debugging & Common Issues
- **database.json warnings:** Do not commit database data logs. Add `database.json` and `.db` to `.gitignore`.
- **Node incompatibility errors:** If native dependency compiles crash, run `nvm install 18 && nvm use 18` to clean target states.