  # Orcutt Schools Chatbot

**This repository is maintained by Tech Reformers and is a fork of the original work by Cal Poly DxHub.**

Original repository: [https://github.com/cal-poly-dxhub/orcutt-schools-chatbot](https://github.com/cal-poly-dxhub/orcutt-schools-chatbot/tree/main)

## Table of Contents

- [About This Fork](#about-this-fork)
- [Disclaimers](#disclaimers)
- [Overview](#overview)
- [Architecture](#architecture)
- [Support](#support)
- [Deployment](#initial-setup)
- [Webscraping](#webscraping)
- [Troubleshooting](#troubleshooting)

# About This Fork

This repository is hosted and maintained by Tech Reformers. We have forked the original Orcutt Schools Chatbot project created by Cal Poly DxHub to continue development and provide enhanced deployment documentation.

**Original Authors:**
- Shrey Shah - <sshah84@calpoly.edu>

**Original Collaboration Note:**
Thanks for your interest in this solution. Having specific examples of replication and usage allows us to continue to grow and scale our work. If you use this repository, please let us know at <info@techreformers.com>

# Disclaimers

**Customers are responsible for making their own independent assessment of the information in this document.**

**This document:**

(a) is for informational purposes only,

(b) represents current AWS product offerings and practices, which are subject to change without notice, and

(c) does not create any commitments or assurances from AWS and its affiliates, suppliers or licensors. AWS products or services are provided “as is” without warranties, representations, or conditions of any kind, whether express or implied. The responsibilities and liabilities of AWS to its customers are controlled by AWS agreements, and this document is not part of, nor does it modify, any agreement between AWS and its customers.

(d) is not to be considered a recommendation or viewpoint of AWS

**Additionally, all prototype code and associated assets should be considered:**

(a) as-is and without warranties

(b) not suitable for production environments

(d) to include shortcuts in order to support rapid prototyping such as, but not limitted to, relaxed authentication and authorization and a lack of strict adherence to security best practices

**All work produced is open source. More information can be found in the GitHub repo.**



## Overview

An AI-powered chatbot built for Orcutt Schools to help students, parents, and staff get information about school programs, schedules, and policies.

## Architecture

The solution consists of several key components:

1. Frontend Interface

    - React 18 application
    - S3 + Cloudfront Hosting
    - Tailwind CSS for responsive design

2. API Layer

    - Amazon API Gateway for REST endpoints
    - AWS Lambda functions for serverless compute

3. AI Services

    - Amazon Bedrock with Claude 3.5 Sonnet V2 for response generation
    - AWS Knowledge Bases for semantic document search
    - Nova Lite for query classification and intent recognition

4. Data Storage and Management

    - Amazon DynamoDB for conversation history and user sessions
    - S3 buckets for document storage and knowledge base artifacts
    - Amazon CloudWatch for application monitoring and logging

Additionally other AWS services are used for additional functionality

## Prerequisites

- AWS CLI configured with appropriate permissions
- Node.js 18+ (for frontend) - Note: Node 23 is end-of-life; versions 20, 22, or 24 are recommended
- Python 3.13+ (for CDK and Lambda functions)
- AWS CDK CLI installed (`npm install -g aws-cdk`)
- Request model access for the required models through AWS console in Bedrock (Amazon Titan Text V2, Claude Sonnet 3.5 V2 & Amazon Nova Lite)
- Docker Desktop (must be running before deployment)


## Initial Setup

1. **Enable Bedrock Model Access:**
   - Navigate to the AWS Bedrock console
   - Request access to all models from both Anthropic and Amazon
   - Ensure you're working in the correct AWS region/branch for your deployment
  
2. **Download and Start Docker Desktop**
   - Verify Docker is running:
    ```bash
    docker --version
    ```
    
## Deployment Steps

1. Clone the repository
  - git clone https://github.com/cal-poly-dxhub/orcutt-schools-chatbot

2. Run the setup script
  ```bash
  cd orcutt-schools-chatbot
  ./scripts/setup.sh
  ```
  
3. Start Docker Desktop
  - Verify Docker is running:
  ```bash
  docker --version
  ```

4. Configure AWS credentials

  **Option A: Using AWS SSO (Recommended for organizations using SSO)**
  ```bash
  aws configure sso
  ```
  You'll be prompted to enter:
  - SSO session name (e.g., your-org-name)
  - SSO start URL (e.g., https://your-org.awsapps.com/start)
  - SSO region (the region where your SSO is configured, may differ from deployment region)
  - Account and role selection
  - Default region for deployment (e.g., us-west-2)
  - Default output format (json recommended)
  
  After configuration, login with:
  ```bash
  aws sso login --profile <profile-name>
  ```
  
  **Option B: Using Access Keys**
  ```bash
  aws configure
  ```
  You'll be prompted to enter:
  - AWS Access Key ID
  - AWS Secret Access Key
  - Default region name
  - Default output format


5. Bootstrap CDK (first time only)
  
  If this is your first time using CDK in this AWS account/region, bootstrap it:
  ```bash
  # If using SSO profile
  cdk bootstrap aws://<ACCOUNT-ID>/<REGION> --profile <profile-name>
  
  # If using default credentials
  cdk bootstrap aws://<ACCOUNT-ID>/<REGION>
  ```
  Replace `<ACCOUNT-ID>` with your AWS account ID and `<REGION>` with your deployment region (e.g., us-west-2)

6. Deploy the application
  ```bash
  # If using SSO profile, set it as active
  export AWS_PROFILE=<profile-name>
  
  ./scripts/deploy.sh
  ```
  
  **Note:** The deployment process takes 10-15 minutes, primarily due to OpenSearch domain provisioning.

## Webscraping

**Important:** After initial deployment, you must run the webscraper to populate the knowledge base with content. Without this step, the chatbot will not have any information to answer questions.

To run the webscraping functionality:

  ```bash
  ./scripts/run_webscraper.sh
  ```

This script executes the webscraping lambda function, which performs the following operations:

1. Web Scraping: Scrapes content from all configured websites
2. Metadata Generation: Creates metadata files for the scraped content
3. Data Processing: Adds metadata to the scraped content
4. S3 Upload: Uploads the processed content and metadata to the S3 bucket
5. Knowledge Base Sync: Synchronizes the knowledge base with the newly added content

The entire pipeline runs automatically once the script is executed. The ingestion process typically takes 2-5 minutes to complete after the script finishes.

## Troubleshooting

### Chatbot Returns Network Error

If the chatbot frontend loads but returns a network error when you try to chat, the frontend may have been built with a placeholder API URL. To fix this:

1. Get the real API URL from CloudFormation outputs:
   - Go to AWS Console > CloudFormation > OrcuttChatbotStack-dev > Outputs
   - Copy the `ApiUrl` value (e.g., https://xxxxx.execute-api.us-west-2.amazonaws.com/prod/)

2. Rebuild the frontend with the correct API URL:
   ```bash
   cd frontend
   REACT_APP_API_BASE_URL="<your-api-url>" npm run build
   cd ..
   ```

3. Redeploy the stack:
   ```bash
   cdk deploy --require-approval never
   ```

### CDK Bootstrap Issues

If you see "No bucket named 'cdk-hnb659fds-assets-...' errors:
- Delete the existing CDKToolkit stack if it exists but is incomplete
- Re-run the bootstrap command from step 5 above

### SSO Session Expired

If using SSO and commands fail with authentication errors:
```bash
aws sso login --profile <profile-name>
```


## Improvements Made by Tech Reformers

This fork includes the following enhancements to improve chatbot accuracy and response quality:

### 1. Enhanced Source Prioritization (Commit: ab41080)
**Problem:** Chatbot was giving outdated answers from old board minutes PDFs instead of current website content.

**Solution:** Updated the system prompt to explicitly prioritize website sources over PDF documents:
- Website content is now treated as more current and authoritative
- PDFs are only used when website sources don't contain the needed information
- Removed meeting_date prioritization that was favoring recent board minutes

**Impact:** Chatbot now provides more current, accurate information from the district website.

### 2. Increased Knowledge Base Retrieval Results (Commit: b01de48)
**Problem:** Relevant information wasn't being retrieved because the result set was too small.

**Solution:** Increased main domain query results from 20 to 40.

**Impact:** More comprehensive context for the chatbot to work with, improving answer accuracy.

**Note:** Attempted upgrade to Claude Sonnet 4.5 but reverted due to AWS Service Control Policy restrictions (Commits: 3ed78d4, d54bc7b, 83e328b). Currently using Claude 3.5 Sonnet V2.

## Support

For any queries or issues with this fork, please contact:

**Tech Reformers:** <info@techreformers.com>

For questions about the original project, please refer to the [original repository](https://github.com/cal-poly-dxhub/orcutt-schools-chatbot/tree/main) or contact:
- Darren Kraker - <dkraker@amazon.com>
- Shrey Shah - <sshah84@calpoly.edu>
