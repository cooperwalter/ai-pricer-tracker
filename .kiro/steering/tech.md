# Technology Stack & Development Guidelines

## Core Stack
- **Framework**: Next.js 15+ with App Router and React 19
- **Language**: TypeScript with strict configuration
- **Database**: PostgreSQL with Drizzle ORM
- **Styling**: Tailwind CSS v4
- **Package Manager**: pnpm (v10.9.0)
- **Runtime**: Node.js with ES modules

## Key Dependencies
- **Database**: `drizzle-orm`, `postgres`, `drizzle-kit`
- **Environment**: `@t3-oss/env-nextjs` with Zod validation
- **Validation**: `zod` for runtime type checking
- **Fonts**: Geist font family

## Development Commands
```bash
# Development
pnpm dev              # Start dev server with Turbo
pnpm build           # Production build
pnpm start           # Start production server
pnpm preview         # Build and start locally

# Code Quality
pnpm lint            # Run ESLint
pnpm lint:fix        # Fix ESLint issues
pnpm typecheck       # TypeScript type checking
pnpm check           # Run lint + typecheck together

# Formatting
pnpm format:check    # Check Prettier formatting
pnpm format:write    # Apply Prettier formatting

# Database
pnpm db:generate     # Generate Drizzle migrations
pnpm db:migrate      # Run migrations
pnpm db:push         # Push schema changes
pnpm db:studio       # Open Drizzle Studio
```

## Code Standards
- **Strict TypeScript**: `noUncheckedIndexedAccess`, `strict` mode enabled
- **ESLint**: Next.js config + TypeScript recommended + Drizzle plugin
- **Prettier**: Tailwind plugin for class sorting
- **Import Style**: Type imports with inline syntax preferred
- **Database Safety**: Drizzle plugin enforces WHERE clauses on updates/deletes

## Environment Configuration
- Uses `@t3-oss/env-nextjs` for type-safe environment variables
- Server/client variable separation enforced
- Runtime validation with Zod schemas
- Empty strings treated as undefined