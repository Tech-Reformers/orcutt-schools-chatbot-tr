# AWS Case Study: Orcutt Union School District AI Chatbot

## Executive Summary

Tech Reformers, an AWS Advanced Partner pursuing Generative AI designation, developed an intelligent chatbot for Orcutt Union School District using AWS's comprehensive AI/ML services. The solution demonstrates expertise in Amazon Bedrock, Knowledge Bases, and serverless architecture to deliver a production-ready conversational AI system.

## Customer Challenge

Orcutt Union School District needed to provide 24/7 access to information for students, parents, and staff across 12 schools. Key challenges included:

- **Information Accessibility**: District website contains 2,800+ pages across multiple school subdomains
- **Response Time**: Staff couldn't respond to routine inquiries outside business hours
- **Consistency**: Information needed to be accurate and up-to-date across all schools
- **Scale**: Solution needed to handle inquiries from entire district community
- **Maintenance**: IT team required self-service content management without developer involvement

## Solution Architecture

### AWS Services Utilized

**Generative AI & Machine Learning:**
- **Amazon Bedrock** - Foundation model hosting and inference
  - Claude 3.5 Sonnet V2 for response generation
  - Amazon Nova Lite for query classification
  - Titan Text Embeddings V2 for semantic search
- **Amazon Bedrock Knowledge Bases** - Managed RAG (Retrieval Augmented Generation)
  - Web crawler for automated content ingestion
  - Semantic chunking with 500-token chunks and 20% overlap
  - Hybrid search (semantic + BM25 keyword matching)
  - OpenSearch Serverless for vector storage

**Compute & API:**
- **AWS Lambda** - Serverless compute for chatbot logic (Python 3.13)
- **Amazon API Gateway** - RESTful API endpoints with CORS support

**Storage & Data:**
- **Amazon DynamoDB** - Conversation history and session management
- **Amazon S3** - Document storage and frontend hosting
- **Amazon CloudFront** - Global content delivery for React frontend

**Monitoring & Operations:**
- **Amazon CloudWatch** - Logging, monitoring, and alerting
- **AWS IAM** - Fine-grained access control and service permissions

**Infrastructure as Code:**
- **AWS CDK** - Python-based infrastructure deployment
- **AWS CloudFormation** - Automated resource provisioning

### Architecture Highlights

1. **Serverless & Scalable**: Fully serverless architecture scales automatically with demand
2. **Managed AI Services**: Leverages Bedrock's managed services for reduced operational overhead
3. **Multi-Model Strategy**: Uses different models for different tasks (classification vs generation)
4. **RAG Implementation**: Knowledge Bases provide accurate, source-cited responses
5. **Cost-Optimized**: Pay-per-use pricing model with no idle infrastructure costs

## Technical Implementation

### Generative AI Capabilities

**1. Intelligent Query Classification**
- Amazon Nova Lite classifies user intent (greeting, farewell, knowledge_base, school-specific)
- Enables context-aware routing and response generation
- Cost-effective classification before expensive generation

**2. Retrieval Augmented Generation (RAG)**
- Bedrock Knowledge Bases with web crawler ingests 2,800+ pages automatically
- Hybrid search combines semantic understanding with keyword matching
- Retrieves top 60 relevant chunks with source attribution
- Claude generates responses grounded in retrieved context

**3. Advanced Prompt Engineering**
- Date-aware responses prioritize upcoming events
- Source prioritization (website content over archived PDFs)
- Conflict resolution (district services take precedence over general policies)
- Few-shot examples for handling edge cases

**4. Multi-School Context Management**
- Domain filtering enables school-specific queries
- Users select school in UI, ask questions without repeating school name
- "When does school start?" returns Pine Grove-specific info when Pine Grove is selected

### Key Technical Decisions

**Web Crawler vs Custom Scraper:**
- Migrated from custom Lambda scraper to Bedrock's managed web crawler
- Benefits: Automatic JavaScript rendering, managed retries, no maintenance
- Simplified architecture from 900 LOC custom scraper to zero-code managed service

**Chunking Strategy:**
- Semantic chunking with 500 tokens (vs fixed-size 300 tokens)
- 20% overlap (buffer=1) ensures context isn't lost at boundaries
- 95th percentile breakpoint for natural semantic boundaries

**Hybrid Search:**
- Combines semantic similarity (vector search) with BM25 keyword matching
- Addresses limitation where pure semantic search missed keyword-heavy queries
- Example: "pizza party" now finds pages with exact keyword matches

## Results & Impact

### Quantitative Results
- **2,800+ pages** indexed automatically across 12 school subdomains
- **~20 second** average response time including retrieval and generation
- **60 context chunks** retrieved per query for comprehensive answers
- **Zero maintenance** required for content updates (self-service sync)

### Qualitative Benefits
- **24/7 Availability**: Parents and students get instant answers anytime
- **Consistent Information**: All responses grounded in official district content
- **Source Attribution**: Every answer includes clickable links to source pages
- **IT Self-Service**: District IT can trigger content syncs without developer involvement
- **Scalable**: Handles unlimited concurrent users with serverless architecture

### AWS Service Expertise Demonstrated

**Amazon Bedrock:**
- Multi-model orchestration (Claude, Nova, Titan)
- Knowledge Bases with web crawler configuration
- Prompt engineering and few-shot learning
- Guardrails for content safety (planned)

**Serverless Architecture:**
- Lambda function optimization (timeout, memory, layers)
- API Gateway with CORS and multiple endpoints
- DynamoDB for session management
- S3 + CloudFront for global frontend delivery

**Infrastructure as Code:**
- AWS CDK for Python-based infrastructure
- Custom resources for OpenSearch index creation
- Environment-based deployments (dev/prod)
- Automated CI/CD pipeline

## Lessons Learned

### Technical Insights

1. **Chunking Matters**: Increasing chunk size from 300 to 500 tokens significantly improved context quality
2. **Hybrid Search is Essential**: Pure semantic search missed keyword-heavy queries; hybrid search solved this
3. **Managed Services Win**: Web crawler eliminated 900 LOC and ongoing maintenance burden
4. **Source Prioritization**: Explicit prompt instructions needed to prioritize current website content over archived PDFs

### Operational Insights

1. **Customer Self-Service**: Enabling IT team to manage content syncs reduced dependency on developers
2. **Monitoring is Critical**: CloudWatch logs essential for debugging retrieval quality issues
3. **Iterative Improvement**: Few-shot examples in prompts allow rapid iteration on edge cases
4. **Cost Transparency**: Bedrock's pay-per-token model provides clear cost attribution

## Future Enhancements

1. **Multimodal Support**: Add image understanding for diagrams and infographics
2. **Guardrails**: Implement Bedrock Guardrails for content filtering
3. **Analytics Dashboard**: Track popular queries and identify content gaps
4. **Feedback Loop**: Use thumbs up/down to improve retrieval and responses
5. **Multi-Language**: Support Spanish for bilingual community

## AWS Partner Value Proposition

This case study demonstrates Tech Reformers' expertise in:

- **Generative AI**: Production RAG implementation with prompt engineering
- **AWS Bedrock**: Multi-model orchestration and Knowledge Bases
- **Serverless Architecture**: Cost-optimized, scalable infrastructure
- **Customer Success**: Self-service operations for non-technical teams
- **Best Practices**: IaC, monitoring, security, and documentation

## Technical Specifications

**AWS Services:**
- Amazon Bedrock (Claude 3.5 Sonnet V2, Nova Lite, Titan Embeddings V2)
- Amazon Bedrock Knowledge Bases with Web Crawler
- AWS Lambda (Python 3.13)
- Amazon API Gateway
- Amazon DynamoDB
- Amazon S3
- Amazon CloudFront
- Amazon CloudWatch
- AWS IAM
- AWS CDK / CloudFormation

**Deployment:**
- Region: us-west-2
- Account: 785054116835
- Infrastructure: AWS CDK (Python)
- Frontend: React 18 + Tailwind CSS
- Backend: Python 3.13 Lambda functions

**Repository:**
- GitHub: https://github.com/Tech-Reformers/orcutt-schools-chatbot-tr
- License: Open Source
- Documentation: Comprehensive README and operations guide

## Contact Information

**Tech Reformers**
- Email: info@techreformers.com
- AWS Partner: Advanced Tier
- Pursuing: Generative AI Competency

**Customer**
- Orcutt Union School District
- Location: Orcutt, California
- Website: https://www.orcuttschools.net
- Live Chatbot: https://d3dolln1x7yei7.cloudfront.net

---

*This case study demonstrates production-ready Generative AI implementation using AWS Bedrock and serverless architecture, showcasing Tech Reformers' expertise as an AWS Advanced Partner.*
