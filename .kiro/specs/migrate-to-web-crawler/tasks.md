# Implementation Plan

- [x] 1. Create new Knowledge Base - Step 1: Provide details
  - Go to AWS Bedrock Console → Knowledge Bases → Create
  - Knowledge Base name: "OrcuttSchoolsKB-WebCrawler"
  - Knowledge Base description: (optional)
  - Service role: AmazonBedrockExecutionRoleForKnowledgeBase_kwns0 (or auto-create)
  - Knowledge base type: Knowledge base use vector store
  - Data source type: Web Crawler
  - _Requirements: 1.1_

- [ ] 2. Configure data source - Step 2: Configure data source
  - Data source name: (auto-generated or custom)
  - Source URL: https://www.orcuttschools.net
  - Sync scope: Subdomains (automatically discovers all 12 school subdomains)
  - Authentication: No Authentication
  - Regex include patterns: None (leave empty - patterns cause crawl failures)
  - Regex exclude patterns: None (leave empty - patterns cause crawl failures)
  - Note: Subdomains scope automatically crawls all *.orcuttschools.net domains
  - Note: ParentSquare PDFs will fail due to robots.txt restrictions (expected)
  - _Requirements: 5.1, 5.2_

- [ ] 3. Configure data storage and processing - Step 3
  - Parsing strategy: Amazon Bedrock Default Parser
  - Chunking strategy: Semantic chunking
    - Buffer: 1
    - Max tokens: 300
    - Breakpoint percentile threshold: 95
  - Embeddings model: Titan Text Embeddings v2
    - Embedding type: Float vector embeddings
    - Vector dimensions: 1024
  - Vector store: Quick create vector store - recommended
    - Type: Amazon OpenSearch Serverless
  - _Requirements: 5.3_

- [ ] 4. Run initial web crawler sync
  - Start the crawl job
  - Monitor progress in AWS Console
  - Wait for completion (may take 30-60 minutes)
  - Verify content is indexed
  - _Requirements: 1.3_

- [ ] 5. Update Lambda code for web crawler metadata
  - Modify `query_knowledge_base_semantic()` to use startsWith filter on x-amz-bedrock-kb-source-uri
  - Update `process_knowledge_base_response()` to extract source URI from standard metadata
  - Extract domain from source URI for school name mapping
  - Handle missing metadata fields gracefully
  - _Requirements: 4.1, 4.2, 4.3_

- [ ] 6. Update environment variable
  - Get new Knowledge Base ID from AWS Console
  - Update Lambda environment variable KNOWLEDGE_BASE_ID to new KB
  - Or update config.yaml if making permanent
  - _Requirements: 1.1_

- [ ] 7. Deploy and test basic queries
  - Deploy Lambda changes
  - Test: "Who is the Superintendent?"
  - Test: "Who are the Executive Directors?"
  - Test: "How do I sign up for a classroom pizza party?"
  - Test: "pizza"
  - Verify all queries return correct answers
  - _Requirements: 6.1, 6.2, 6.3_

- [ ] 8. Test school-specific filtering
  - Select "Pine Grove Elementary" in UI dropdown
  - Test: "When does school start?" (without mentioning Pine Grove in query)
  - Verify results are filtered to Pine Grove domain only
  - Verify answer is specific to Pine Grove
  - Test with other schools (Lakeview, Alice Shaw, etc.)
  - Verify filtering works for all schools
  - Test with no school selected - verify searches all domains
  - _Requirements: 3.1, 3.2, 3.3, 3.4_

- [ ] 9. Verify external content capture
  - Check if bus schedules (external PDFs) are indexed
  - Test query: "What are the bus routes?"
  - Verify staff directory is complete
  - Test query: "Who is [specific staff member name]?"
  - _Requirements: 2.1, 2.2, 6.1, 6.2_

- [ ] 10. Create customer documentation for web crawler management
  - Create step-by-step guide for triggering manual sync jobs
  - Document how to view crawl job status and logs in AWS Console
  - Document how to modify inclusion/exclusion filters
  - Document how to add new seed URLs if needed
  - Include screenshots of AWS Console navigation
  - Document troubleshooting common issues
  - _Requirements: 8.1, 8.2, 8.3, 8.4_

- [ ] 11. Update README with web crawler information
  - Update README with web crawler setup instructions
  - Document seed URLs and filters used
  - Link to customer management documentation
  - Note that custom webscraper is deprecated
  - _Requirements: 7.3_

- [ ] 12. Remove custom webscraper (optional, after validation)
  - Remove webscraper Lambda function from CDK stack
  - Remove webscraper layer
  - Remove scripts/run_webscraper.sh
  - Update documentation
  - _Requirements: 7.1, 7.2_

- [ ] 13. Final validation and comparison
  - Compare answer quality with old KB
  - Test all critical user queries
  - Verify no regressions
  - Document any improvements or issues found
  - _Requirements: All_
