# Testing with Cloud Storage & Managed Services

## What It Is

This note covers testing against cloud-managed services — object storage (S3, Cloud Storage), managed databases (RDS, Cloud SQL), and message queues (SQS, Pub/Sub) — where the service itself is fully managed by the cloud provider, which changes how you test setup/teardown, resilience, and realistic data flows compared to a self-hosted equivalent (see [Docker Compose for Test Stacks](../09-docker/docker-compose-for-test-stacks.md) for the self-hosted comparison).

## Why It Matters

- Managed services can't always be freely reset/restarted the way a local Docker container can (see [Docker Fundamentals for QA](../09-docker/docker-fundamentals-for-qa.md)) — test data isolation and cleanup strategy needs rethinking for a shared, managed, harder-to-fully-reset resource.
- These services have their own distinct failure modes (S3 eventual consistency edge cases, RDS connection limits, queue message visibility timeouts) that are genuinely different from a self-hosted equivalent's failure modes, and worth testing for specifically rather than assuming identical behavior.
- Using local emulation tools (LocalStack, Testcontainers) for these managed services during automated testing is a genuinely practical, cost-effective, and fast approach worth knowing — extending the containerization principles from [09-docker](../09-docker/) to simulate managed cloud services locally.

## How It Works

**Object storage (S3-style) testing considerations:**
- Testing upload/download correctness, including realistic file sizes/types.
- Testing access control (bucket policies, object ACLs) — verifying unauthorized access is correctly rejected, extending [API Security Testing Basics](../04-api-testing/api-security-testing-basics.md)'s authorization testing principles to storage-level permissions.
- Testing cleanup — object storage buckets used for testing need explicit teardown/lifecycle policies, since orphaned test objects accumulate cost over time (connecting to [Test Data & Volumes in Docker](../09-docker/test-data-and-volumes-in-docker.md)'s cleanup discipline).

**Managed database testing considerations:**
- Connection pool behavior under test load — a managed database often has connection limits that differ from a locally-run one, worth verifying test suites don't exhaust them (see [Identifying Performance Bottlenecks](../14-performance-testing/identifying-performance-bottlenecks.md) on connection pool exhaustion).
- Testing against a genuinely isolated test database instance/schema, not a shared one, to avoid the contention issues covered throughout [05-sql-database-testing](../05-sql-database-testing/).

**Message queue testing considerations:**
- Testing message visibility timeout behavior — what happens if a consumer doesn't process a message within the configured window (does it get redelivered correctly, does this cause duplicate processing that needs idempotency handling — see [Idempotency & Rate Limiting](../04-api-testing/idempotency-and-rate-limiting.md)).
- Testing dead-letter queue behavior for messages that repeatedly fail processing.

**Local emulation — LocalStack (AWS services) and Testcontainers (a broader containerized-dependency testing library):** both let tests run against local, containerized emulations of managed cloud services, avoiding real cloud costs and network latency during routine test execution, while still exercising realistic API behavior.

## Example

**Using LocalStack to test S3 upload/download logic locally, without any real AWS cost:**
```python
import boto3
import pytest

@pytest.fixture
def s3_client():
    # Points at LOCALSTACK's local S3 emulation, not real AWS
    return boto3.client(
        's3',
        endpoint_url='http://localhost:4566',   # LocalStack's default endpoint
        aws_access_key_id='test',
        aws_secret_access_key='test',
        region_name='us-east-1',
    )

def test_document_upload_and_retrieval(s3_client):
    bucket_name = 'test-documents-bucket'
    s3_client.create_bucket(Bucket=bucket_name)

    s3_client.put_object(
        Bucket=bucket_name,
        Key='test-visa-application.pdf',
        Body=b'fake pdf content for testing',
    )

    response = s3_client.get_object(Bucket=bucket_name, Key='test-visa-application.pdf')
    assert response['Body'].read() == b'fake pdf content for testing'

def test_unauthorized_access_to_private_object_rejected(s3_client):
    bucket_name = 'test-private-bucket'
    s3_client.create_bucket(Bucket=bucket_name)
    s3_client.put_object(Bucket=bucket_name, Key='private-doc.pdf', Body=b'sensitive', ACL='private')

    # A different, unauthorized client attempting access
    unauthorized_client = boto3.client(
        's3', endpoint_url='http://localhost:4566',
        aws_access_key_id='different-unauthorized-key',
        aws_secret_access_key='different-secret',
    )
    with pytest.raises(Exception):   # should be rejected, not silently succeed
        unauthorized_client.get_object(Bucket=bucket_name, Key='private-doc.pdf')
```

**Testing message queue redelivery/visibility timeout behavior, verifying idempotent handling of a redelivered message:**
```python
def test_queue_message_redelivered_after_visibility_timeout(sqs_client):
    queue_url = sqs_client.create_queue(QueueName='test-orders-queue')['QueueUrl']
    sqs_client.send_message(QueueUrl=queue_url, MessageBody='{"order_id": 501}')

    # Receive but DON'T delete — simulating a consumer that failed
    # to process/acknowledge in time
    messages = sqs_client.receive_message(QueueUrl=queue_url, VisibilityTimeout=1)
    time.sleep(2)   # wait past the visibility timeout

    # The message should be redelivered since it was never deleted
    redelivered = sqs_client.receive_message(QueueUrl=queue_url)
    assert 'Messages' in redelivered

    # A well-designed consumer should handle this redelivery IDEMPOTENTLY —
    # processing the same order_id twice should not create a duplicate order
```

## Production Considerations

- Use local emulation (LocalStack, Testcontainers) for the bulk of routine automated test execution — this avoids real cloud costs and network latency for tests run many times a day, reserving real cloud infrastructure testing for periodic validation that emulation behaves faithfully (see [Testing Serverless Applications](./testing-serverless-applications.md)'s similar emulation-vs-real trade-off).
- Always test consumer idempotency for queue-based processing explicitly (as in the redelivery example) — message redelivery is a normal, expected occurrence in managed queue systems, not an edge case, making idempotent handling a functional requirement, not just a nice-to-have.
- Apply explicit lifecycle/cleanup policies to test-used cloud storage buckets (auto-delete objects after N days) — this is the storage-specific version of the cleanup discipline from [Test Data & Volumes in Docker](../09-docker/test-data-and-volumes-in-docker.md), preventing unbounded cost accumulation from orphaned test objects.

## Common Pitfalls

- Testing only the happy path of queue message processing, never testing redelivery/duplicate-message handling — a real, common occurrence in managed queue systems that needs explicit idempotency testing, not just assumed correctness.
- Running all tests against real cloud infrastructure by default, incurring unnecessary cost and latency for routine test runs that local emulation could handle just as effectively.
- Not testing storage access-control boundaries (bucket policies, object ACLs), missing the storage-layer equivalent of the authorization bugs covered in [API Security Testing Basics](../04-api-testing/api-security-testing-basics.md).
- Letting test-created cloud storage objects accumulate indefinitely without lifecycle policies, resulting in unexpected, creeping storage costs over time.

## Interview Notes

- Be ready to describe how managed service testing differs from self-hosted equivalent testing — particularly around setup/teardown constraints and distinct failure modes (queue redelivery, storage eventual consistency).
- Understand why queue-based consumers need to be idempotent, and be able to describe a test verifying idempotent handling of a redelivered message.
- Be able to describe local emulation tools (LocalStack, Testcontainers) and the cost/speed trade-off they offer versus testing against real cloud infrastructure directly.

## References

- [LocalStack — Documentation](https://docs.localstack.cloud/)
- [Testcontainers — Documentation](https://testcontainers.com/)
- [AWS — Amazon SQS Visibility Timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)