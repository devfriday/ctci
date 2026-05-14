Kiro’s spec flow commonly works well with structured requirements, design, and tasks documents; Kiro also emphasizes EARS-style acceptance criteria, so I shaped this for that workflow. �
Kiro +1
Copy this into Kiro as a single spec prompt, or split it into:
Plain text
.kiro/specs/gitlab-config-manager/requirements.md
.kiro/specs/gitlab-config-manager/design.md
.kiro/specs/gitlab-config-manager/tasks.md
requirements.md
Markdown
# GitLab-Based Spring Boot Configuration Manager

## Overview

Build a web application that allows authorized users to view, edit, validate, and submit environment-specific Spring Boot configuration files stored in GitLab.

The application must not deploy configuration directly to Kubernetes and must not use Spring Cloud Config Server. Deployment is handled by an existing DevOps deployment API owned by the DevOps team.

The system creates GitLab merge requests for configuration changes. Once a merge request is approved and merged into the target branch, a GitLab webhook notifies this application. The application verifies the merge event and calls the DevOps deployment API. The DevOps deployment API is responsible for checking out the environment-specific configuration and copying it to the appropriate configuration folder used by the Spring Boot microservice in Kubernetes.

## Goals

- Provide a secure UI for managing Spring Boot configuration files stored in GitLab.
- Support application and environment selection.
- Read current configuration from GitLab.
- Allow users to edit YAML/properties files.
- Validate configuration before submission.
- Show a diff preview before creating a merge request.
- Create a GitLab branch, commit changes, and open a merge request.
- Track merge request status.
- Receive GitLab webhook events.
- Trigger DevOps deployment API only after the merge request is merged.
- Maintain audit history for all configuration changes.
- Prevent direct production changes without GitLab MR approval.

## Non-Goals

- Do not implement Spring Cloud Config Server.
- Do not directly copy files to Kubernetes.
- Do not directly mutate running Spring Boot service configuration.
- Do not bypass GitLab merge request review.
- Do not store GitLab or DevOps secrets in the frontend.
- Do not store raw configuration secrets in the application database.
- Do not expose GitLab write tokens to the browser.
- Do not trigger deployment on MR approval alone.
- Do not deploy unmerged branches.

## Users

### Configuration Editor

A user who can view and submit configuration changes for applications/environments they are authorized to modify.

### Reviewer

A GitLab reviewer who approves and merges merge requests in GitLab.

### Admin

A user who can manage application metadata, environment metadata, repository mappings, and access rules.

### DevOps System

An external API responsible for deploying merged configuration to Kubernetes.

## Functional Requirements

### Application and Environment Browsing

#### Requirement 1

As a configuration editor, I want to select an application and environment so that I can view the correct configuration files.

Acceptance Criteria:

- WHEN a user opens the application list, THE SYSTEM SHALL display only applications the user is authorized to view.
- WHEN a user selects an application, THE SYSTEM SHALL display available environments for that application.
- WHEN an environment is protected, THE SYSTEM SHALL mark the environment as protected in the UI.
- WHEN a user is not authorized for an application or environment, THE SYSTEM SHALL deny access.

### GitLab Configuration Reading

#### Requirement 2

As a configuration editor, I want to read existing configuration files from GitLab so that I can edit the current source-of-truth configuration.

Acceptance Criteria:

- WHEN a user selects an application and environment, THE SYSTEM SHALL read configured file paths from GitLab.
- WHEN GitLab returns file content, THE SYSTEM SHALL decode and display it to the user.
- WHEN a file does not exist, THE SYSTEM SHALL show a clear not-found message.
- WHEN GitLab is unavailable, THE SYSTEM SHALL show a recoverable error and not create a change request.
- WHEN reading a file, THE SYSTEM SHALL capture the GitLab commit SHA or last commit ID for traceability.

### Configuration Editing

#### Requirement 3

As a configuration editor, I want to edit YAML or properties files safely.

Acceptance Criteria:

- WHEN a user edits YAML, THE SYSTEM SHALL preserve valid YAML formatting where possible.
- WHEN a user edits properties files, THE SYSTEM SHALL preserve key-value format.
- WHEN a user attempts to submit invalid syntax, THE SYSTEM SHALL block submission.
- WHEN a user edits protected values, THE SYSTEM SHALL apply masking rules for secrets.
- WHEN a user changes a file, THE SYSTEM SHALL show the modified state before submission.

### Validation

#### Requirement 4

As a platform owner, I want configuration changes validated before a merge request is created.

Acceptance Criteria:

- WHEN a user submits a change, THE SYSTEM SHALL validate YAML/properties syntax.
- WHEN duplicate YAML keys are present, THE SYSTEM SHALL reject the change.
- WHEN required configuration keys are missing, THE SYSTEM SHALL reject the change.
- WHEN a value has the wrong type, THE SYSTEM SHALL reject the change.
- WHEN a plain-text secret pattern is detected, THE SYSTEM SHALL reject or warn based on policy.
- WHEN the environment is production, THE SYSTEM SHALL apply stricter validation rules.
- WHEN validation fails, THE SYSTEM SHALL show actionable error messages.

### Diff Preview

#### Requirement 5

As a configuration editor, I want to preview differences before creating a merge request.

Acceptance Criteria:

- WHEN a user modifies a file, THE SYSTEM SHALL show a side-by-side or unified diff.
- WHEN multiple files are changed, THE SYSTEM SHALL show a diff per file.
- WHEN secrets are detected, THE SYSTEM SHALL mask sensitive values in the diff display.
- WHEN the user has not reviewed the diff, THE SYSTEM SHALL prevent accidental submission.

### Change Request Creation

#### Requirement 6

As a configuration editor, I want to create a GitLab merge request for configuration changes.

Acceptance Criteria:

- WHEN validation succeeds, THE SYSTEM SHALL create a dedicated Git branch.
- WHEN creating the branch, THE SYSTEM SHALL start from the configured target branch.
- WHEN committing changes, THE SYSTEM SHALL include all file changes in one commit where possible.
- WHEN a commit succeeds, THE SYSTEM SHALL create a GitLab merge request.
- WHEN the merge request is created, THE SYSTEM SHALL store MR metadata locally.
- WHEN the merge request is created, THE SYSTEM SHALL display the GitLab MR URL.
- WHEN branch creation, commit, or MR creation fails, THE SYSTEM SHALL mark the request as failed and show the reason.

### GitLab Merge Request Tracking

#### Requirement 7

As a configuration editor, I want to see the status of my change request.

Acceptance Criteria:

- WHEN a change request is created, THE SYSTEM SHALL show the MR status.
- WHEN the MR is open, THE SYSTEM SHALL show the request as waiting for approval/merge.
- WHEN the MR is closed without merge, THE SYSTEM SHALL mark the request as cancelled or closed.
- WHEN the MR is merged, THE SYSTEM SHALL mark the request as merged.
- WHEN webhook delivery is missed, THE SYSTEM SHALL support manual status reconciliation.

### GitLab Webhook Handling

#### Requirement 8

As the system, I want to receive GitLab merge request webhook events so that deployment can be triggered only after merge.

Acceptance Criteria:

- WHEN a webhook event is received, THE SYSTEM SHALL verify the webhook signature or token.
- WHEN the event is not a merge request event, THE SYSTEM SHALL ignore it safely.
- WHEN the event is an approval event, THE SYSTEM SHALL not trigger deployment.
- WHEN the event is a merge request merge event, THE SYSTEM SHALL verify the MR IID, project ID, source branch, target branch, and merge commit SHA.
- WHEN the event matches a known change request and is merged, THE SYSTEM SHALL enqueue deployment.
- WHEN duplicate webhook events are received, THE SYSTEM SHALL not trigger duplicate deployments.

### DevOps Deployment API Integration

#### Requirement 9

As the system, I want to call the DevOps deployment API after a merge request is merged.

Acceptance Criteria:

- WHEN a valid merge event is processed, THE SYSTEM SHALL call the DevOps deployment API.
- WHEN calling the DevOps deployment API, THE SYSTEM SHALL include application, environment, config path, target branch, MR IID, merge commit SHA, and change request ID.
- WHEN calling the DevOps deployment API, THE SYSTEM SHALL use an idempotency key.
- WHEN the DevOps API accepts the deployment request, THE SYSTEM SHALL mark the change as deployment requested or deploying.
- WHEN the DevOps API returns a failure, THE SYSTEM SHALL mark the deployment as failed and allow authorized retry.
- WHEN the same merge event is processed again, THE SYSTEM SHALL not submit a duplicate deployment request.

### Audit Logging

#### Requirement 10

As an auditor, I want all configuration activity tracked.

Acceptance Criteria:

- WHEN a user views, edits, validates, submits, or cancels a change, THE SYSTEM SHALL record an audit event.
- WHEN a GitLab MR is created, THE SYSTEM SHALL record the MR URL and IID.
- WHEN a webhook is received, THE SYSTEM SHALL record the event type and processing result.
- WHEN deployment is requested, THE SYSTEM SHALL record the request metadata.
- WHEN sensitive data is present, THE SYSTEM SHALL not store raw secret values in audit logs.
- WHEN audit records are viewed, THE SYSTEM SHALL show actor, action, timestamp, application, environment, and result.

### Security and Authorization

#### Requirement 11

As a security owner, I want strong authorization controls around configuration changes.

Acceptance Criteria:

- WHEN a user logs in, THE SYSTEM SHALL authenticate using SSO/OIDC or the configured enterprise identity provider.
- WHEN a user accesses an application, THE SYSTEM SHALL enforce RBAC.
- WHEN a user accesses a protected environment, THE SYSTEM SHALL enforce protected-environment permissions.
- WHEN the frontend communicates with the backend, THE SYSTEM SHALL not expose GitLab tokens.
- WHEN the backend communicates with GitLab or DevOps API, THE SYSTEM SHALL use server-side credentials.
- WHEN secrets appear in config files, THE SYSTEM SHALL mask them in UI and logs.
- WHEN webhook verification fails, THE SYSTEM SHALL reject the event.

### Admin Configuration

#### Requirement 12

As an admin, I want to manage repository and deployment mappings.

Acceptance Criteria:

- WHEN an admin creates an application record, THE SYSTEM SHALL store the GitLab project ID and base config path.
- WHEN an admin creates an environment record, THE SYSTEM SHALL store target branch, config path mapping, protected flag, and DevOps environment name.
- WHEN mappings are invalid, THE SYSTEM SHALL prevent saving.
- WHEN an application or environment is disabled, THE SYSTEM SHALL hide it from normal users.

## Non-Functional Requirements

### Reliability

- The system shall process GitLab webhooks idempotently.
- The system shall support retrying failed DevOps deployment API calls.
- The system shall preserve change request state across restarts.
- The system shall not lose audit history.

### Security

- All credentials shall be stored server-side using environment variables, Kubernetes secrets, or an approved secret manager.
- GitLab tokens shall have least-privilege access.
- DevOps API tokens shall have least-privilege access.
- Sensitive values shall be masked in UI, logs, diffs, and audit records.
- Production changes shall require GitLab merge request review.

### Observability

- The backend shall expose health checks.
- The backend shall log structured events for GitLab calls, MR creation, webhook handling, and DevOps API calls.
- The backend shall expose metrics for validation failures, MR creation failures, webhook events, deployment requests, and deployment failures.
- Logs shall include correlation IDs and change request IDs.

### Performance

- Reading small and medium configuration files should complete within acceptable UI latency.
- Large config files should be handled without freezing the browser.
- GitLab API calls should use timeouts and retries where appropriate.
- Webhook handling should return quickly by enqueueing deployment work.

### Maintainability

- The system shall separate GitLab client logic, validation logic, webhook logic, and deployment orchestration logic.
- The frontend shall not contain deployment or GitLab write logic.
- The backend shall expose clean REST APIs.
- Business logic shall be covered by unit tests.
- GitLab and DevOps integrations shall be covered by integration tests or mocked contract tests.

## Assumptions

- GitLab is the source of truth for configuration.
- DevOps owns the deployment API and Kubernetes copy process.
- The Spring Boot microservice reads configuration from a file location populated by DevOps deployment.
- Configuration is environment-specific.
- Merge request approval happens inside GitLab.
- Deployment should happen only after merge.
- The application backend is implemented in Spring Boot.
- The frontend may be implemented in React, Angular, or another approved frontend framework.
design.md
Markdown
# Design: GitLab-Based Spring Boot Configuration Manager

## System Summary

This system provides a secure web UI and backend service for managing environment-specific Spring Boot configuration files stored in GitLab.

The system does not deploy configuration directly. It creates GitLab merge requests. After a merge request is approved and merged, GitLab sends a webhook to the backend. The backend verifies the event and calls the DevOps deployment API. The DevOps deployment API handles checkout, copy, Kubernetes update, restart, rollout, or any other deployment behavior.

## Architecture

```text
User
  |
  v
Config Management UI
  |
  v
Config Manager Backend
  |
  +--> GitLab API
  |
  +<-- GitLab Webhook
  |
  +--> DevOps Deployment API
  |
  v
Database
Main Components
Frontend
The frontend provides:
Application selector
Environment selector
Config file browser
YAML/properties editor
Validation results panel
Diff preview
Create merge request flow
Change request status view
Deployment status view
Audit history view
Admin mapping screens
The frontend must not call GitLab directly for write operations.
Backend
The backend is a Spring Boot service responsible for:
Authentication and authorization enforcement
Application/environment metadata
GitLab file reads
GitLab branch creation
GitLab commits
GitLab merge request creation
Configuration validation
Diff generation
Change request tracking
GitLab webhook verification
Deployment orchestration
Audit logging
GitLab
GitLab stores configuration files and performs the review/approval/merge workflow.
The backend integrates with GitLab APIs to:
Read repository files
Create branches
Create commits
Create merge requests
Optionally poll MR status for reconciliation
DevOps Deployment API
The DevOps deployment API is an external service that deploys merged configuration.
The backend calls the DevOps API only after verifying a merged GitLab MR event.
The DevOps API is responsible for:
Checking out merged config
Selecting environment-specific files
Copying config files to the required resource/config folder or Kubernetes-mounted config location
Triggering any needed Kubernetes rollout or restart
Returning deployment request status
Database
The database stores application metadata, environment metadata, change request metadata, file change metadata, deployment events, webhook events, and audit logs.
The database must not store raw secret values.
Recommended Technology Stack
Backend
Java 21
Spring Boot 3.x
Spring Web
Spring Security
OAuth2/OIDC Resource Server or SAML/OIDC integration as required
Spring Data JPA
PostgreSQL
Flyway or Liquibase
WebClient for GitLab and DevOps API calls
Jackson YAML or SnakeYAML Engine for YAML parsing
Micrometer for metrics
Actuator for health checks
Frontend
React with TypeScript, or Angular with TypeScript
Monaco Editor or CodeMirror for YAML/properties editing
Diff viewer component
Enterprise SSO integration through backend/session or token flow
Form validation library
API client generated or manually typed
Infrastructure
Docker
Kubernetes
Ingress
Kubernetes secrets or approved secret manager
Centralized logging
Metrics dashboard
GitLab webhook configured to call backend webhook endpoint
Configuration Repository Layout
Recommended default layout:
Plain text
config-repo/
  services/
    order-service/
      dev/
        application.yml
        feature-flags.yml
      qa/
        application.yml
        feature-flags.yml
      prod/
        application.yml
        feature-flags.yml

    payment-service/
      dev/
        application.yml
      qa/
        application.yml
      prod/
        application.yml
Alternative layout can be supported through metadata mappings:
Plain text
{basePath}/{application}/{environment}/{fileName}
Example resolved path:
Plain text
services/order-service/prod/application.yml
Change Request Lifecycle
Plain text
DRAFT
  -> VALIDATED
  -> BRANCH_CREATED
  -> COMMITTED
  -> MR_CREATED
  -> WAITING_FOR_MERGE
  -> MERGED
  -> DEPLOYMENT_QUEUED
  -> DEPLOYMENT_REQUESTED
  -> DEPLOYING
  -> DEPLOYED
Failure/cancellation states:
Plain text
VALIDATION_FAILED
BRANCH_CREATE_FAILED
COMMIT_FAILED
MR_CREATE_FAILED
MR_CLOSED_WITHOUT_MERGE
WEBHOOK_VERIFICATION_FAILED
DEPLOYMENT_REQUEST_FAILED
DEPLOYMENT_FAILED
CANCELLED
Deployment Trigger Rule
Deployment must be triggered only when all conditions are true:
Webhook event is a GitLab merge request event.
Event action indicates merge.
MR state is merged.
GitLab project ID matches the stored change request.
MR IID matches the stored change request.
Source branch matches the generated source branch.
Target branch matches the configured target branch.
Merge commit SHA is present.
Deployment has not already been requested for the same MR and merge commit.
Deployment must not be triggered on:
MR opened
MR updated
MR approved
MR approval removed
MR closed without merge
Push events not associated with the stored MR
Unknown branch
Unknown project
Failed webhook verification
Backend Package Structure
Plain text
com.company.configmanager
  ConfigManagerApplication.java

  config
    SecurityConfig.java
    WebClientConfig.java
    ObjectMapperConfig.java

  controller
    ApplicationController.java
    ConfigFileController.java
    ChangeRequestController.java
    GitlabWebhookController.java
    AdminController.java

  service
    ApplicationService.java
    EnvironmentService.java
    ConfigReadService.java
    ConfigValidationService.java
    DiffService.java
    ChangeRequestService.java
    GitlabBranchService.java
    GitlabCommitService.java
    GitlabMergeRequestService.java
    GitlabWebhookService.java
    DeploymentOrchestrationService.java
    AuditService.java

  client
    GitlabClient.java
    DevOpsDeploymentClient.java

  entity
    ApplicationEntity.java
    EnvironmentEntity.java
    ConfigChangeRequestEntity.java
    ConfigChangeFileEntity.java
    DeploymentEventEntity.java
    AuditLogEntity.java
    WebhookEventEntity.java

  repository
    ApplicationRepository.java
    EnvironmentRepository.java
    ConfigChangeRequestRepository.java
    ConfigChangeFileRepository.java
    DeploymentEventRepository.java
    AuditLogRepository.java
    WebhookEventRepository.java

  dto
    ApplicationDto.java
    EnvironmentDto.java
    ConfigFileDto.java
    ValidateConfigRequest.java
    ValidateConfigResponse.java
    DiffRequest.java
    DiffResponse.java
    CreateChangeRequest.java
    ChangeRequestDto.java
    GitlabWebhookPayload.java
    DeploymentRequest.java
    DeploymentResponse.java

  security
    CurrentUser.java
    AuthorizationService.java
    GitlabWebhookVerifier.java

  validation
    ConfigRule.java
    YamlSyntaxRule.java
    PropertiesSyntaxRule.java
    DuplicateYamlKeyRule.java
    SecretDetectionRule.java
    RequiredKeysRule.java
    ProductionSafetyRule.java
Data Model
applications
SQL
CREATE TABLE applications (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL UNIQUE,
    display_name VARCHAR(255) NOT NULL,
    description TEXT,
    gitlab_project_id BIGINT NOT NULL,
    config_base_path VARCHAR(1000) NOT NULL,
    owner_team VARCHAR(255),
    enabled BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
environments
SQL
CREATE TABLE environments (
    id UUID PRIMARY KEY,
    application_id UUID NOT NULL REFERENCES applications(id),
    name VARCHAR(100) NOT NULL,
    target_branch VARCHAR(255) NOT NULL,
    config_path_template VARCHAR(1000) NOT NULL,
    protected BOOLEAN NOT NULL DEFAULT FALSE,
    devops_environment_name VARCHAR(255) NOT NULL,
    enabled BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    UNIQUE(application_id, name)
);
config_change_requests
SQL
CREATE TABLE config_change_requests (
    id UUID PRIMARY KEY,
    application_id UUID NOT NULL REFERENCES applications(id),
    environment_id UUID NOT NULL REFERENCES environments(id),
    requested_by VARCHAR(255) NOT NULL,
    status VARCHAR(100) NOT NULL,
    source_branch VARCHAR(500),
    target_branch VARCHAR(500) NOT NULL,
    gitlab_project_id BIGINT NOT NULL,
    gitlab_mr_iid BIGINT,
    gitlab_mr_url VARCHAR(2000),
    base_commit_sha VARCHAR(100),
    merge_commit_sha VARCHAR(100),
    deployment_request_id VARCHAR(255),
    idempotency_key VARCHAR(500),
    error_message TEXT,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
config_change_files
SQL
CREATE TABLE config_change_files (
    id UUID PRIMARY KEY,
    change_request_id UUID NOT NULL REFERENCES config_change_requests(id),
    file_path VARCHAR(1000) NOT NULL,
    change_type VARCHAR(50) NOT NULL,
    old_content_hash VARCHAR(128),
    new_content_hash VARCHAR(128),
    created_at TIMESTAMP NOT NULL
);
deployment_events
SQL
CREATE TABLE deployment_events (
    id UUID PRIMARY KEY,
    change_request_id UUID NOT NULL REFERENCES config_change_requests(id),
    status VARCHAR(100) NOT NULL,
    request_payload_hash VARCHAR(128),
    response_code INTEGER,
    response_body TEXT,
    error_message TEXT,
    created_at TIMESTAMP NOT NULL
);
webhook_events
SQL
CREATE TABLE webhook_events (
    id UUID PRIMARY KEY,
    event_id VARCHAR(255),
    gitlab_project_id BIGINT,
    object_kind VARCHAR(100),
    event_action VARCHAR(100),
    gitlab_mr_iid BIGINT,
    processing_status VARCHAR(100) NOT NULL,
    error_message TEXT,
    received_at TIMESTAMP NOT NULL,
    UNIQUE(event_id)
);
audit_log
SQL
CREATE TABLE audit_log (
    id UUID PRIMARY KEY,
    actor VARCHAR(255) NOT NULL,
    action VARCHAR(255) NOT NULL,
    entity_type VARCHAR(255) NOT NULL,
    entity_id VARCHAR(255),
    application_name VARCHAR(255),
    environment_name VARCHAR(255),
    details TEXT,
    created_at TIMESTAMP NOT NULL
);
REST API
Applications
Http
GET /api/apps
GET /api/apps/{appId}
GET /api/apps/{appId}/envs
Config Files
Http
GET /api/apps/{appId}/envs/{envId}/files
GET /api/apps/{appId}/envs/{envId}/files/{fileName}
POST /api/config/validate
POST /api/config/diff
Change Requests
Http
POST /api/change-requests
GET /api/change-requests
GET /api/change-requests/{changeRequestId}
POST /api/change-requests/{changeRequestId}/retry-deployment
POST /api/change-requests/{changeRequestId}/reconcile-status
Webhooks
Http
POST /api/webhooks/gitlab
Admin
Http
GET /api/admin/apps
POST /api/admin/apps
PUT /api/admin/apps/{appId}

GET /api/admin/apps/{appId}/envs
POST /api/admin/apps/{appId}/envs
PUT /api/admin/apps/{appId}/envs/{envId}
Core Request/Response Contracts
Validate Config Request
JSON
{
  "applicationId": "uuid",
  "environmentId": "uuid",
  "files": [
    {
      "filePath": "services/order-service/prod/application.yml",
      "content": "server:\n  port: 8080\n",
      "format": "YAML"
    }
  ]
}
Validate Config Response
JSON
{
  "valid": true,
  "errors": [],
  "warnings": [
    {
      "filePath": "services/order-service/prod/application.yml",
      "message": "logging.level.root is DEBUG in a non-production environment",
      "line": 12,
      "severity": "WARNING"
    }
  ]
}
Create Change Request
JSON
{
  "applicationId": "uuid",
  "environmentId": "uuid",
  "title": "Update order-service prod timeout",
  "description": "Increase downstream timeout for incident remediation",
  "files": [
    {
      "filePath": "services/order-service/prod/application.yml",
      "changeType": "UPDATE",
      "content": "server:\n  port: 8080\n"
    }
  ]
}
Change Request Response
JSON
{
  "id": "uuid",
  "status": "MR_CREATED",
  "sourceBranch": "config/order-service/prod/change-uuid",
  "targetBranch": "main",
  "gitlabMrIid": 123,
  "gitlabMrUrl": "https://gitlab.example.com/group/config-repo/-/merge_requests/123"
}
DevOps Deployment Request
JSON
{
  "application": "order-service",
  "environment": "prod",
  "devopsEnvironmentName": "production",
  "gitlabProjectId": 12345,
  "targetBranch": "main",
  "mergeRequestIid": 123,
  "mergeCommitSha": "abc123def456",
  "configPath": "services/order-service/prod",
  "changeRequestId": "uuid",
  "requestedBy": "user@example.com",
  "idempotencyKey": "order-service:prod:123:abc123def456"
}
GitLab Integration Design
Read File
The backend reads file content from GitLab using project ID, file path, and branch ref.
The backend stores the last commit SHA for traceability.
Create Commit
The backend creates one commit with multiple file actions where possible.
Supported file actions:
create
update
delete
Create Merge Request
The backend creates a merge request with:
Source branch
Target branch
Title
Description
Labels
Remove source branch enabled where allowed
MR description must include:
Internal change request ID
Application
Environment
Requested by
Validation summary
Changed files
Rollback guidance if available
Webhook Processing Design
Webhook processing must be fast and safe.
Flow:
Plain text
Receive webhook
  -> Verify signature/token
  -> Store webhook event
  -> Ignore non-MR events
  -> Check event action/state
  -> Find local change request by project ID and MR IID
  -> Verify branch and target branch
  -> Check idempotency
  -> Mark as merged
  -> Enqueue deployment request
  -> Return 200
Deployment call should happen in a worker/service method after the webhook is accepted or in an async transaction-safe process.
Validation Design
Validation rules are pluggable.
Initial rules:
YAML syntax rule
Properties syntax rule
Duplicate YAML key rule
Required key rule
Secret detection rule
Production safety rule
File path allowlist rule
Environment branch rule
Max file size rule
Example production safety rules:
Block logging.level.root=DEBUG
Block disabling authentication
Block empty datasource URLs
Block plain-text passwords
Block unknown config files
Warn on very high timeout values
Warn on feature flags without owner or expiration
Secret Handling
The system must not store raw secrets in the database.
Secret masking patterns should include:
password
passwd
secret
token
apiKey
api-key
clientSecret
privateKey
credential
Masking example:
Plain text
database.password: ********
api.token=********
The system should either reject plain-text secrets or require secret references depending on organizational policy.
Authorization Model
Authorization must support:
View application
Edit application config
Submit change request
Access protected environment
Retry deployment
Administer application mappings
Suggested roles:
Plain text
CONFIG_VIEWER
CONFIG_EDITOR
PROD_CONFIG_EDITOR
CONFIG_ADMIN
DEPLOYMENT_RETRY_OPERATOR
Authorization decisions should include:
User identity
User groups
Application owner team
Environment protected flag
Operation type
Observability
Logs
Log structured events for:
Config read
Validation result
Branch creation
Commit creation
MR creation
Webhook received
Webhook ignored
Webhook rejected
Deployment API called
Deployment API failed
Deployment retry
Each log should include:
correlationId
changeRequestId
application
environment
gitlabProjectId
gitlabMrIid where available
Metrics
Expose metrics for:
config_validation_success_total
config_validation_failure_total
gitlab_api_failure_total
merge_request_created_total
webhook_received_total
webhook_rejected_total
deployment_requested_total
deployment_failed_total
deployment_retry_total
Health Checks
Expose:
Http
GET /actuator/health
GET /actuator/info
GET /actuator/metrics
Optional health indicators:
Database health
GitLab API health
DevOps API health
Error Handling
GitLab Errors
Unauthorized: fail fast and alert admin.
File not found: return user-friendly not-found message.
Branch already exists: generate alternate branch name or fail with clear message.
Merge conflict: mark change request as conflict and ask user to rebase/recreate.
Rate limit: retry with backoff when safe.
DevOps API Errors
400: mark as failed, no automatic retry unless configured.
401/403: mark as failed and alert admin.
409: treat according to idempotency response.
429/500/502/503/504: retry with backoff.
Timeout: retry with idempotency key.
Testing Strategy
Unit Tests
Validation rules
Diff masking
Authorization checks
Webhook event classification
Idempotency key generation
Change request state transitions
Integration Tests
GitLab client with mocked GitLab API
DevOps client with mocked DevOps API
Database repositories
Webhook endpoint verification
Change request creation flow
End-to-End Tests
User reads config
User edits config
Validation passes
MR is created
Merge webhook is received
Deployment API is called
Status is updated
Negative Tests
Invalid YAML
Duplicate YAML keys
Unauthorized environment access
Webhook signature failure
MR approved but not merged
Unknown MR webhook
Duplicate webhook
DevOps API failure
Plain-text secret detection
Security Notes
Store GitLab token only on backend.
Store DevOps token only on backend.
Use least privilege service accounts.
Use HTTPS for all external calls.
Verify GitLab webhook tokens/signatures.
Never log raw config content unless explicitly approved.
Never log raw secret-like values.
Use audit logs for user and system actions.
Require protected-environment authorization for production changes.
Open Questions
What exact GitLab authentication method should be used: project access token, group access token, or service account PAT?
What exact webhook verification method is enabled in the organization?
What is the exact DevOps deployment API contract?
Does the DevOps API return synchronous deployment status or only accepted/request ID?
Should deployment status be polled from DevOps API?
Should the UI support rollback by reverting prior Git commits?
Should plain-text secrets be blocked entirely or allowed with warnings for non-production?
What is the maximum supported config file size?

---

# tasks.md

```markdown id="7d1feh"
# Implementation Tasks

## Phase 1: Project Setup

### Task 1.1: Create backend Spring Boot project

- Set up Java 21 Spring Boot application.
- Add dependencies:
  - Spring Web
  - Spring Security
  - OAuth2 Resource Server or selected auth module
  - Spring Data JPA
  - PostgreSQL driver
  - Flyway or Liquibase
  - Actuator
  - Micrometer
  - Validation
  - WebClient
  - YAML parser
- Configure application profiles for local, dev, qa, and prod.

Acceptance Criteria:

- Backend starts locally.
- Health endpoint works.
- Database connection is configurable.
- Basic security configuration is present.

### Task 1.2: Create frontend project

- Set up React/TypeScript or Angular/TypeScript frontend.
- Add routing.
- Add API client layer.
- Add authentication integration placeholder.
- Add layout shell.

Acceptance Criteria:

- Frontend starts locally.
- App shell renders.
- API base URL is configurable.

### Task 1.3: Add database migration framework

- Add initial schema migration.
- Create tables:
  - applications
  - environments
  - config_change_requests
  - config_change_files
  - deployment_events
  - webhook_events
  - audit_log

Acceptance Criteria:

- Database migrations run automatically.
- Tables are created.
- Backend can connect to database.

## Phase 2: Application and Environment Metadata

### Task 2.1: Implement application entity and API

- Create ApplicationEntity.
- Create repository.
- Create service.
- Create REST endpoints:
  - GET /api/apps
  - GET /api/apps/{appId}
  - Admin create/update endpoints.

Acceptance Criteria:

- Applications can be created by admin.
- Applications can be listed by authorized users.
- Disabled applications are hidden from normal users.

### Task 2.2: Implement environment entity and API

- Create EnvironmentEntity.
- Create repository.
- Create service.
- Create endpoints:
  - GET /api/apps/{appId}/envs
  - Admin create/update environment endpoints.

Acceptance Criteria:

- Environments can be configured per application.
- Protected environments are marked.
- Target branch and config path template are stored.

## Phase 3: GitLab Read Integration

### Task 3.1: Implement GitLab client base

- Create GitlabClient.
- Configure WebClient.
- Add GitLab base URL and token config.
- Add error handling and timeouts.

Acceptance Criteria:

- GitLab token is read from backend config only.
- Client handles 401, 403, 404, 429, and 5xx responses.
- Client has unit tests with mocked responses.

### Task 3.2: Implement repository file read

- Implement read file by project ID, file path, and ref.
- Decode Base64 content.
- Return file metadata and content.

Acceptance Criteria:

- Backend can read a configured file from GitLab.
- File content is decoded correctly.
- Missing file returns a clean error.
- Last commit ID is captured.

### Task 3.3: Implement config file listing

- Support listing configured files for an application/environment.
- Resolve file paths from environment config path template.
- Return file metadata to frontend.

Acceptance Criteria:

- UI can show available config files.
- Backend only exposes allowed configured paths.

## Phase 4: Config Editor and Validation

### Task 4.1: Add config editor UI

- Add file browser.
- Add YAML/properties editor.
- Add modified-state indicator.
- Add save/validate action.

Acceptance Criteria:

- User can view file content.
- User can edit file content.
- Unsaved changes are visible.

### Task 4.2: Implement validation engine

- Create ConfigRule interface.
- Implement YAML syntax validation.
- Implement properties syntax validation.
- Implement duplicate YAML key detection.
- Implement required key validation.
- Implement file path allowlist validation.
- Implement secret pattern detection.
- Implement production safety rules.

Acceptance Criteria:

- Invalid YAML is rejected.
- Duplicate YAML keys are rejected.
- Plain-text secret-like values are detected.
- Production unsafe changes are blocked or warned based on policy.
- Validation response includes file, line, severity, and message where possible.

### Task 4.3: Add validation UI

- Display validation results.
- Separate errors and warnings.
- Block MR creation when errors exist.

Acceptance Criteria:

- Validation errors are visible and actionable.
- MR creation is disabled when validation errors exist.

## Phase 5: Diff Preview

### Task 5.1: Implement backend diff service

- Generate diff between original GitLab content and edited content.
- Support multiple files.
- Mask secret-like values in diff output.

Acceptance Criteria:

- Backend returns diff per file.
- Secrets are masked.
- Empty diff is detected.

### Task 5.2: Implement frontend diff viewer

- Show unified or side-by-side diff.
- Show file-level changed status.
- Require user to review diff before submission.

Acceptance Criteria:

- User can see exactly what changed.
- Submission is blocked until diff is reviewed.

## Phase 6: GitLab Branch, Commit, and MR Creation

### Task 6.1: Implement branch naming strategy

- Generate deterministic branch names:
  - config/{app}/{env}/{changeRequestId}
- Sanitize application and environment names.
- Handle branch collision.

Acceptance Criteria:

- Branch names are safe for GitLab.
- Branch collision is handled cleanly.

### Task 6.2: Implement GitLab commit creation

- Use GitLab commit API.
- Support create, update, and delete file actions.
- Commit all changed files together where possible.

Acceptance Criteria:

- Backend creates a commit on a new branch.
- Multiple file changes are supported.
- Commit failures update change request status.

### Task 6.3: Implement GitLab merge request creation

- Create MR from source branch to target branch.
- Add title, description, labels, and remove-source-branch option.
- Store MR IID and URL.

Acceptance Criteria:

- MR is created in GitLab.
- MR metadata is stored locally.
- User receives MR URL.

### Task 6.4: Implement change request create endpoint

- Endpoint:
  - POST /api/change-requests
- Flow:
  - authorize
  - validate
  - create local request
  - create branch/commit/MR
  - update status
  - write audit log

Acceptance Criteria:

- User can submit valid changes.
- MR is created.
- Local status is accurate.
- Audit logs are created.

## Phase 7: Change Request Status UI

### Task 7.1: Implement change request list API

- Endpoint:
  - GET /api/change-requests
- Filters:
  - application
  - environment
  - status
  - requestedBy
  - date range

Acceptance Criteria:

- Users can view their change requests.
- Admins can view all change requests.

### Task 7.2: Implement change request detail API

- Endpoint:
  - GET /api/change-requests/{id}
- Include:
  - files changed
  - MR URL
  - status
  - deployment status
  - audit summary

Acceptance Criteria:

- User can inspect a single change request.

### Task 7.3: Implement status UI

- Show lifecycle status.
- Link to GitLab MR.
- Show deployment request status.

Acceptance Criteria:

- User can understand whether change is waiting, merged, deploying, deployed, or failed.

## Phase 8: GitLab Webhook Handling

### Task 8.1: Implement webhook verification

- Verify GitLab webhook token/signature.
- Reject invalid webhooks.
- Store webhook event metadata.

Acceptance Criteria:

- Invalid webhook is rejected.
- Valid webhook is accepted.
- Webhook event is persisted.

### Task 8.2: Implement MR merge event processing

- Parse merge request webhook.
- Ignore non-MR events.
- Ignore approval-only events.
- Process only merged MR events.
- Match project ID and MR IID to local change request.
- Verify source branch and target branch.
- Store merge commit SHA.

Acceptance Criteria:

- MR approval does not trigger deployment.
- MR merge triggers deployment workflow.
- Unknown MR events are ignored safely.
- Duplicate events are idempotent.

### Task 8.3: Add webhook tests

- Test valid merge event.
- Test invalid signature.
- Test approval event.
- Test closed event.
- Test duplicate event.
- Test unknown MR.

Acceptance Criteria:

- Webhook behavior is covered by automated tests.

## Phase 9: DevOps Deployment API Integration

### Task 9.1: Implement DevOpsDeploymentClient

- Configure base URL and token.
- Build deployment request payload.
- Add timeout and retry policy.
- Support idempotency key header or payload field.

Acceptance Criteria:

- Client can call mocked DevOps API.
- Token is backend-only.
- Timeouts and errors are handled.

### Task 9.2: Implement deployment orchestration service

- Enqueue or execute deployment request after merge.
- Generate idempotency key.
- Persist deployment event.
- Update change request status.

Acceptance Criteria:

- Deployment request is sent only after verified merge.
- Duplicate merge events do not trigger duplicate deployment.
- Failures are persisted.

### Task 9.3: Implement retry deployment endpoint

- Endpoint:
  - POST /api/change-requests/{id}/retry-deployment
- Only authorized users can retry.
- Retry uses same idempotency model or new retry attempt as agreed.

Acceptance Criteria:

- Failed deployment can be retried by authorized users.
- Retry is audited.

## Phase 10: Security and Authorization

### Task 10.1: Implement authentication

- Integrate with SSO/OIDC or selected identity provider.
- Extract user identity and groups.
- Create CurrentUser abstraction.

Acceptance Criteria:

- Backend knows authenticated user.
- Unauthenticated access is rejected.

### Task 10.2: Implement RBAC

- Define roles:
  - CONFIG_VIEWER
  - CONFIG_EDITOR
  - PROD_CONFIG_EDITOR
  - CONFIG_ADMIN
  - DEPLOYMENT_RETRY_OPERATOR
- Enforce application/environment access.

Acceptance Criteria:

- Unauthorized users cannot view restricted apps.
- Unauthorized users cannot submit changes.
- Protected environments require elevated permission.

### Task 10.3: Implement secret masking

- Mask secret-like keys in:
  - UI display
  - diff output
  - logs
  - audit records
  - validation messages

Acceptance Criteria:

- Sensitive-looking values are not exposed in normal UI/log paths.

## Phase 11: Audit and Observability

### Task 11.1: Implement audit service

- Record audit events for:
  - config read
  - validation
  - diff preview
  - change request creation
  - MR creation
  - webhook processing
  - deployment request
  - retry

Acceptance Criteria:

- Audit records are created for major actions.
- Audit records avoid raw secrets.

### Task 11.2: Implement audit UI

- Add audit list view.
- Add filters by app, env, actor, action, date.
- Add change request audit detail view.

Acceptance Criteria:

- Admins can inspect audit trail.

### Task 11.3: Add metrics and structured logging

- Add metrics for validation, GitLab, webhook, deployment.
- Add correlation IDs.
- Add change request IDs to logs.

Acceptance Criteria:

- Operational dashboards can be built from metrics.
- Logs can trace one change request end-to-end.

## Phase 12: Reconciliation and Hardening

### Task 12.1: Implement MR status reconciliation

- Add manual reconcile endpoint.
- Optionally add scheduled reconciliation for open MRs.
- Query GitLab MR status and update local state.

Acceptance Criteria:

- Missed webhooks can be recovered.
- Closed or merged MRs are reflected locally.

### Task 12.2: Implement error recovery states

- Add status transitions for:
  - GitLab branch failure
  - GitLab commit failure
  - MR creation failure
  - deployment request failure
  - deployment failure

Acceptance Criteria:

- Failed states are clear.
- User can see actionable next steps.

### Task 12.3: Add production readiness checks

- Review token permissions.
- Review webhook endpoint security.
- Review logging for secrets.
- Review DB migrations.
- Review deployment config.
- Review health checks.
- Review performance on large config files.

Acceptance Criteria:

- System is ready for non-production deployment.

## Phase 13: End-to-End Testing

### Task 13.1: Build E2E happy path test

Flow:

1. User selects app and environment.
2. User reads config.
3. User edits config.
4. User validates.
5. User reviews diff.
6. User creates MR.
7. Mock GitLab sends merge webhook.
8. System calls mocked DevOps API.
9. Status updates to deployment requested/deployed.

Acceptance Criteria:

- Happy path passes end-to-end.

### Task 13.2: Build E2E negative path tests

Scenarios:

- Invalid YAML blocks MR.
- Unauthorized user cannot edit prod.
- MR approval does not deploy.
- MR closed does not deploy.
- Invalid webhook rejected.
- Duplicate webhook does not duplicate deployment.
- DevOps API failure marks deployment failed.

Acceptance Criteria:

- Critical safety scenarios are covered.
One-shot prompt for Kiro
Plain text
Build a GitLab-based Spring Boot configuration management web application.

The application must let authorized users view, edit, validate, diff, and submit environment-specific Spring Boot configuration files stored in GitLab.

Do not use Spring Cloud Config Server. Do not deploy directly to Kubernetes. Do not copy files directly from this application.

Deployment is handled by an existing DevOps deployment API. This application should create GitLab branches, commits, and merge requests. After a merge request is approved and merged in GitLab, a GitLab webhook should call this backend. The backend should verify the merge event and then call the DevOps deployment API with application, environment, config path, target branch, MR IID, merge commit SHA, requester, change request ID, and idempotency key.

Use a Spring Boot backend, a TypeScript frontend, PostgreSQL, GitLab API integration, secure webhook handling, validation rules, audit logs, RBAC, and deployment status tracking.

Use the requirements, design, and tasks from this spec as the source of truth.