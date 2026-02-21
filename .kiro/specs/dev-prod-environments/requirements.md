# Requirements Document

## Introduction

This document specifies requirements for implementing separate development and production environments for the Orcutt Schools chatbot system. Currently, all development and testing occurs directly in the production environment, creating risk when making changes. This feature will establish isolated dev and prod environments within the same AWS account, enabling safe testing of changes before production deployment.

The current environment is named "dev" but is actually serving production traffic. This feature will address this naming issue by properly designating environments.

Since the primary goal is testing code changes (not content changes), both environments will share a single Knowledge Base to minimize costs. The estimated cost increase will be significantly less than doubling, primarily from additional Lambda, DynamoDB, and CloudFront resources.

## Glossary

- **Dev_Environment**: The development environment used for testing changes, experiments, and new features before production deployment
- **Prod_Environment**: The production environment serving end users with stable, tested functionality
- **CDK_Stack**: AWS Cloud Development Kit infrastructure-as-code stack that provisions AWS resources
- **Knowledge_Base**: AWS Bedrock Knowledge Base containing web-crawled content for RAG (Retrieval Augmented Generation), shared between both environments
- **Lambda_Function**: AWS Lambda serverless function handling chatbot logic and API requests
- **DynamoDB_Table**: AWS DynamoDB NoSQL database storing conversation history and session data
- **CloudFront_Distribution**: AWS CloudFront CDN distribution serving the chatbot frontend
- **ENVIRONMENT_Variable**: Configuration variable in config.yaml that determines which environment to deploy
- **Custom_Domain**: User-facing domain name mapped to CloudFront distribution
- **Deployment_Workflow**: The process of deploying changes from development to production

## Requirements

### Requirement 1: Environment Isolation

**User Story:** As a developer, I want separate dev and prod environments, so that I can test changes without affecting end users.

#### Acceptance Criteria

1. THE CDK_Stack SHALL provision separate infrastructure resources for Dev_Environment and Prod_Environment
2. WHEN deploying to Dev_Environment, THE CDK_Stack SHALL use the ENVIRONMENT_Variable value "dev"
3. WHEN deploying to Prod_Environment, THE CDK_Stack SHALL use the ENVIRONMENT_Variable value "prod"
4. THE Dev_Environment SHALL have no shared resources with Prod_Environment
5. WHEN a change is deployed to Dev_Environment, THE Prod_Environment SHALL remain unaffected

### Requirement 2: Shared Knowledge Base

**User Story:** As a project stakeholder, I want a single shared Knowledge Base, so that I can minimize AWS costs while still testing code changes effectively.

#### Acceptance Criteria

1. THE CDK_Stack SHALL create a single Knowledge_Base shared by both Dev_Environment and Prod_Environment
2. WHEN Dev_Environment Lambda_Function queries the Knowledge_Base, THE results SHALL be identical to Prod_Environment queries
3. WHEN Prod_Environment Lambda_Function queries the Knowledge_Base, THE results SHALL be identical to Dev_Environment queries
4. THE Knowledge_Base SHALL be updated independently of environment deployments
5. WHEN testing code changes in Dev_Environment, THE shared Knowledge_Base SHALL provide consistent content for validation

### Requirement 3: Current Environment Migration

**User Story:** As a system administrator, I want to properly designate the current "dev" environment as production, so that the naming accurately reflects its actual usage.

#### Acceptance Criteria

1. THE system SHALL provide a migration path to rename the current "OrcuttChatbotStack-dev" to "OrcuttChatbotStack-prod"
2. WHEN migrating the current environment, THE production URL "https://orcutt-ai.techreformers.com" SHALL remain functional without downtime
3. WHEN migrating the current environment, THE CloudFront distribution "https://d3dolln1x7yei7.cloudfront.net" SHALL be preserved or seamlessly transitioned
4. THE migration process SHALL preserve all existing DynamoDB conversation data
5. THE system SHALL provide documentation for the migration process with rollback procedures

### Requirement 4: Separate Lambda Functions

**User Story:** As a developer, I want separate Lambda functions for dev and prod, so that I can test prompt changes, code updates, and configuration changes safely.

#### Acceptance Criteria

1. THE CDK_Stack SHALL create separate Lambda_Function instances for Dev_Environment and Prod_Environment
2. WHEN Dev_Environment Lambda_Function is updated, THE Prod_Environment Lambda_Function SHALL remain unchanged
3. WHEN Prod_Environment Lambda_Function is updated, THE Dev_Environment Lambda_Function SHALL remain unchanged
4. THE Dev_Environment Lambda_Function SHALL connect only to Dev_Environment resources
5. THE Prod_Environment Lambda_Function SHALL connect only to Prod_Environment resources

### Requirement 5: Separate DynamoDB Tables

**User Story:** As a developer, I want separate DynamoDB tables for dev and prod, so that development testing conversations don't pollute production data.

#### Acceptance Criteria

1. THE CDK_Stack SHALL create separate DynamoDB_Table instances for Dev_Environment and Prod_Environment
2. WHEN conversations are stored in Dev_Environment, THE Prod_Environment DynamoDB_Table SHALL remain unchanged
3. WHEN conversations are stored in Prod_Environment, THE Dev_Environment DynamoDB_Table SHALL remain unchanged
4. THE Dev_Environment DynamoDB_Table SHALL be independently queryable from Prod_Environment DynamoDB_Table
5. THE Dev_Environment DynamoDB_Table SHALL have the same schema as Prod_Environment DynamoDB_Table

### Requirement 6: Separate CloudFront Distributions

**User Story:** As a developer, I want separate CloudFront distributions for dev and prod, so that frontend changes can be tested independently.

#### Acceptance Criteria

1. THE CDK_Stack SHALL create separate CloudFront_Distribution instances for Dev_Environment and Prod_Environment
2. WHEN Dev_Environment CloudFront_Distribution is updated, THE Prod_Environment CloudFront_Distribution SHALL remain unchanged
3. WHEN Prod_Environment CloudFront_Distribution is updated, THE Dev_Environment CloudFront_Distribution SHALL remain unchanged
4. THE Dev_Environment CloudFront_Distribution SHALL serve content from Dev_Environment resources only
5. THE Prod_Environment CloudFront_Distribution SHALL serve content from Prod_Environment resources only

### Requirement 7: Custom Domain Configuration

**User Story:** As an end user, I want clear, separate URLs for dev and prod environments, so that I know which environment I'm using.

#### Acceptance Criteria

1. THE Prod_Environment SHALL be accessible at the Custom_Domain "https://orcutt-ai.techreformers.com"
2. THE Dev_Environment SHALL be accessible at a separate Custom_Domain distinct from production
3. WHEN a user accesses the Dev_Environment Custom_Domain, THE system SHALL connect to Dev_Environment resources only
4. WHEN a user accesses the Prod_Environment Custom_Domain, THE system SHALL connect to Prod_Environment resources only
5. THE CDK_Stack SHALL configure SSL certificates for both Custom_Domain instances

### Requirement 8: Environment-Specific Configuration

**User Story:** As a developer, I want environment-specific configuration in config.yaml, so that I can customize settings for dev and prod independently.

#### Acceptance Criteria

1. THE config.yaml SHALL contain separate configuration sections for Dev_Environment and Prod_Environment
2. WHEN deploying to Dev_Environment, THE CDK_Stack SHALL read configuration from the dev section of config.yaml
3. WHEN deploying to Prod_Environment, THE CDK_Stack SHALL read configuration from the prod section of config.yaml
4. THE config.yaml SHALL support environment-specific values for all configurable parameters
5. WHERE cost optimization is desired, THE config.yaml SHALL allow reduced resource sizes for Dev_Environment

### Requirement 9: Deployment Workflow

**User Story:** As a developer, I want a clear deployment workflow, so that I can safely promote changes from dev to prod.

#### Acceptance Criteria

1. THE Deployment_Workflow SHALL require testing in Dev_Environment before Prod_Environment deployment
2. THE CDK_Stack SHALL support deploying to Dev_Environment with a single command
3. THE CDK_Stack SHALL support deploying to Prod_Environment with a single command
4. WHEN deploying to Prod_Environment, THE system SHALL not automatically deploy to Dev_Environment
5. WHEN deploying to Dev_Environment, THE system SHALL not automatically deploy to Prod_Environment

### Requirement 10: Automated Validation

**User Story:** As a developer, I want automated validation after deployments, so that I can catch issues before they reach production users.

#### Acceptance Criteria

1. WHEN a deployment to Dev_Environment completes, THE system SHALL provide a mechanism to run automated health checks
2. THE automated health checks SHALL verify Lambda_Function connectivity and basic functionality
3. THE automated health checks SHALL verify DynamoDB_Table accessibility
4. THE automated health checks SHALL verify CloudFront_Distribution is serving content correctly
5. WHEN health checks fail in Dev_Environment, THE system SHALL provide clear error messages to prevent Prod_Environment deployment

### Requirement 11: Cost Estimation and Monitoring

**User Story:** As a project stakeholder, I want cost estimation and monitoring for both environments, so that I can track AWS spending and optimize costs.

#### Acceptance Criteria

1. THE system SHALL provide estimated monthly costs for Dev_Environment
2. THE system SHALL provide estimated monthly costs for Prod_Environment
3. THE system SHALL provide total estimated monthly costs for both environments combined
4. WHERE cost optimization is implemented, THE system SHALL document cost savings strategies
5. THE system SHALL enable AWS cost monitoring tags for Dev_Environment and Prod_Environment resources

### Requirement 12: Documentation

**User Story:** As a developer, I want clear documentation on environment usage, so that I know when to use dev vs prod and how to deploy to each.

#### Acceptance Criteria

1. THE system SHALL provide documentation explaining when to use Dev_Environment vs Prod_Environment
2. THE system SHALL provide documentation for deploying to Dev_Environment
3. THE system SHALL provide documentation for deploying to Prod_Environment
4. THE system SHALL provide documentation for the recommended Deployment_Workflow
5. THE system SHALL provide documentation for cost optimization strategies in Dev_Environment

### Requirement 13: Resource Naming Convention

**User Story:** As a developer, I want clear resource naming conventions, so that I can easily identify which resources belong to which environment in the AWS console.

#### Acceptance Criteria

1. THE CDK_Stack SHALL append the ENVIRONMENT_Variable value to all resource names
2. WHEN viewing AWS resources, THE Dev_Environment resources SHALL be clearly distinguishable from Prod_Environment resources
3. THE resource naming convention SHALL be consistent across all AWS services
4. THE CDK_Stack SHALL use the format "{ResourceName}-{ENVIRONMENT_Variable}" for resource names
5. THE stack name SHALL include the ENVIRONMENT_Variable value

### Requirement 14: Cost Optimization for Development

**User Story:** As a project stakeholder, I want cost optimization options for the dev environment, so that I can minimize AWS costs while maintaining testing capability.

#### Acceptance Criteria

1. WHERE cost optimization is enabled, THE Dev_Environment Lambda_Function SHALL support reduced memory allocation compared to Prod_Environment
2. WHERE cost optimization is enabled, THE Dev_Environment SHALL support reduced DynamoDB_Table capacity compared to Prod_Environment
3. THE cost optimization settings SHALL be configurable in config.yaml
4. THE Prod_Environment SHALL not be affected by Dev_Environment cost optimization settings
5. THE shared Knowledge_Base SHALL minimize overall infrastructure costs compared to separate Knowledge Bases
