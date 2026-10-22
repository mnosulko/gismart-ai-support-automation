# Gismart AI Support Automation

## 1. Project Overview

Gismart AI Support Automation is an n8n workflow designed to simulate the processing of customer support tickets for a subscription-based fitness application called PulseFit.

The workflow validates incoming ticket data, classifies valid tickets by intent, routes them to the appropriate processing branch, generates customer-facing responses using AI, validates the generated responses, and records structured processing logs.

The solution covers five main support scenarios:

- Subscription cancellation
- Subscription cancellation with billing clarification
- Refund requests
- Technical issues
- General support requests

It also handles invalid tickets through a separate validation-error branch.

**Important:** All subscription and billing operations in this assessment are simulations. The workflow does not perform real subscription cancellations or issue actual refunds.

## 2. Repository Contents

The repository contains the following files:

| File | Description |
|---|---|
| `Gismart AI Support Automation.json` | Exported n8n workflow, including nodes, connections, and workflow configuration. |
| `sample-input-data.json` | Sample customer support tickets used for testing the workflow. |
| `Знімок екрана 2026-10-22 о 21.26.20.png` | Screenshot documenting the workflow. |


## 3. How to Run the Workflow

### Prerequisites

- An n8n instance.
- Access to the exported workflow JSON file.
- The appropriate credentials for the AI model used by the `AI Response Generator` node.

### Setup and execution

1. Download or clone this repository.
2. Open n8n.
3. Import `Gismart AI Support Automation.json` into your n8n instance.
4. Configure the required AI model credentials in n8n if they are not already available.
5. Open the imported workflow and review the `Read JSON` node.
6. The current workflow contains sample ticket data directly in this node. The separate `sample-input-data.json` file is provided for reference and reproducible testing; the current workflow does not automatically read that separate file.
7. Click **Execute Workflow** to run the workflow manually.
8. Inspect the output of `Log Results` for valid tickets.
9. Inspect `Log Validation Error` for tickets that fail validation.

The expected test dataset contains six tickets. Five valid tickets should reach `Log Results`, while invalid ticket `10005` should be routed to `Log Validation Error`.

The workflow can be tested manually without publishing it as an active production workflow.

## 4. Workflow Architecture

The workflow is organized into validation, classification, routing, processing, response generation, and logging stages.

### 4.1. Input and validation

**Nodes:** `Read JSON`, `Validate and Normalize`, `If`

The workflow begins with sample support tickets.

`Validate and Normalize` checks the incoming data for required fields and normalizes ticket information before further processing. Invalid records are separated from valid ones.

The `If` node routes valid tickets toward classification and invalid tickets toward the validation-error logging branch.

This prevents tickets with missing or invalid required information from being processed as normal support requests.

### 4.2. Intent classification

**Node:** `Mock AI Classifier`

The classifier assigns an intent to each valid ticket. In the current implementation, classification uses mock, deterministic logic rather than a production-trained AI classifier.

The supported intents are:

- `cancellation`
- `cancellation_with_billing_clarification`
- `refund`
- `technical_issue`
- `other`

Using a mock classifier makes the workflow predictable and easier to test with a fixed sample dataset.

### 4.3. Routing and business logic

**Node:** `Switch`

The `Switch` node routes each ticket to the corresponding processing branch based on its detected intent.

The workflow contains dedicated handlers:

- `Process Cancellation`
- `Process Cancellation with Billing Clarification`
- `Process Refund`
- `Process Technical Issue`
- `Process Other`

Each handler sets the appropriate action type, action status, execution status, and supporting context.

For example, refund requests are marked as requiring manual review rather than being treated as completed refunds. Technical issues are queued for support, while general questions are routed for manual review.

Subscription matching is handled using deterministic logic. The workflow records the number of subscriptions matching the relevant product where that information is calculated.

### 4.4. Merging processing branches

**Nodes:** `Merge`, `Merge1`, `Merge2`, `Merge3`

The Merge nodes combine the outputs of the separate processing branches.

This design allows different ticket types to retain their individual business logic while sharing the same downstream response-generation, validation, and logging stages.

### 4.5. AI response generation

**Node:** `AI Response Generator`

The AI model generates a natural-language response based on the ticket details and the context provided by the workflow.

AI is used to make responses more readable and customer-friendly. It does not replace the deterministic validation, routing, or business-action logic.

Because the business actions in this exercise are simulated, generated responses must not be interpreted as evidence that a real cancellation or refund has occurred.

### 4.6. Response validation

**Node:** `Validate AI Response`

The workflow checks the generated response and extracts the response text into structured fields.

The output includes fields such as:

- `response_text`
- `response_valid`
- `response_status`

This provides a basic check that a usable response was generated. It does not guarantee that every statement in the response is factually correct or consistent with every business rule.

### 4.7. Logging

**Nodes:** `Log Results`, `Log Validation Error`

Valid tickets are recorded in `Log Results`. Invalid tickets are handled separately by `Log Validation Error`.

The processing logs include fields such as:

- `timestamp`
- `ticket_id`
- `detected_intent`
- `matching_subscription_count`
- `action_type`
- `action_status`
- `execution_status`
- `error_message`

The valid-ticket log also includes customer and ticket context, the generated response, and response-validation results.

Keeping action status separate from execution status is important. For example, a workflow step can execute successfully even if the corresponding action requires manual review or remains queued.

The current implementation returns structured log records as workflow output. Persistent storage and production monitoring would be added in a production deployment.

## 5. AI vs. Deterministic Logic

The solution separates predictable business rules from natural-language generation.

| Component | Approach | Purpose |
|---|---|---|
| Input validation | Deterministic | Detect missing or invalid required fields. |
| Data normalization | Deterministic | Prepare ticket fields for consistent processing. |
| Intent classification | Mock/deterministic | Assign a support intent using predefined rules. |
| Intent routing | Deterministic | Send each ticket to the correct processing branch. |
| Subscription matching | Deterministic | Count matching subscriptions where applicable. |
| Action status assignment | Deterministic | Represent simulated outcomes, queued work, or manual review. |
| Customer response generation | AI | Generate readable, context-aware responses. |
| Response validation | Deterministic | Check and extract the generated response. |
| Logging | Deterministic | Produce structured records for inspection and troubleshooting. |

This separation makes the workflow easier to understand, test, and maintain. Business-critical decisions should remain controlled by explicit rules and trusted backend services rather than relying only on generated text.

## 6. Error Handling

The workflow uses a separate path for invalid tickets.

If a ticket fails validation, it is routed to `Log Validation Error` rather than being sent through normal intent processing.

For example, ticket `10005` contains a missing or invalid message. The expected validation error is:

`Missing or invalid message`

The invalid record is marked as failed and includes an error message explaining why processing was stopped.

For valid tickets, action and execution statuses are recorded separately. This helps distinguish a technical processing failure from a business action that was not performed or requires additional review.

### Logging approach

Structured logs make it possible to inspect:

- Which ticket was processed.
- Which intent was detected.
- Which processing branch was used.
- Whether an action was simulated, queued, or sent for review.
- Whether an execution error occurred.
- Whether a response was generated and passed basic validation.

In production, these logs should be persisted to a durable storage system and connected to monitoring and alerting.

## 7. Known Limitations and Future Improvements

The current implementation is an assessment prototype rather than a production-ready billing automation system.

### Current limitations

1. **Mock classification:** The classifier relies on predefined rules and may not handle ambiguous, unusual, or differently worded requests correctly.
2. **Basic response validation:** A response can be non-empty and still contain inaccurate or unsupported claims.
3. **Simulated business actions:** Cancellation and refund processing do not interact with real billing systems.
4. **Limited persistence:** Logs are produced as workflow output rather than stored in a dedicated database or logging platform.
5. **Embedded test data:** Sample tickets are currently defined inside the `Read JSON` node rather than loaded dynamically from `sample-input-data.json`.
6. **Limited operational safeguards:** Production-level retries, monitoring, authorization, and recovery mechanisms would need to be added.

### Improvements I would make next

- Replace the mock classifier with a tested classifier or an AI-assisted classifier that returns structured output and confidence scores.
- Add automated tests for each intent, validation failure, and action status.
- Strengthen response validation to detect unsupported claims and ensure that customer-facing text matches the actual action status.
- Integrate a trusted backend API for subscription and billing operations.
- Add durable logging, monitoring, alerting, retries, and timeout handling.
- Introduce human approval for sensitive or ambiguous billing actions.
- Load input data through a dedicated input mechanism rather than embedding it in a Code node.

## 8. Handling Low-Confidence Classifier Results in Production

The current mock classifier is not a calibrated production classifier and does not implement a reliable confidence threshold. A production implementation should explicitly address low-confidence results.

I would introduce the following process:

1. **Define a confidence threshold.** Calibrate the threshold using a reviewed dataset of real support messages rather than selecting it arbitrarily.
2. **Route uncertain cases to human review.** If confidence falls below the threshold, or if the message is ambiguous, the workflow should avoid triggering account-changing actions.
3. **Ask clarifying questions when appropriate.** If a ticket lacks enough information to determine the customer's intent, request clarification instead of guessing.
4. **Record the decision.** Log the predicted intent, confidence score, routing decision, and review outcome.
5. **Evaluate performance continuously.** Review classification errors and update the classifier using approved, representative examples.

For refund and cancellation requests, low confidence should result in a safe fallback rather than an automatic billing action.

## 9. Integration Question: Zendesk and Secure API Credentials

In production, I would connect Zendesk to n8n using a Zendesk trigger or an authenticated webhook.

### Ticket ingestion

The trigger would receive a new or updated Zendesk ticket. The workflow would map relevant ticket fields into a normalized schema, including:

- Zendesk ticket ID
- Requester email
- Subject
- Message
- Product or subscription reference, where available

The normalized ticket would then pass through validation, classification, and routing.

### Returning results to Zendesk

After processing, the workflow could use Zendesk API operations to add a public reply or internal note, apply relevant tags, and update the ticket status.

Public replies and internal notes should be handled separately so that internal processing details are not accidentally exposed to customers.

### Billing API integration

Subscription cancellation and refund requests would be sent to an authenticated backend billing API using n8n HTTP Request nodes.

The backend should independently verify authorization, subscription ownership, refund eligibility, and applicable business policies. The workflow should rely on the backend's confirmed result before telling a customer that an action has completed.

### Secure credential management

I would store API credentials in n8n's credential store or a managed secrets vault rather than hard-coding them in workflow code, prompts, or JSON files.

Additional safeguards would include:

- Least-privilege service accounts.
- Separate development and production credentials.
- Credential rotation and revocation procedures.
- Authenticated and validated webhooks.
- Restricted access to workflow editing and credential management.
- Redaction of tokens and sensitive customer or payment data from logs.
- Auditable records of billing actions and their confirmed outcomes.

This approach keeps integration credentials separate from workflow logic and reduces the risk of unauthorized access.

## 10. Production Question: Three Most Important Improvements Before Deployment

If this system had to process 10,000 customer tickets per month and could trigger real subscription cancellations or refunds, I would prioritize the following three improvements.

### 1. Billing safety, authorization, and idempotency

Before enabling real billing operations, I would integrate a trusted backend API that verifies eligibility and authorization for every action.

The system should use idempotency keys to prevent duplicate cancellations or refunds when requests are retried. High-risk or ambiguous cases should require human approval. Customer-facing responses should be based on confirmed backend results, not on the fact that a workflow node executed successfully.

**Why:** Incorrect or duplicated billing actions can directly affect customers and create financial, operational, and reputational risks.

### 2. Reliability and scalability

I would introduce queue-based processing where appropriate, bounded retries with backoff, timeouts, rate limits, and a dead-letter or recovery process for tickets that repeatedly fail.

The workflow should be load-tested, and its behavior under concurrent execution and external API failures should be evaluated. Processing should also be idempotent so that a retry does not unintentionally repeat a business action.

**Why:** A system handling thousands of tickets each month needs predictable processing, controlled failure recovery, and resilience to external service outages or rate limits.

### 3. Observability, evaluation, and human fallback

I would add persistent structured logs, audit trails, dashboards, and alerts for processing failures, unusual billing activity, and response-quality problems.

I would also evaluate classification accuracy and AI response quality against reviewed examples, version prompts and business policies, and route low-confidence or sensitive cases to human review.

**Why:** Monitoring and evaluation make failures visible, support investigations, and help maintain reliable customer support as traffic and system complexity grow.

## 11. Conclusion

This project demonstrates a modular n8n workflow for validating, classifying, routing, responding to, and logging customer support tickets.

Its main design principle is to keep deterministic processing and simulated business actions separate from AI-generated customer communication. The prototype provides a foundation for further development, but real billing operations would require stronger authorization, backend verification, reliable logging, and production safeguards before deployment.
