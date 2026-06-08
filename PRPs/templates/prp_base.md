# PRP: [Feature Name]

> **Version:** 1.0
> **Created:** [Date]
> **Status:** Draft | Ready | In Progress | Completed

---

## Goal
[What needs to be built - be specific about the end state]

## Why
- [Business value and user impact]
- [Integration with existing features]
- [Problems this solves and for whom]

## What
[User-visible behavior and technical requirements]

### Success Criteria
- [ ] [Specific measurable outcome 1]
- [ ] [Specific measurable outcome 2]
- [ ] [Specific measurable outcome 3]

---

## All Needed Context

### Documentation & References
```yaml
# MUST READ - Include these in your context window
- file: backend/CLAUDE.md
  why: Backend architecture and patterns

- file: frontend/CLAUDE.md
  why: Frontend architecture and patterns

- url: [Library documentation URL]
  why: [Specific sections needed]
```

### Current Codebase Structure
```
project/
├── backend/                    # FastAPI backend (port 8090)
│   ├── app/
│   │   ├── core/              # Config, security, dependencies
│   │   ├── models/            # SQLModel database models
│   │   ├── schemas/           # Pydantic request/response
│   │   ├── controllers/       # FastAPI route handlers
│   │   ├── services/          # Business logic layer
│   │   ├── db/                # Database session
│   │   └── main.py            # App initialization
│   └── alembic/               # Database migrations
├── frontend/                   # React frontend (port 5190)
│   └── src/
│       ├── components/
│       │   ├── ui/            # Shadcn/ui primitives
│       │   ├── layout/        # AppLayout, Sidebar, Header
│       │   └── features/      # Feature-specific components
│       ├── pages/             # Route page components
│       ├── hooks/             # Custom React hooks (API calls)
│       ├── lib/               # api-client.ts, utils.ts
│       ├── router/            # React Router config
│       └── types/             # TypeScript interfaces
└── PRPs/                      # Product Requirements Prompts
```

### Desired Structure (files to add/modify)
```bash
# Show new files with comments about responsibility
backend/app/
├── models/[feature].py        # SQLModel models
├── schemas/[feature].py       # Pydantic schemas
├── services/[feature]_service.py  # Business logic
└── controllers/[feature].py   # API endpoints

frontend/src/
├── pages/[Feature]Page.tsx    # Page component
├── hooks/use[Feature].ts      # React Query hooks
└── components/features/[feature]/  # Feature components
```

### Known Gotchas & Project Conventions

```python
# BACKEND CONVENTIONS

# 1. Multi-tenancy: Always filter by organization
async def get_items(session: AsyncSession, org_id: UUID) -> list[Item]:
    result = await session.exec(
        select(Item).where(Item.organization_id == org_id)
    )
    return result.all()

# 2. Auth: Use dependencies for authorization
@router.get("/items")
async def list_items(
    session: AsyncSession = Depends(get_session),
    current_user: AuthUser = Depends(require_role("member")),  # or "admin", "owner"
):
    pass

# 3. Login expects form-data with 'username' field (not 'email')
# OAuth2PasswordRequestForm uses 'username' by convention

# 4. Role hierarchy: owner > admin > member
# Use require_role("admin") for admin AND owner access
```

```typescript
// FRONTEND CONVENTIONS

// 1. API calls use React Query hooks in src/hooks/
export function useItems() {
  return useQuery({
    queryKey: ['items'],
    queryFn: async () => {
      const { data } = await apiClient.get<Item[]>('/items')
      return data
    },
  })
}

// 2. Mutations invalidate related queries
export function useCreateItem() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: (data: CreateItemRequest) =>
      apiClient.post('/items', data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['items'] })
    },
  })
}

// 3. Protected routes use ProtectedRoute component
<ProtectedRoute requiredRole="admin">
  <AdminPage />
</ProtectedRoute>

// 4. API errors have shape: { detail: string, status_code?: number }
```

---

## Implementation Blueprint

### Data Models (Backend)
```python
# backend/app/models/[feature].py
from sqlmodel import Field, SQLModel, Relationship
from uuid import UUID, uuid4
from datetime import datetime

class FeatureModel(SQLModel, table=True):
    __tablename__ = "features"

    id: UUID = Field(default_factory=uuid4, primary_key=True)
    organization_id: UUID = Field(foreign_key="organizations.id", index=True)
    name: str = Field(max_length=255)
    created_at: datetime = Field(default_factory=datetime.utcnow)

    # Relationships
    organization: "Organization" = Relationship(back_populates="features")
```

### Tasks (in execution order)
```yaml
Task 1: Create backend model
  - CREATE: backend/app/models/[feature].py
  - MODIFY: backend/app/models/__init__.py (add export)
  - MIRROR pattern from: backend/app/models/tenant.py

Task 2: Create backend schemas
  - CREATE: backend/app/schemas/[feature].py
  - MIRROR pattern from: backend/app/schemas/organization.py

Task 3: Create backend service
  - CREATE: backend/app/services/[feature]_service.py
  - MIRROR pattern from: backend/app/services/organization_service.py

Task 4: Create backend controller
  - CREATE: backend/app/controllers/[feature].py
  - MODIFY: backend/app/main.py (register router)
  - MIRROR pattern from: backend/app/controllers/organizations.py

Task 5: Create database migration
  - RUN: cd backend && make makemigrations m="add [feature] table"
  - RUN: cd backend && make migrate

Task 6: Create frontend types
  - MODIFY: frontend/src/types/api.ts (add interfaces)

Task 7: Create frontend hooks
  - CREATE: frontend/src/hooks/use[Feature].ts
  - MIRROR pattern from: frontend/src/hooks/useOrganization.ts

Task 8: Create frontend page
  - CREATE: frontend/src/pages/[Feature]Page.tsx
  - MODIFY: frontend/src/router/index.tsx (add route)
  - MODIFY: frontend/src/components/layout/Sidebar.tsx (add nav)

Task 9: Create feature components
  - CREATE: frontend/src/components/features/[feature]/
  - MIRROR pattern from: frontend/src/components/features/organization/
```

---

## Validation Loop

### Level 1: Backend - Syntax & Types
```bash
cd backend

# Run tests (if available)
make test

# Manual API test
make dev  # Start server in another terminal
curl http://localhost:8090/[endpoint]
```

### Level 2: Frontend - Lint & Types
```bash
cd frontend

# TypeScript check + build
npm run build

# Lint
npm run lint
```

### Level 3: Integration Test
```bash
# Start both servers
cd backend && make dev      # Terminal 1
cd frontend && npm run dev  # Terminal 2

# Test full flow in browser at http://localhost:5190
# 1. Login
# 2. Navigate to new feature
# 3. Test CRUD operations
# 4. Check error handling
```

---

## Final Checklist

### Backend
- [ ] Model created with proper relationships
- [ ] Schemas for request/response
- [ ] Service with business logic
- [ ] Controller with proper auth dependencies
- [ ] Router registered in main.py
- [ ] Migration created and applied
- [ ] Manual API test successful

### Frontend
- [ ] Types added to api.ts
- [ ] React Query hooks created
- [ ] Page component created
- [ ] Route added to router
- [ ] Sidebar navigation updated (if needed)
- [ ] `npm run build` passes
- [ ] `npm run lint` passes
- [ ] Manual browser test successful

---

## Anti-Patterns to Avoid

- ❌ Don't skip organization_id filtering (breaks multi-tenancy)
- ❌ Don't use 'email' in login form (use 'username')
- ❌ Don't create auth logic without require_role() dependency
- ❌ Don't make API calls directly - use React Query hooks
- ❌ Don't skip query invalidation after mutations
- ❌ Don't hardcode URLs - use VITE_API_BASE_URL

---

## Notes

[Additional context, decisions made, or future considerations]
