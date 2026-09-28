## Vehicle Analytics Cloud Assessment – Justification

Use this file to briefly explain your design decisions. Bullet points are fine.

### 1. High-level architecture

- Summary of your overall cloud design:
  - Simplified the previous microservice-heavy design to cut cost, complexity and the number of deployable units.
  - Users sign in through Cognito (JWT) and use one Amplify React UI. The legacy ECS UI was removed.
  - One public ALB with path rules replaces the two API Gateways, VPC Link and Cloud Maps: ```/v1/*``` goes to va-api, and ```/ws``` and ```/v1/logging/*``` go to telemetry-gateway.
  - telemetry-gateway merges CanDecoder, Streaming and Log Generator, with an in-process latest-value store replacing Redis.
  - va-api is one Fargate service replacing the garage, log and tyre services and Lambdas.
  - sim-worker runs on Fargate Spot, scaling 0 to N on SQS queue depth. va-api queues the jobs.
  - Storage is DynamoDB (garage, track day, weather readings) and S3 (tyre datasets, logs, weather archive).
  - One raw WebSocket protocol: NanoPB in from the car, MsgPack out to dashboards.

### 2. Weather station integration

- How the weather station connects to the cloud:
  - The station connects to AWS IoT Core over MQTT/TLS with its own X.509 certificate.
  - It publishes JSON to ```weather/<stationId>/data```. The IoT policy only allows publishing to its own topic, and each device can be revoked individually.
  - There is no public write API, which keeps the attack surface small.
  - MQTT suits poor networks: small messages, one long-lived connection, QoS 1 retries and Last Will for offline detection.
  - The station buffers readings locally during long outages.
- How weather data flows into the Vehicle Analytics platform:
  - IoT Rule 1 writes each reading to DynamoDB ```weather_readings``` (PK ```stationId```, SK ```ts```). The station's own timestamp from the payload is used as ts, so buffered readings sort correctly.
  - IoT Rule 2 sends the raw message to Firehose, which batches and gzips it into an S3 archive that Athena can query later.
  - A weather adapter inside telemetry-gateway polls the latest reading about every 5 s and injects it as extra channels from a reserved ID range (e.g. ```Weather.AirTemp```).
  - Dashboards show weather like any CAN signal, with no frontend protocol change. The Log Generator writes weather columns into the same log CSV.
  - va-api exposes ```/v1/weather/latest```, ```/v1/weather/history``` and ```/v1/logs/{id}/weather``` to read current and historical conditions.
    
### 3. Infrastructure as Code (IaC)

- Which IaC tool(s) you would use (e.g. Terraform, AWS CDK) and why:
  - Terraform, with AWS CDK as an equally valid alternative.
  - It makes the whole stack reproducible and reviewable in pull requests, and avoids manual console setup. That covers IoT Things, certificates, policies and rules, Firehose, DynamoDB, S3, ECS, ALB, SQS/DLQ and IAM roles.
- How IaC fits into deployment / environments:
   - GitHub Actions assumes an AWS role through OIDC, so no long-lived keys are stored.
   - Merging to main builds an image to ECR and updates ECS. Amplify builds a preview for each pull request.
   - One production stack plus ephemeral dev stacks per branch, destroyed afterwards. A permanent staging copy would add a second ALB and extra tasks for little benefit.
   - docker compose runs va-api, telemetry-gateway, DynamoDB Local and a mock CAN/weather publisher locally.

### 4. Security, reliability, observability

- Key security choices (network boundaries, authn/z, secrets, etc.):
  - Network: the ALB is the only public entry point. The task security group accepts traffic only from the ALB security group. S3 and DynamoDB use gateway endpoints. Public subnets avoid NAT cost, and private subnets with NAT are the documented upgrade path.
  - Authn/z: Cognito JWT for dashboard users, a per-vehicle credential for the car, and per-station IoT certificates with topic-scoped policies.
  - Access and secrets: IAM Identity Center groups with least privilege and no shared admin password. OIDC for CI. Any remaining secrets go in Secrets Manager or SSM Parameter Store.
- Reliability and failure modes:
  - telemetry-gateway is a single task and a single point of failure. It never runs on Spot, and there is a deploy freeze on test and competition days.
  - CanForwarder must auto-reconnect, since deploys drop WebSocket connections.
  - The competition-day option is a second gateway task with Valkey behind the state interface.
  - va-api runs min 1, max 2 (it cannot scale to zero behind an ALB).
  - sim-worker uses SQS retries, and the DLQ catches failing jobs.
  - If a weather station goes offline, the last value ages out and is flagged as stale (older than about 3 sampling periods).
  - Safety-critical alerts should not depend only on the cloud link.
- Monitoring/alerting and operational concerns:
  - The IoT Rule error action logs failed payloads to CloudWatch, so bad data is never lost silently.
  - Suggested alarms: ALB 5xx, running task count, DLQ depth, and no message from a station within N minutes.
  - AWS Budgets and anomaly alerts go to a shared channel.
  - Set log retention explicitly, and right-size CPU and memory from CloudWatch data after the first test day.

### 5. Cost and scalability considerations

- Expected cost drivers and how you would keep costs under control:
  - Fixed costs: one ALB, two small always-on tasks, public IPv4 addresses, Amplify build minutes beyond the free tier, and CloudWatch logs.
  - Variable costs: IoT messages, DynamoDB requests, S3 and Spot. These are small at this scale.
  - Controls:
    - Sample weather every 5 to 10 s. One message per second is about 2.6M IoT messages a month
    - Firehose batching to avoid tiny S3 objects
    - S3 lifecycle rules
    - Spot for batch work only
    - Ephemeral dev stacks
    - Budgets and alerts
    - Verify numbers in the AWS Pricing Calculator with real test-day hours
- How the design scales to more stations/events:
  - More stations: IoT Core scales per Thing, and the ```weather/+/data``` rules cover every station. DynamoDB is keyed by ```stationId```, and Firehose partitions by date. Fleet Provisioning only becomes worthwhile at large numbers.
  - More events or load: schedule va-api and telemetry-gateway up on race days, and sim-worker scales 0 to N on queue depth.
  - Concurrent vehicles: bring Valkey back behind the state interface and run multiple gateway tasks.
