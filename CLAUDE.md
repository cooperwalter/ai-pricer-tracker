# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an AI-powered price tracking application built with the T3 Stack (Next.js, TypeScript, Drizzle ORM, tRPC). The application monitors product prices across multiple e-commerce stores and supports different subscription tiers.

## Development Commands

```bash
# Development
pnpm dev              # Start development server with Turbo
pnpm preview          # Build and start production preview

# Code Quality
pnpm check            # Run linting and type checking
pnpm lint             # Run ESLint
pnpm lint:fix         # Fix linting issues
pnpm typecheck        # Run TypeScript type checking
pnpm format:check     # Check Prettier formatting
pnpm format:write     # Fix formatting issues

# Database
pnpm db:generate      # Generate Drizzle migrations
pnpm db:migrate       # Run database migrations
pnpm db:push          # Push schema to database
pnpm db:studio        # Open Drizzle Studio for database management

# Build
pnpm build            # Build for production
pnpm start            # Start production server
```

## Architecture

### Tech Stack
- **Framework**: Next.js 15 with App Router
- **Language**: TypeScript with strict mode enabled
- **Database**: PostgreSQL with Drizzle ORM
- **Styling**: Tailwind CSS v4
- **Package Manager**: pnpm 10.9.0

### Key Directories
- `/src/app/` - Next.js App Router pages and layouts
- `/src/server/db/` - Database schema and configuration
- `/src/env.js` - Environment variable validation using @t3-oss/env-nextjs

### Database Configuration
- Schema defined in `src/server/db/schema.ts`
- Uses table prefix `ai-pricer-tracker_` for multi-project database support
- Connection configured via `DATABASE_URL` environment variable

### TypeScript Configuration
- Strict mode enabled with `noUncheckedIndexedAccess`
- Path alias: `~/*` maps to `./src/*`
- Target: ES2022

### Environment Variables
Required environment variables are validated in `src/env.js`:
- `DATABASE_URL` - PostgreSQL connection string
- Additional client variables prefixed with `NEXT_PUBLIC_`

## Implementation Notes

The PRODUCT.md file contains detailed specifications for:
- Multi-tenant subscription tier system (free, premium, premium_plus)
- Dynamic queue-based scraping system
- GitHub Actions workflow configurations
- Database schema with user management and price tracking tables
- Queue processing algorithms that respect subscription tiers

When implementing features, refer to PRODUCT.md for detailed requirements and database schema.