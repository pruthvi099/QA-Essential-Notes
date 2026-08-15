# API Contract Testing

## What It Is

Contract testing verifies that a producer (the API) and its consumers (frontend apps, other services) agree on the shape and behavior of their interactions — without needing to run the entire integrated system end-to-end. **Consumer-driven contract testing** (the Pact-style approach) inverts the usual direction: the *consumer* defines what it expects from the API, generates a contract from that expectation, and the *producer* verifies it can satisfy that contract independently.

This is distinct from [JSON Schema Validation](./json-schema-validation.md), which validates a single response's structure — contract testing validates the *agreement* between two independently deployed systems, catching breaking changes before they reach a real integration.

## Why It Matters

- In a microservices or frontend/backend-separated architecture, a backend team can change an API in a way that silently breaks a frontend team's integration — full end-to-end tests would eventually catch this, but slowly and expensively; contract tests catch it immediately, in each side's own CI pipeline, without needing both systems running together.
- Consumer-driven contracts flip the traditional testing direction in a genuinely useful way: instead of the producer guessing what consumers need and testing against that guess, actual consumer expectations become the source of truth the producer must satisfy.
- This is an increasingly relevant topic as systems become more distributed — understanding contract testing (and how it differs from schema validation and full E2E integration testing) is what separates candidates with modern, large-system testing experience from those who've only tested monolithic applications.

## How It Works

**The consumer-driven contract testing flow (Pact-style):**

1. **Consumer test** — the consuming application (e.g., a frontend) writes a test defining exactly what it expects from a specific API interaction (request shape, expected response shape). Running this test generates a **contract file** (a JSON document describing the interaction).
2. **Contract sharing** — the contract file is published to a shared broker (e.g., Pact Broker) accessible to both teams.
3. **Provider verification** — the API team's CI pipeline pulls the contract and replays the consumer's expected requests against the *real* API, verifying the actual responses match what the consumer expects.
4. **Can-I-Deploy check** — before deploying either side, a check confirms the contract is still satisfied by both the current consumer and current provider versions — preventing a breaking deploy on either side.

This differs fundamentally from schema validation (checking one response's structure in isolation) and from full E2E tests (running both systems together, which is slow and requires complex environment coordination).

## Example

**Consumer side (Python, using the `pact-python` library) — defining the expected contract:**
```python
from pact import Consumer, Provider

pact = Consumer("OrderFrontend").has_pact_with(Provider("OrderAPI"))

def test_get_order_contract():
    expected = {
        "id": 501,
        "customer_email": "test@example.com",
        "status": "pending",
        "total": 999.0,
    }

    (pact
        .given("order 501 exists")
        .upon_receiving("a request for order 501")
        .with_request("GET", "/api/orders/501")
        .will_respond_with(200, body=expected))

    with pact:
        response = requests.get(f"{pact.uri}/api/orders/501")
        assert response.json()["status"] == "pending"

    # Running this test generates a contract file, published to a broker
    # for the OrderAPI team to verify against
```

**Provider side (the API team's CI) — verifying the API actually satisfies the contract:**
```python
from pact import Verifier

def test_verify_order_api_against_consumer_contracts():
    verifier = Verifier(provider="OrderAPI", provider_base_url="http://localhost:8000")

    # Pulls the published contract(s) from the broker and replays the
    # consumer's expected requests against the REAL, running API
    success, logs = verifier.verify_with_broker(
        broker_url="https://pact-broker.example.com",
        provider_states_setup_url="http://localhost:8000/_pact/provider_states",
    )
    assert success == 0, f"Contract verification failed: {logs}"
```

## Production Considerations

- Contract testing is most valuable at genuine service boundaries (frontend↔backend, service↔service) — it's not a replacement for testing business logic within a single service, and it's overkill for a small monolith with no separate consumers.
- The "can-I-deploy" check integrated into CI/CD is what actually delivers contract testing's biggest value — catching a breaking change *before* deployment, not after — without it, contract tests still provide useful documentation but lose their strongest safety-net benefit.
- Contract tests require genuine cross-team coordination (a shared broker, agreed provider states) — introducing them successfully is as much an organizational/process change as a technical one, and is usually harder to adopt than schema validation for that reason.

## Common Pitfalls

- Confusing contract testing with schema validation — schema validation checks one response's shape; contract testing verifies an actual agreement between two independently deployed, evolving systems, which is a fundamentally different and broader guarantee.
- Treating contract tests as a full replacement for E2E/integration tests — contracts verify the *interface* agreement, not full business-flow correctness across the whole system; some genuine end-to-end coverage is still valuable for critical flows.
- Letting contracts go stale — if consumer expectations change but the contract isn't regenerated and re-verified, the safety net silently stops working while still appearing to exist.
- Adopting contract testing without the "can-I-deploy" CI integration — without it, contract tests still document expectations but don't prevent an actual breaking deploy, losing much of the practical value.

## Interview Notes

- Be ready to explain what makes contract testing "consumer-driven" specifically, and why that direction (consumer defines, provider verifies) is deliberate rather than arbitrary.
- Understand the distinction between contract testing, schema validation, and full E2E integration testing — a common, specific interview question, especially for roles touching microservices architectures.
- Be able to describe a scenario where contract testing would have caught a break that isolated unit/schema tests on either side wouldn't have — this shows genuine understanding of the gap contract testing fills.

## References

- [Pact — Contract Testing](https://docs.pact.io/)
- [Martin Fowler — Contract Testing](https://martinfowler.com/bliki/ContractTest.html)