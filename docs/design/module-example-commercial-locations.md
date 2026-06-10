# Reference module: Commercial Locations

> **Docs only — this module does NOT ship in the base.** It is the canonical
> example of the *customer-defined collection + retriever tool* archetype
> (ARCHITECTURE §8.3). Copy this shape when a client asks for "can the bot
> answer about our branches / catalog / pricing / policies?".

## What it does

The company maintains a table of store locations from the admin; the agent
gains a `store_lookup` tool that answers "¿tienen tienda en Miraflores?".

## Anatomy (everything lives in one folder)

```
app/modules/locations/
├── __init__.py      # module = LocationsModule()  ← discovered at startup
├── models.py        # SQLModel table(s) + Alembic migration
└── admin.py         # optional admin CRUD routes
```

```python
# app/modules/locations/models.py
class StoreLocation(SQLModel, table=True):
    __tablename__ = "store_locations"
    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    city: str = Field(index=True)
    address: str
    hours: str | None = None
    notes: str | None = None
```

```python
# app/modules/locations/__init__.py
from langchain.tools import ToolRuntime, tool
from pydantic import BaseModel, Field
from sqlmodel import select

from app.services.agent_context import TurnContext
from .models import StoreLocation


class StoreLookupInput(BaseModel):
    city: str = Field(description="Ciudad donde el usuario busca una tienda")


@tool(args_schema=StoreLookupInput)
async def store_lookup(city: str, runtime: ToolRuntime[TurnContext]) -> str:
    """Busca las tiendas de la empresa en una ciudad.

    Usa esta herramienta cuando el usuario pregunte dónde hay tiendas,
    direcciones, u horarios de atención de una sede.
    """
    session = runtime.context.session
    result = await session.exec(
        select(StoreLocation).where(StoreLocation.city.ilike(f"%{city}%"))
    )
    stores = result.all()
    if not stores:
        return f"No hay tiendas registradas en {city}."
    return "\n".join(f"- {s.address} ({s.hours or 'horario no publicado'})" for s in stores)


class LocationsModule:
    """Store locations the agent can look up."""

    name = "locations"

    def register_tools(self):
        return [store_lookup]

    # Optional hooks:
    def register_models(self):                # tables → include in Alembic
        return [StoreLocation]

    # def register_admin_routes(self, router): ...  # admin CRUD (Sprint 5 UI)
    # def config_schema(self): ...                  # per-project settings


module = LocationsModule()
```

## The contract recap

| Hook | Required | Purpose |
|------|----------|---------|
| `name` | ✓ | unique module id (used in `agent_config.tool_config`) |
| `register_tools()` | ✓ | LangChain tools fed to the agent |
| `register_models()` | – | SQLModel tables (add an Alembic migration in your project) |
| `register_admin_routes(router)` | – | admin CRUD endpoints backing a panel page |
| `config_schema()` | – | Pydantic model for per-project settings |

## Rules of the game

- **Self-contained:** the module folder owns its tools, tables, and admin
  routes. The core is never edited.
- **Runtime access** through `runtime.context` (`TurnContext`): DB `session`,
  `contact_id`, `conversation_id`, the editable `config`.
- **Enable/disable without redeploy:** `agent_config.enabled_tools`
  (`{"store_lookup": false}` hides it from the model that same turn).
- **Errors are safe:** a crash inside the tool becomes an error `ToolMessage`
  (ToolErrorMiddleware) — the agent apologizes and keeps going.
- Tools return **plain serializable text/data** the model can read.

## Roadmap note

A semi-generic version of this archetype (admin defines a schema → fills
rows → a retriever tool is auto-exposed) is planned so the common case
becomes configuration instead of code.
