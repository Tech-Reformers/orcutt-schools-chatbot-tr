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

## Current Status (February 2026)

**Production URLs:** 
- https://orcutt-ai.techreformers.com (custom domain)
- https://d3dolln1x7yei7.cloudfront.net (CloudFront URL)

**Knowledge Base Configuration:**
- KB ID: GCERPWLGOK
- Data Source: AWS Bedrock Web Crawler (managed service)
- Indexed Pages: 2,806 website pages
- Scope: www.orcuttschools.net and all subdomains
- Chunking: Semantic (buffer 1, max tokens 300, breakpoint 95%)
- Embeddings: Titan Text Embeddings v2
- Retrieval: 40 chunks, default search type (hybrid)

**Recent Fixes (Feb 2026):**
- Fixed Knowledge Base ID reference (was pointing to old KB)
- Removed domain filter that was blocking content
- Optimized retrieval to 40 chunks (matches Bedrock console test)
- Removed unnecessary reranking logic
- Migrated from custom web scraper to AWS Bedrock managed web crawler
- Improved conversational tone (removed robotic phrases like "Based on the provided context")

**Verified Working Queries:**
- "Who is the superintendent?"
- "Who are the executive directors?"
- "How do I order pizza for a classroom celebration?"
- School-specific queries using the school selector dropdown

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
- Request model access for the required models through AWS console in Bedrock (Amazon Titan Text V2, Claude 3.5 Sonnet V2 & Amazon Nova Lite)
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

## Knowledge Base Management

**The chatbot uses AWS Bedrock's managed web crawler** - no manual scraping required.

### Updating Content

When website content changes, trigger a Knowledge Base sync:

1. Go to AWS Console → Bedrock → Knowledge Bases
2. Select Knowledge Base: `OrcuttSchoolsKB` (ID: GCERPWLGOK)
3. Click on the Data Source (Web Crawler)
4. Click "Sync" button
5. Monitor sync progress (typically 2-3 hours for full site)

**Sync frequency recommendations:**
- After major website updates: Immediately
- Routine updates: Weekly or bi-weekly
- Before important events: Day before

**What gets indexed:**
- All pages on www.orcuttschools.net
- All school subdomain pages (e.g., aliceshaw.orcuttschools.net)
- Respects robots.txt rules
- Automatically chunks content for optimal retrieval

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

### 1. Migration to AWS Bedrock Web Crawler (Completed: 2026-02-09)
**Problem:** Custom web scraper required manual maintenance and didn't integrate well with AWS Bedrock Knowledge Bases.

**Solution:** 
- Migrated from custom Lambda-based web scraper to AWS Bedrock's managed web crawler
- Configured web crawler to index www.orcuttschools.net and all school subdomains
- Indexed 2,806 pages automatically with semantic chunking
- Sources now point directly to website URLs instead of S3 files

**Impact:** 
- Simplified architecture - no custom scraper code to maintain
- IT team can manage updates through AWS Bedrock console
- Automatic re-crawling on schedule or manual trigger
- Better integration with Knowledge Base retrieval

### 2. Improved Conversational Tone (Completed: 2026-02-10)
**Problem:** Chatbot responses started with robotic phrases like "Based on the provided context" and "According to the district website".

**Solution:** Updated system prompt with explicit instructions to avoid robotic intros and provided examples of natural vs. robotic responses.

**Impact:** Responses now sound like a helpful staff member rather than a robot, improving user experience.

### 3. Enhanced Source Prioritization (Commit: ab41080)
**Problem:** Chatbot was giving outdated answers from old board minutes PDFs instead of current website content.

**Solution:** Updated the system prompt to explicitly prioritize website sources over PDF documents:
- Website content is now treated as more current and authoritative
- PDFs are only used when website sources don't contain the needed information
- Removed meeting_date prioritization that was favoring recent board minutes

**Impact:** Chatbot now provides more current, accurate information from the district website.

### 4. Increased Knowledge Base Retrieval Results (Commit: b01de48)
**Problem:** Relevant information wasn't being retrieved because the result set was too small.

**Solution:** Increased main domain query results from 20 to 40.

**Impact:** More comprehensive context for the chatbot to work with, improving answer accuracy.

### 5. Few-Shot Examples for Conflicting Information (Commit: 2297f04)
**Problem:** When sources contained both district services AND restrictive policies, the chatbot would cite the policy instead of the service (e.g., saying "no food allowed" instead of explaining the pizza catering service).

**Solution:** Added few-shot examples directly in the prompt showing Claude how to handle conflicts:
- Example 1: Pizza party question - use the catering service info, not the wellness policy
- Example 2: Program enrollment - provide registration process, not just eligibility restrictions
- Clear principle: District-provided services are official offerings and take precedence over general policies

**Impact:** Chatbot now correctly identifies and prioritizes district services over restrictive policies. This approach is extensible - new examples can be added as edge cases are discovered.

### 6. Source Display Improvement (Completed: 2025-12-16)
**Problem:** Sources showed broken S3 presigned URLs instead of original website URLs, resulting in "Access Denied" errors when users clicked on sources.

**Solution:** 
- Backend: Removed presigned URL generation, sources now only contain original website URLs
- Frontend: Changed display to show `🔗 [URL]` format with clickable hyperlinks directly to source websites
- Simplified source metadata structure

**Impact:** Users can now click on sources and be taken directly to the original website pages. No more broken S3 links.

### 7. Website Content Prioritization (Completed: 2025-12-19)
**Problem:** Chatbot was retrieving and using outdated information from PDF documents instead of current website content, leading to inaccurate answers.

**Solution:**
- Switched from pure semantic search to hybrid search (semantic + BM25 keyword matching)
- Increased retrieval results from 40 to 60 to ensure relevant content is found
- Added source type detection to identify website vs PDF sources
- Implemented reranking logic that always places website sources before PDF sources in results
- Added date extraction and filtering to prioritize sources with future dates for date-related queries
- Enhanced prompt with date-aware response guidance

**Impact:** 
- Hybrid search combines meaning-based and keyword-based matching for better accuracy
- Increased result count ensures relevant pages aren't missed
- Website content consistently appears before PDFs in search results
- Date-related queries focus on upcoming events rather than past dates
- Simplified architecture - no complex query preprocessing needed

**Note:** Currently using Claude 3.5 Sonnet V2. Claude 3.7 requires inference profiles which are not yet configured.

## Known Limitations

### Web Crawler Knowledge Base
**Current Status (as of 2026-02-10):** The chatbot uses AWS Bedrock's managed web crawler for content ingestion.

**Configuration:**
- Knowledge Base ID: `GCERPWLGOK`
- Chunking: Semantic, 300 tokens, buffer 1 (20% overlap), breakpoint 95%
- Indexed: ~2,806 pages from all school subdomains
- Data Source: AWS Bedrock Web Crawler (managed service)

**Known Issues:**
1. **External PDFs Not Captured**: PDFs hosted on external domains (e.g., ParentSquare) are blocked by robots.txt and cannot be crawled.

2. **JavaScript-Rendered Content**: Pages that rely heavily on JavaScript may not be fully indexed if content loads after initial page render.

### Search Quality
- Hybrid search (semantic + keyword) provides good results for most queries
- Selective reranking (disabled for main domain, enabled for school-specific queries) optimizes retrieval
- Larger chunk sizes (300 tokens) with overlap (buffer 1) balance context and precision

**Content Management:**
- IT team can trigger re-crawls through AWS Bedrock console
- No custom scraper code to maintain
- Automatic respect for robots.txt rules

## Operations Guide for IT Team

### Daily Operations

**No daily maintenance required** - the system runs automatically.

### Content Updates

**When website content changes, trigger a Knowledge Base sync:**

1. Go to AWS Console → Bedrock → Knowledge Bases
2. Select Knowledge Base: `OrcuttSchoolsKB` (ID: GCERPWLGOK)
3. Click on the Data Source (Web Crawler)
4. Click "Sync" button
5. Monitor sync progress (typically 2-3 hours for full site)

**Sync frequency recommendations:**
- After major website updates: Immediately
- Routine updates: Weekly or bi-weekly
- Before important events: Day before

**Note:** The web crawler is a managed AWS service - no custom scraper code to maintain.

### Monitoring

**Check chatbot health:**
1. Visit: https://orcutt-ai.techreformers.com (or https://d3dolln1x7yei7.cloudfront.net)
2. Test with standard queries:
   - "Who is the Superintendent?"
   - "What are the school hours?"
   - "How do I enroll my child?"

**Check logs for errors:**
1. AWS Console → CloudWatch → Log Groups
2. Select: `/aws/lambda/OrcuttChatbotStack-dev-ChatbotLambda...`
3. Look for ERROR messages in recent logs

**Monitor costs:**
1. AWS Console → Cost Explorer
2. Filter by service: Bedrock, Lambda, CloudFront
3. Set up billing alerts if needed

### Conversation History and Analytics

**DynamoDB stores all chatbot conversations** for analytics and quality monitoring.

**Table Name:** `orcutt-conversations-785054116835`

**What's Stored:**
- Every user message and chatbot response
- Session IDs (tracks individual conversations)
- Timestamps
- Query types (greeting, farewell, knowledge_base)
- Response times
- Sources used for each response
- User feedback (if thumbs up/down feature is enabled)

**How to Use This Data:**

1. **View Conversations:**
   - AWS Console → DynamoDB → Tables → `orcutt-conversations-785054116835`
   - Click "Explore table items" to browse conversations
   - Search by session_id to see full conversation threads

2. **Analytics and Insights:**
   - Identify most common questions users ask
   - Find topics that need better content on the website
   - Track usage patterns (peak times, popular queries)
   - Monitor response times and performance

3. **Quality Monitoring:**
   - Review conversations to find incorrect answers
   - Identify queries that need better responses
   - See which sources are being used most frequently
   - Find gaps in the Knowledge Base content

4. **Export Data for Analysis:**
   - AWS Console → DynamoDB → Tables → Export to S3
   - Export to CSV/JSON for spreadsheet analysis
   - Use for monthly reports or trend analysis

5. **Debugging User Issues:**
   - When users report problems, look up their session_id
   - See exactly what they asked and what the chatbot responded
   - Check response times and error types

**Cost:** DynamoDB charges per read/write operation, but typically costs less than $5/month for this use case.

**Data Retention:** Conversations are stored indefinitely unless manually deleted. Consider setting up a lifecycle policy if you want to automatically delete old conversations after a certain period.

### Troubleshooting

**Chatbot not responding:**
1. Check Lambda function is running (AWS Console → Lambda)
2. Check API Gateway endpoint is accessible
3. Check CloudWatch logs for errors

**Incorrect or outdated answers:**
1. Verify Knowledge Base sync completed successfully
2. Check if the correct page exists on the website
3. Manually test retrieval in Bedrock console

**High costs:**
1. Check CloudWatch for unusual traffic patterns
2. Verify no infinite loops in Lambda functions
3. Review Bedrock usage metrics

### Making Code Changes

**Prerequisites:**
- Git installed
- AWS CLI configured with SSO profile `orcutt-ai`
- Python 3.13+ with virtual environment
- Node.js 20+ (for CDK)
- Docker Desktop running

**Deployment process:**
```bash
# 1. Clone repository
git clone https://github.com/Tech-Reformers/orcutt-schools-chatbot-tr
cd orcutt-schools-chatbot-tr/orcutt-chatbot

# 2. Activate virtual environment
source venv/bin/activate

# 3. Login to AWS
aws sso login --profile orcutt-ai

# 4. Make your code changes
# Edit files in lambda/chatbot/ or infrastructure/

# 5. Deploy changes
export CDK_DEFAULT_ACCOUNT=785054116835
export CDK_DEFAULT_REGION=us-west-2
cdk deploy --profile orcutt-ai --require-approval never

# 6. Test changes
# Visit https://orcutt-ai.techreformers.com

# 7. Commit to git
git add -A
git commit -m "Description of changes"
git push
```

### Key Configuration Files

**Lambda Function:**
- Code: `orcutt-chatbot/lambda/chatbot/lambda_function.py`
- Environment variables set in: `orcutt-chatbot/infrastructure/orcutt_chatbot_stack.py` (line ~365)

**Frontend:**
- Code: `orcutt-chatbot/frontend/src/`
- Build: `orcutt-chatbot/frontend/build/`

**Infrastructure:**
- CDK Stack: `orcutt-chatbot/infrastructure/orcutt_chatbot_stack.py`
- Config: `orcutt-chatbot/config.yaml`

### AWS Resources

**Key Resources:**
- Custom Domain: https://orcutt-ai.techreformers.com
- CloudFront Distribution: https://d3dolln1x7yei7.cloudfront.net
- API Gateway: https://4rm7hu9b29.execute-api.us-west-2.amazonaws.com/prod/
- Lambda Function: `OrcuttChatbotStack-dev-ChatbotLambda4595A29D-MISNPrFfoYqr`
- Knowledge Base: `GCERPWLGOK` (OrcuttSchoolsKB with Web Crawler)
- DynamoDB Table: `orcutt-conversations-785054116835`
- S3 Bucket: `orcutt-chatbot-kb-dev-785054116835-us-west-2`

**AWS Account:**
- Account ID: 785054116835
- Region: us-west-2
- SSO Profile: orcutt-ai

### Emergency Contacts

**For technical issues:**
- Tech Reformers: info@techreformers.com
- GitHub Issues: https://github.com/Tech-Reformers/orcutt-schools-chatbot-tr/issues

**For AWS support:**
- AWS Support Console (if you have a support plan)
- Or contact Tech Reformers for assistance

## Support

For any queries or issues with this fork, please contact:

**Tech Reformers:** <info@techreformers.com>

For questions about the original project, please refer to the [original repository](https://github.com/cal-poly-dxhub/orcutt-schools-chatbot/tree/main) or contact:
- Darren Kraker - <dkraker@amazon.com>
- Shrey Shah - <sshah84@calpoly.edu>
