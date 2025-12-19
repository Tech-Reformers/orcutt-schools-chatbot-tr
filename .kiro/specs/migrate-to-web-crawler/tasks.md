# Implementation Plan

- [ ] 1. Create new Knowledge Base with web crawler
  - Go to AWS Bedrock Console → Knowledge Bases → Create
  - Name: "OrcuttSchoolsKB-WebCrawler"
  - Choose "Web Crawler" as data source
  - Configure embedding model: amazon.titan-embed-text-v2:0
  - _Requirements: 1.1_

- [ ] 2. Configure web crawler seed URLs
  - Add all 12 school domain URLs as seed URLs
  - Main: https://www.orcuttschools.net
  - Schools: orcuttacademy, oahs, lakeview, ojhs, aliceshaw, joenightingale, olgareed, pattersonroad, pinegrove, ralphdunlap, osis
  - _Requirements: 5.1_

- [ ] 3. Configure crawl scope and filters
  - Set crawl scope to "Host crawling"
  - Enable subdomain crawling
  - Set crawl depth to 5
  - Add inclusion filter: https://*.orcuttschools.net/*
  - Add inclusion filter for external PDFs: https://files.smartsites.parentsquare.com/*.pdf
  - Add exclusion filters if needed (calendar events, search pages)
  - _Requirements: 5.2, 5.3_

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
  - Select "Pine Grove Elementary" in UI
  - Test: "When does school start?"
  - Verify results are filtered to Pine Grove domain
  - Test with other schools
  - Verify filtering works for all schools
  - _Requirements: 3.1, 3.2, 3.3_

- [ ] 9. Verify external content capture
  - Check if bus schedules (external PDFs) are indexed
  - Test query: "What are the bus routes?"
  - Verify staff directory is complete
  - Test query: "Who is [specific staff member name]?"
  - _Requirements: 2.1, 2.2, 6.1, 6.2_

- [ ] 10. Document web crawler configuration
  - Update README with web crawler setup instructions
  - Document seed URLs and filters used
  - Document how to trigger manual sync
  - Document how to monitor crawl jobs
  - _Requirements: 7.3_

- [ ] 11. Remove custom webscraper (optional, after validation)
  - Remove webscraper Lambda function from CDK stack
  - Remove webscraper layer
  - Remove scripts/run_webscraper.sh
  - Update documentation
  - _Requirements: 7.1, 7.2_

- [ ] 12. Final validation and comparison
  - Compare answer quality with old KB
  - Test all critical user queries
  - Verify no regressions
  - Document any improvements or issues found
  - _Requirements: All_
