# Climate Risk Analyzer

## Overview

This is an AI-powered climate risk analysis platform that evaluates companies against a 44-measure physical climate risk assessment framework. The application imports company lists from Excel files, discovers relevant sustainability documents via Google Search, processes document content, and uses Claude AI to score companies on their climate risk management practices (0-220 points scale).

## User Preferences

Preferred communication style: Simple, everyday language.

## System Architecture

### Frontend Architecture
- **Framework**: React 18 with TypeScript
- **Routing**: Wouter (lightweight React router)
- **State Management**: TanStack React Query for server state caching and synchronization
- **UI Components**: Shadcn/ui component library built on Radix UI primitives
- **Styling**: Tailwind CSS with custom theme variables (emerald/teal climate theme)
- **Build Tool**: Vite with React plugin

The frontend follows a pages-based structure with reusable components. Key pages include a Dashboard for company listing and a CompanyDetail page for viewing analysis results organized by the 9 assessment categories.

### Backend Architecture
- **Runtime**: Node.js with Express
- **Language**: TypeScript with ESM modules
- **API Design**: RESTful endpoints defined in shared route contracts with Zod validation
- **AI Integration**: Anthropic Claude SDK for document analysis against the 44-measure framework

The backend uses a layered architecture with route handlers calling into library modules for specific functions:
- `importer.ts` - Fetches and parses Excel company lists using xlsx
- `discovery.ts` - Searches for company documents via Google Custom Search API
- `processor.ts` - Extracts text from PDFs (pdf-parse) and HTML (cheerio)
- `analyzer.ts` - Orchestrates Claude AI analysis against measure definitions

### Data Storage
- **Database**: PostgreSQL via Drizzle ORM
- **Schema**: Four main tables - companies, documents, analysisResults, measureScores
- **Migrations**: Drizzle Kit for schema management (`db:push` command)

### Shared Code
The `shared/` directory contains code used by both frontend and backend:
- `schema.ts` - Drizzle table definitions and Zod insert schemas
- `routes.ts` - API contract definitions with input/output schemas
- `measures.ts` - Complete 44-measure assessment framework with scoring guidance

## External Dependencies

### APIs and Services
- **Anthropic Claude API**: Powers the AI analysis engine (requires `ANTHROPIC_API_KEY`)
- **Google Custom Search API**: Discovers company sustainability documents (requires `GOOGLE_SEARCH_API_KEY` and `GOOGLE_SEARCH_ENGINE_ID`)
- **PostgreSQL Database**: Requires `DATABASE_URL` environment variable

### Key NPM Packages
- `@anthropic-ai/sdk` - Claude AI integration
- `drizzle-orm` / `drizzle-kit` - Database ORM and migrations
- `xlsx` - Excel file parsing for company imports
- `pdf-parse` - PDF text extraction
- `cheerio` - HTML parsing and text extraction
- `axios` - HTTP client for document fetching
- `@tanstack/react-query` - Frontend data fetching
- `react-markdown` - Rendering analysis summaries
- `recharts` - Data visualization for risk scores

## 44-Measure Assessment Framework

The analysis framework evaluates companies across 9 categories with 44 total measures:

1. **Governance & Strategic Oversight** (M01-M07): Board oversight, management responsibility, ERM integration
2. **Risk Identification & Assessment** (M08-M16): Hazard identification, exposure mapping, scenario analysis
3. **Asset Design & Resilience** (M17-M21): Climate-resilient design, protective measures, maintenance
4. **Crisis Management & Response** (M22-M26): Emergency planning, business continuity, recovery
5. **Supply Chain Resilience** (M27-M31): Supplier risk, redundancy, geographic diversification
6. **Insurance & Risk Transfer** (M32-M35): Coverage quality, gap analysis, claims management
7. **Data Quality & Assurance** (M36-M37): Data governance, external assurance
8. **Social & Community Resilience** (M38-M39): Employee safety, community engagement
9. **Performance Metrics & Continuous Improvement** (M40-M44): Metrics tracking, adaptation ROI

Each measure is scored 0-5 with:
- Evidence summaries explaining the score rationale
- Confidence levels (High/Medium/Low)
- Verbatim quotes from source documents
- Coverage metrics where applicable

The analyzer processes all 44 measures by category, ensuring every measure receives a score even if no evidence is found (score 0 with "No evidence found" summary).