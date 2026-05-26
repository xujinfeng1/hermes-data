# Structured Output Pattern — Full Implementation

## Core Type
```python
from dataclasses import dataclass, field

@dataclass
class ToolResult:
    """工具返回的统一包装：text 给 LLM 看，structured 给 API 消费"""
    text: str
    structured: Optional[dict] = None

    def __str__(self):
        return self.text
```

## Domain Output Types
```python
@dataclass
class SegmentOutput:
    type: str = "segment"
    conditions: list[dict] = field(default_factory=list)
    sql_where: str = ""
    user_count: int = 0
    user_ids: list[str] = field(default_factory=list)

@dataclass
class CanvasOutput:
    type: str = "canvas"
    audience_name: str = ""
    goal: str = ""
    nodes: list[dict] = field(default_factory=list)

@dataclass
class FunnelOutput:
    type: str = "funnel"
    steps: list[str] = field(default_factory=list)
    counts: list[int] = field(default_factory=list)
    overall_rate: float = 0.0
```

## Tool Implementation
```python
def segment_users(gender=None, min_total_spent=None) -> str:  # return type is str for LLM compat
    # ... filter users ...
    conditions = []
    if gender: conditions.append({"field": "gender", "op": "eq", "value": gender})
    if min_total_spent: conditions.append({"field": "total_spent", "op": "gte", "value": min_total_spent})

    return ToolResult(
        text=f"圈选到 {len(results)} 个用户\n  U001 张三 ...",
        structured=SegmentOutput(
            conditions=conditions,
            sql_where=build_sql_where(conditions),
            user_count=len(results),
            users=user_summaries,
        ).__dict__
    )
```

## Agent Integration
```python
def _execute_tool(name: str, args: dict) -> tuple[str, dict | None]:
    result = fn(**args)
    if isinstance(result, ToolResult):
        return result.text, result.structured
    return str(result), None

# In ReAct loop:
text, struct = _execute_tool(fn_name, fn_args)
if struct:
    structured_data.append(struct)  # goes to API response
messages.append({"role": "tool", "content": text})  # goes to LLM
```

## API Response
```python
# /chat returns ChatResponse with optional structured field
# /chat/structured returns StructuredResponse with top-level sql + json_filter
class StructuredResponse(BaseModel):
    reply: str
    structured: list[dict] = []
    sql: Optional[str] = None       # extracted from segment data
    json_filter: Optional[dict] = None
```

## SQL Builder
```python
OPS = {"eq": "=", "gte": ">=", "lte": "<=", "in": "IN", "contains": "LIKE"}

def build_sql_where(conditions: list[dict], operator: str = "AND") -> str:
    clauses = []
    for c in conditions:
        field, op, value = c["field"], c["op"], c["value"]
        sql_op = OPS.get(op, "=")
        if isinstance(value, str):
            clauses.append(f"{field} {sql_op} '{value}'")
        else:
            clauses.append(f"{field} {sql_op} {value}")
    return f" {operator} ".join(clauses) or "1=1"
```
