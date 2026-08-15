# JSON Schema Validation

## What It Is

JSON Schema is a standard, declarative format for describing the expected structure of a JSON document — required fields, data types, allowed values, nesting rules. Schema validation checks an actual API response against this definition automatically, replacing dozens of manual field-by-field assertions (see [Request & Response Validation](./request-response-validation.md)) with a single, comprehensive, reusable check.

## Why It Matters

- Manually asserting on every field's presence and type (as shown in the previous note) doesn't scale — a response with 30 fields needs 30+ assertions written by hand; a schema expresses the same contract once, declaratively, and validates the whole structure in one call.
- A schema is a living, checkable contract between API producer and consumer — when a field is accidentally removed or its type changes, schema validation fails immediately and specifically, rather than a consumer discovering it as a confusing downstream bug.
- Schema validation is a standard, expected tool in any serious API testing framework — knowing how to define and apply one (rather than manually asserting every field forever) is a clear signal of scalable test design thinking.

## How It Works

A JSON Schema defines: the expected `type` of the document (object, array), `required` fields, `properties` with their own types/constraints, and rules for nested objects/arrays. Validation libraries (`jsonschema` in Python, `ajv` in TypeScript/Node) compare an actual JSON document against the schema and report every violation, not just the first one — which is more useful for debugging than a single generic assertion failure.

**What schema validation checks well:** structure, required fields, types, allowed value sets (enums), string formats (email, date), numeric ranges.

**What it doesn't check:** business logic correctness (a schema can confirm `total` is a number, but not that it's the *correct* number — that still needs the manual assertions from [Request & Response Validation](./request-response-validation.md)).

## Example

**Python (using the `jsonschema` library):**
```python
from jsonschema import validate, ValidationError

order_schema = {
    "type": "object",
    "required": ["id", "customer_email", "items", "total", "status"],
    "properties": {
        "id": {"type": "integer"},
        "customer_email": {"type": "string", "format": "email"},
        "items": {
            "type": "array",
            "minItems": 1,
            "items": {
                "type": "object",
                "required": ["product_id", "qty", "price"],
                "properties": {
                    "product_id": {"type": "integer"},
                    "qty": {"type": "integer", "minimum": 1},
                    "price": {"type": "number", "minimum": 0},
                },
            },
        },
        "total": {"type": "number", "minimum": 0},
        "status": {"type": "string", "enum": ["pending", "shipped", "delivered", "cancelled"]},
    },
    "additionalProperties": False,   # rejects any field NOT defined in the schema
}

def test_order_response_matches_schema(api_context):
    response = api_context.get("/api/orders/501")
    body = response.json()

    try:
        validate(instance=body, schema=order_schema)
    except ValidationError as e:
        assert False, f"Schema validation failed: {e.message} at {list(e.path)}"
```

**TypeScript (using `ajv`):**
```typescript
import { test, expect } from '@playwright/test';
import Ajv from 'ajv';
import addFormats from 'ajv-formats';

const ajv = new Ajv({ allErrors: true });
addFormats(ajv);

const orderSchema = {
  type: 'object',
  required: ['id', 'customer_email', 'items', 'total', 'status'],
  properties: {
    id: { type: 'integer' },
    customer_email: { type: 'string', format: 'email' },
    items: {
      type: 'array',
      minItems: 1,
      items: {
        type: 'object',
        required: ['product_id', 'qty', 'price'],
        properties: {
          product_id: { type: 'integer' },
          qty: { type: 'integer', minimum: 1 },
          price: { type: 'number', minimum: 0 },
        },
      },
    },
    total: { type: 'number', minimum: 0 },
    status: { type: 'string', enum: ['pending', 'shipped', 'delivered', 'cancelled'] },
  },
  additionalProperties: false,
};

test('order response matches schema', async ({ request }) => {
  const response = await request.get('/api/orders/501');
  const body = await response.json();

  const validateSchema = ajv.compile(orderSchema);
  const valid = validateSchema(body);

  expect(valid, JSON.stringify(validateSchema.errors)).toBe(true);
});
```

The key advantage over manual field-by-field assertions: if three fields are wrong simultaneously, schema validation reports all three violations at once (`allErrors: true` / catching the full `ValidationError`), while a chain of individual `assert` statements would stop at the first failure.

## Production Considerations

- Store schemas as reusable files/modules (not inline per test) and share them across every test hitting the same endpoint — this is what makes schema validation scale, since a single schema update propagates to every test using it.
- Generate schemas from real API documentation (OpenAPI/Swagger specs, if available) rather than hand-writing them from scratch where possible — this reduces the risk of the test schema silently drifting from the actual documented contract.
- Combine schema validation (structure/type) with targeted manual assertions (business logic correctness) — neither replaces the other; using schema validation alone gives false confidence that a structurally valid response is also a *correct* one.

## Common Pitfalls

- Treating schema validation as sufficient on its own and skipping business-logic assertions — a response can be perfectly schema-valid while still being wrong (a total that doesn't match the sum of items, as covered in the previous note).
- Not setting `additionalProperties: false` (or the equivalent) — without it, a schema won't catch unexpected/leaked fields being added to a response, missing a category of bug schema validation is otherwise well-suited to catch.
- Hand-maintaining schemas that drift out of sync with the real API contract over time, especially after backend changes — schemas need the same maintenance discipline as any other test asset.
- Writing an overly loose schema (skipping `required`, using generic types everywhere) just to make tests pass — this defeats the purpose, since a schema that accepts almost anything catches almost nothing.

## Interview Notes

- Be ready to explain what JSON Schema validation is and why it scales better than manual field-by-field assertions for structural/type checking.
- Understand what schema validation does NOT catch (business logic correctness) and be able to explain why manual assertions are still needed alongside it.
- Be able to describe `additionalProperties: false` and why it matters specifically for catching data leakage/unexpected fields — a good, specific example of schema validation's practical security value.

## References

- [JSON Schema — Official Specification](https://json-schema.org/)
- [jsonschema (Python)](https://python-jsonschema.readthedocs.io/)
- [Ajv (TypeScript/Node)](https://ajv.js.org/)