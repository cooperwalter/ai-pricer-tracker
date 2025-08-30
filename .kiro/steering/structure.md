# Project Structure & Organization

## Directory Layout
```
├── src/
│   ├── app/                 # Next.js App Router pages
│   │   ├── layout.tsx       # Root layout with fonts and metadata
│   │   └── page.tsx         # Homepage component
│   ├── server/              # Server-side code
│   │   └── db/              # Database configuration
│   │       ├── index.ts     # Drizzle connection setup
│   │       └── schema.ts    # Database schema definitions
│   ├── styles/              # Global styles
│   │   └── globals.css      # Tailwind imports and theme config
│   └── env.js               # Environment variable validation
├── .kiro/                   # Kiro IDE configuration
│   └── steering/            # AI assistant guidance documents
├── public/                  # Static assets
└── logs/                    # Application logs
```

## Naming Conventions
- **Files**: kebab-case for config files, camelCase for TypeScript
- **Components**: PascalCase React components
- **Database**: Snake_case with project prefix (`ai-pricer-tracker_`)
- **Paths**: Use `~/` alias for src directory imports

## Database Architecture
- **Multi-project schema**: Uses `pgTableCreator` with project prefix
- **Connection pooling**: Cached in development, fresh in production
- **Type safety**: Full TypeScript integration with Drizzle
- **Migrations**: Generated and managed via `drizzle-kit`

## Configuration Files
- **TypeScript**: Strict mode with path aliases and Next.js plugin
- **ESLint**: Flat config with TypeScript and Drizzle rules
- **Prettier**: Tailwind class sorting enabled
- **Drizzle**: PostgreSQL dialect with table filtering
- **Next.js**: Minimal config with environment validation

## Import Patterns
- Use `~/` for internal imports from src directory
- Type imports with inline syntax: `import { type Config }`
- Environment variables through validated `env` object
- Database access through exported `db` instance

## Development Workflow
1. Schema changes in `src/server/db/schema.ts`
2. Generate migrations with `pnpm db:generate`
3. Apply with `pnpm db:push` or `pnpm db:migrate`
4. Type checking with `pnpm check` before commits
5. Format code with `pnpm format:write`