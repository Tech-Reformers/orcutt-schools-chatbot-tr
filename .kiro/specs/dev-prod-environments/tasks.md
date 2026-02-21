# Implementation Plan: Dev/Prod Environments

## Overview

This implementation plan converts the dev/prod environment design into actionable coding tasks. The approach follows this sequence:

1. Update configuration structure to support multiple environments
2. Modify CDK stack to read environment-specific configuration
3. Implement resource naming with environment suffixes
4. Create health check automation
5. Document migration procedures
6. Implement testing for environment isolation

Each task builds incrementally, ensuring the system remains functional throughout implementation.

## Tasks

- [ ] 1. Update configuration structure for multi-environment support
  - Modify config.yaml to include separate dev and prod sections
  - Add environment-specific settings (stackName, domainName, lambdaMemory, dynamoDbBillingMode, knowledgeBaseId)
  - Ensure backward compatibility with existing configuration
  - _Requirements: 8.1, 8.4, 8.5_

- [ ] 2. Implement environment context reading in CDK
  - [ ] 2.1 Add environment parameter to CDK app
    - Read ENVIRONMENT from CDK context (--context environment=dev/prod)
    - Add validation to ensure environment is either "dev" or "prod"
    - Throw clear error if environment is not specified or invalid
    - _Requirements: 1.2, 1.3_
  
  - [ ] 2.2 Create configuration loader utility
    - Write function to load environment-specific config from config.yaml
    - Parse YAML and extract configuration for specified environment
    - Return typed configuration object (EnvironmentConfig interface)
    - _Requirements: 8.2, 8.3_
  
  - [ ]* 2.3 Write unit tests for configuration loader
    - Test loading dev configuration
    - Test loading prod configuration
    - Test error handling for missing environment
    - Test error handling for invalid YAML
    - _Requirements: 8.2, 8.3_

- [ ] 3. Implement resource naming with environment suffixes
  - [ ] 3.1 Update Lambda function naming
    - Modify Lambda construct to append environment suffix to function name
    - Format: "OrcuttChatbot-{environment}"
    - Update all references to Lambda function name
    - _Requirements: 13.1, 13.3, 13.5_
  
  - [ ] 3.2 Update DynamoDB table naming
    - Modify DynamoDB construct to append environment suffix to table name
    - Format: "OrcuttChatbot-Conversations-{environment}"
    - Update Lambda environment variables to reference correct table name
    - _Requirements: 13.1, 13.3, 13.5_
  
  - [ ] 3.3 Update CloudFront distribution naming
    - Modify CloudFront construct to append environment suffix to distribution name
    - Update origin configuration to point to environment-specific Lambda
    - _Requirements: 13.1, 13.3, 13.5_
  
  - [ ] 3.4 Update CDK stack naming
    - Modify stack name to include environment suffix
    - Format: "OrcuttChatbotStack-{environment}"
    - _Requirements: 13.5_
  
  - [ ]* 3.5 Write property test for resource naming convention
    - **Property 3: Resource Naming Convention Consistency**
    - **Validates: Requirements 13.1, 13.3**
    - Generate test that verifies all resources follow "{ResourceName}-{environment}" pattern
    - Test with both dev and prod configurations

- [ ] 4. Implement shared Knowledge Base logic
  - [ ] 4.1 Modify Knowledge Base construct to support reuse
    - Check if knowledgeBaseId is provided in configuration
    - If provided, reference existing Knowledge Base instead of creating new one
    - If not provided, create new Knowledge Base (first deployment only)
    - Return Knowledge Base ID for Lambda environment variables
    - _Requirements: 2.1_
  
  - [ ] 4.2 Update Lambda environment variables
    - Set KNOWLEDGE_BASE_ID from configuration or newly created KB
    - Ensure both dev and prod Lambda functions reference same KB ID
    - _Requirements: 2.1_
  
  - [ ]* 4.3 Write property test for Knowledge Base query consistency
    - **Property 1: Knowledge Base Query Consistency**
    - **Validates: Requirements 2.2**
    - Generate random queries and verify results are identical from dev and prod
    - Requires both environments to be deployed

- [ ] 5. Implement environment-specific resource configuration
  - [ ] 5.1 Configure Lambda memory based on environment
    - Read lambdaMemory from environment-specific configuration
    - Apply to Lambda function construct
    - Dev: 512MB, Prod: 1024MB (configurable)
    - _Requirements: 14.1_
  
  - [ ] 5.2 Configure DynamoDB billing mode based on environment
    - Read dynamoDbBillingMode from environment-specific configuration
    - Apply to DynamoDB table construct
    - Support both PAY_PER_REQUEST and PROVISIONED modes
    - _Requirements: 14.2_
  
  - [ ] 5.3 Configure CloudWatch log retention based on environment
    - Set log retention to 7 days for dev, 30 days for prod
    - Apply to all Lambda function log groups
    - _Requirements: 14.3_
  
  - [ ]* 5.4 Write unit tests for environment-specific configuration
    - Test Lambda memory is correctly set for dev and prod
    - Test DynamoDB billing mode is correctly set
    - Test log retention is correctly set
    - _Requirements: 14.1, 14.2, 14.3_

- [ ] 6. Checkpoint - Ensure CDK synthesis works for both environments
  - Run `cdk synth --context environment=dev` and verify output
  - Run `cdk synth --context environment=prod` and verify output
  - Ensure all resources have correct naming and configuration
  - Ask the user if questions arise

- [ ] 7. Implement IAM permissions for environment isolation
  - [ ] 7.1 Create environment-specific IAM roles for Lambda
    - Create separate IAM role for dev Lambda
    - Create separate IAM role for prod Lambda
    - Grant permissions only to environment-specific resources
    - _Requirements: 4.4, 4.5_
  
  - [ ] 7.2 Configure DynamoDB permissions
    - Grant dev Lambda access only to dev DynamoDB table
    - Grant prod Lambda access only to prod DynamoDB table
    - Use least privilege principle
    - _Requirements: 4.4, 4.5_
  
  - [ ] 7.3 Configure Bedrock Knowledge Base permissions
    - Grant both Lambda functions read access to shared Knowledge Base
    - Ensure permissions are identical for both environments
    - _Requirements: 2.1_
  
  - [ ]* 7.4 Write unit tests for IAM permissions
    - Test dev Lambda role has access to dev DynamoDB only
    - Test prod Lambda role has access to prod DynamoDB only
    - Test both roles have access to shared Knowledge Base
    - _Requirements: 4.4, 4.5_

- [ ] 8. Implement custom domain configuration
  - [ ] 8.1 Configure SSL certificates
    - Reference or create SSL certificate for dev domain
    - Reference or create SSL certificate for prod domain
    - Add certificate ARNs to config.yaml
    - _Requirements: 7.5_
  
  - [ ] 8.2 Configure CloudFront custom domains
    - Set dev CloudFront distribution to use dev domain
    - Set prod CloudFront distribution to use prod domain
    - Update CloudFront construct with domain configuration
    - _Requirements: 7.1, 7.2_
  
  - [ ] 8.3 Document DNS configuration steps
    - Create documentation for updating Route53 or DNS provider
    - Include steps for pointing dev domain to dev CloudFront
    - Include steps for pointing prod domain to prod CloudFront
    - _Requirements: 7.1, 7.2_

- [ ] 9. Implement AWS resource tagging
  - [ ] 9.1 Add environment tags to all resources
    - Tag all resources with "Environment: dev" or "Environment: prod"
    - Tag all resources with "Project: OrcuttChatbot"
    - Add cost allocation tags
    - _Requirements: 11.5_
  
  - [ ]* 9.2 Write unit tests for resource tagging
    - Test all resources have Environment tag
    - Test all resources have Project tag
    - Verify tag values match environment
    - _Requirements: 11.5_

- [ ] 10. Create health check script
  - [ ] 10.1 Implement Lambda connectivity check
    - Write script to invoke Lambda function via CloudFront URL
    - Verify response status code is 200
    - Verify response contains expected content
    - _Requirements: 10.2_
  
  - [ ] 10.2 Implement DynamoDB accessibility check
    - Write script to query DynamoDB table
    - Verify table exists and is accessible
    - Handle access denied errors gracefully
    - _Requirements: 10.3_
  
  - [ ] 10.3 Implement CloudFront check
    - Write script to make HTTP request to CloudFront URL
    - Verify SSL certificate is valid
    - Verify content is served correctly
    - _Requirements: 10.4_
  
  - [ ] 10.4 Implement error reporting
    - Return non-zero exit code on any failure
    - Output clear error messages indicating which check failed
    - Include troubleshooting suggestions in error messages
    - _Requirements: 10.5_
  
  - [ ] 10.5 Create health check wrapper script
    - Accept environment parameter (dev or prod)
    - Load environment-specific configuration
    - Run all health checks sequentially
    - Report overall success or failure
    - _Requirements: 10.1_
  
  - [ ]* 10.6 Write unit tests for health check script
    - Test successful health check scenario
    - Test Lambda failure scenario
    - Test DynamoDB failure scenario
    - Test CloudFront failure scenario
    - Test error message clarity
    - _Requirements: 10.2, 10.3, 10.4, 10.5_

- [ ] 11. Checkpoint - Test health check script
  - Deploy dev environment
  - Run health check script for dev
  - Verify all checks pass
  - Ask the user if questions arise

- [ ] 12. Implement data isolation testing
  - [ ]* 12.1 Write property test for DynamoDB data isolation
    - **Property 2: DynamoDB Data Isolation**
    - **Validates: Requirements 5.2**
    - Generate random conversation data
    - Write to dev DynamoDB table
    - Query prod DynamoDB table
    - Assert prod table does not contain dev data
    - Requires both environments to be deployed

- [ ] 13. Create migration documentation
  - [ ] 13.1 Document current state analysis
    - List all resources in current "dev" stack
    - Document resource ARNs and IDs
    - Document current domain and CloudFront configuration
    - _Requirements: 3.1_
  
  - [ ] 13.2 Document migration steps
    - Write step-by-step migration procedure
    - Include commands for deploying new prod stack
    - Include commands for data migration
    - Include DNS cutover steps
    - _Requirements: 3.1, 3.2_
  
  - [ ] 13.3 Document data migration procedure
    - Write script to export DynamoDB data from old table
    - Write script to import DynamoDB data to new prod table
    - Include data integrity verification steps
    - _Requirements: 3.4_
  
  - [ ] 13.4 Document rollback procedures
    - Write rollback steps in case migration fails
    - Include DNS rollback steps
    - Include data restoration steps
    - _Requirements: 3.5_
  
  - [ ] 13.5 Create migration checklist
    - Create pre-migration checklist
    - Create during-migration checklist
    - Create post-migration validation checklist
    - _Requirements: 3.1, 3.2, 3.4_

- [ ] 14. Create deployment documentation
  - [ ] 14.1 Document dev deployment workflow
    - Write instructions for deploying to dev
    - Include CDK commands with correct context
    - Include health check verification steps
    - _Requirements: 9.2, 12.2_
  
  - [ ] 14.2 Document prod deployment workflow
    - Write instructions for deploying to prod
    - Emphasize testing in dev first
    - Include health check verification steps
    - Include rollback procedures
    - _Requirements: 9.3, 12.3_
  
  - [ ] 14.3 Document deployment best practices
    - Document when to use dev vs prod
    - Document recommended deployment workflow (dev → test → prod)
    - Document common issues and troubleshooting
    - _Requirements: 9.1, 12.1, 12.4_

- [ ] 15. Create cost documentation
  - [ ] 15.1 Document cost breakdown
    - List estimated costs for dev environment
    - List estimated costs for prod environment
    - List shared resource costs (Knowledge Base)
    - Calculate total estimated monthly cost
    - _Requirements: 11.1, 11.2, 11.3_
  
  - [ ] 15.2 Document cost optimization strategies
    - Document shared Knowledge Base savings
    - Document reduced dev resource sizing
    - Document pay-per-request DynamoDB benefits
    - Document optional dev environment shutdown
    - _Requirements: 11.4, 14.5_
  
  - [ ] 15.3 Document cost monitoring setup
    - Document how to enable cost allocation tags
    - Document how to create cost reports by environment
    - Document how to set up budget alerts
    - _Requirements: 11.5_

- [ ] 16. Final checkpoint - Complete testing and documentation review
  - Review all documentation for completeness
  - Verify all tests pass
  - Ensure CDK can deploy to both dev and prod
  - Ask the user if ready to proceed with actual deployment

## Notes

- Tasks marked with `*` are optional testing tasks and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation at key milestones
- Property tests validate universal correctness properties (require both environments deployed)
- Unit tests validate specific examples and configuration correctness
- Migration is documented but not automated - requires manual execution with careful validation
- Health checks provide automated validation after each deployment
- Cost optimization is built into configuration (dev uses fewer resources than prod)
