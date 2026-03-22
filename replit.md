# HR Management System

## Overview
This is a full-stack HR Management System designed to manage various HR functions including employees, attendance, leaves, payroll, expenses, assets, and organizational announcements. It provides a comprehensive dashboard for HR administrators to oversee workforce operations, track the employee lifecycle, and manage daily HR processes, streamlining HR operations from onboarding to exit. Key capabilities include employee lifecycle management, a detailed payroll system with statutory compliance, loan management with multi-level approval workflows, comprehensive communication and notification features, document management, overtime and comp-off management, shift management, configurable attendance and leave policies, remote login/geolocation tracking, and template-based letter generation. The system aims to provide tools for compliance, employee self-service, and detailed reporting.

## User Preferences
Preferred communication style: Simple, everyday language.

## System Architecture

### Frontend
- **Framework**: React 18 with TypeScript.
- **Styling**: Tailwind CSS, `shadcn/ui` (New York style variant).
- **Forms**: React Hook Form with Zod for validation.
- **Data Visualization**: Recharts for charts.
- **Design**: Page-based architecture with shared components, protected routes, sidebar navigation, and lazy loading.

### Backend
- **Runtime**: Node.js with Express.js (TypeScript, ES modules).
- **API Design**: RESTful API with Zod schemas for validation.
- **Build Tool**: Vite (frontend) and esbuild (server).

### Data Storage
- **Database**: PostgreSQL.
- **ORM**: Drizzle ORM with `drizzle-zod` for schema integration and Zod validation.
- **Schema Management**: Database table definitions centralized, Drizzle Kit for migrations.
- **Key Entities**: Supports employees, departments, attendance, leave requests, payroll, expenses, assets, announcements, onboarding tasks, exit records, holidays, documents, positions, tax declarations, and salary structures.

### Authentication
- **Provider**: Replit Auth (OpenID Connect).
- **Session Management**: Express-session with `connect-pg-simple` (PostgreSQL-backed).
- **Access Control**: Passport.js with OIDC strategy and `isAuthenticated` middleware for route protection. Role-based access control.

### Core Features
- **Employee Lifecycle Management**: Onboarding, daily operations (attendance, leaves, payroll), and exit management.
- **Payroll System**: Supports salary structures, statutory compliance (PT, LWF, Form 16, TDS, Income Tax calculation for old/new regimes), loan EMI auto-deduction, variable pay, and birthday allowances.
- **Loans**: 3-level approval workflow (Reporting Manager → VP → Finance) with a maximum 12-month repayment term.
- **Communication & Notifications**: Email notifications (via nodemailer SMTP) for various events (onboarding, payslips, leave/loan requests, announcements, birthdays/anniversaries). Chatter notifications with @mention.
- **Document Management**: Uploading and syncing onboarding documents, general storage, employee self-service access, and search functionality.
- **Overtime Tracking**: Auto-detection, manual request creation, approval workflow, and payroll integration.
- **Comp-Off Management**: Request workflow for weekend/holiday work, manager/admin approval, CO leave balance crediting, and 60-day expiry.
- **Shift Management**: Admin interface for creating and assigning work shifts with configurable timings and grace periods.
- **Attendance & Leave Policy**: Configurable policies, detailed tracking, grace periods, policy-specific deductions. Attendance cycle (26th-25th) for LOP calculation, with manual override. Regularization for missing swipes. Multiple check-in/check-out.
- **Remote Login/Geo-Location**: Browser geolocation capture for employees with "remote" or "hybrid" `locationPermission`.
- **Attendance Exempt**: Option to exempt employees from attendance tracking.
- **ESS Personal Details Tab**: Employees can edit personal information, triggering profile change requests for HR approval.
- **On Duty (OD) Requests**: Two-level approval workflow (Reporting Manager → VP/HOD). Approved OD dates are excluded from LOP.
- **Letter Generation**: Template-based generation of HR letters (Offer, Appointment, Confirmation, Experience).
- **Statutory Compliance**: Management of Professional Tax and Labour Welfare Fund rules by state, and Form 16 generation.
- **Company Policies**: Admin page for uploading, managing, and tracking employee acknowledgments of company policies.
- **Salary Journal Entries**: Post-payroll journal entry generation with configurable GL account mappings. Entries follow the JOURNALV/SALARY format with debit/credit amounts, location codes, and payee names. Exportable as CSV. Available via external API.
- **Integrations Management**: Real-time CRUD for biometric devices and ERP/Tally integrations, API key management (view/copy/reset), and GL account mapping configuration.
- **External API**: Secured with API key (`X-API-Key` header). Endpoints for projects, payroll, employees, salary structures, attendance, departments, loans, and journal entries.
- **Multi-Entity (Multi-Company) Support**: `entities` table with name, code, legalName, logo, address, statutory, bank, payslip config. `entityId` on employees, departments, salary_structures, projects. EntityProvider context (`client/src/lib/entityContext.tsx`) with sidebar entity selector. All pages filter employees/departments by selected entity. Payslip/salary register templates dynamically use entity name/address. Entity Management admin page for CRUD.
- **Performance Optimizations**: In-memory caching for employee data, optimized date-range attendance queries, targeted lookups, and database indexing.

## External Dependencies

### Database
- **PostgreSQL**
- **connect-pg-simple**

### Authentication
- **Replit Auth**

### UI/UX Libraries
- **shadcn/ui**
- **Radix UI**
- **Lucide React**

### Data Utilities
- **date-fns**
- **Zod**
- **drizzle-zod**