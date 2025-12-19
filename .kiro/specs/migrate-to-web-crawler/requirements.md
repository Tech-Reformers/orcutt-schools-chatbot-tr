# Requirements Document

## Introduction

This spec addresses the migration from a custom webscraper to AWS Bedrock's built-in web crawler for knowledge base content ingestion. The current custom webscraper has limitations in capturing external PDFs, JavaScript-rendered content, and requires ongoing maintenance. Bedrock's web crawler provides a managed solution that handles these scenarios automatically while simplifying the architecture.

## Glossary

- **Web Crawler**: AWS Bedrock's managed service that crawls websites and indexes content into a knowledge base
- **Custom Webscraper**: The current Lambda function that manually scrapes content and uploads to S3
- **Knowledge Base**: AWS Bedrock Knowledge Base that stores and indexes content for retrieval
- **Seed URL**: The starting URL(s) from which the web crawler begins crawling
- **Crawl Scope**: Configuration that defines which URLs the crawler should follow
- **Source URI**: The standard Bedrock metadata field containing the original URL of crawled content
- **Domain Filtering**: The ability to retrieve content from specific domains or subdomains

## Requirements

### Requirement 1

**User Story:** As a system administrator, I want to use AWS Bedrock's managed web crawler, so that I don't have to maintain custom scraping code.

#### Acceptance Criteria

1. WHEN the system ingests content THEN the system SHALL use Bedrock's web crawler instead of the custom Lambda webscraper
2. WHEN content needs to be updated THEN the system SHALL trigger a web crawler sync job instead of running the custom scraper
3. WHEN the web crawler completes THEN the system SHALL have all website content indexed in the knowledge base

### Requirement 2

**User Story:** As a user, I want the chatbot to access all content from the website including external PDFs, so that I receive complete and accurate information.

#### Acceptance Criteria

1. WHEN the web crawler encounters external PDF links THEN the system SHALL follow and index those PDFs from allowed domains
2. WHEN the web crawler indexes external PDFs THEN the system SHALL preserve the source URL in metadata
3. WHEN the web crawler encounters JavaScript-rendered content THEN the system SHALL execute JavaScript and capture the rendered content

### Requirement 3

**User Story:** As a user asking school-specific questions, I want to select a school from the dropdown and ask questions without repeating the school name, so that I can get relevant information for that school.

#### Acceptance Criteria

1. WHEN a user selects a specific school from the UI dropdown THEN the system SHALL automatically filter knowledge base results to that school's subdomain
2. WHEN a school is selected and the user asks "When does school start?" THEN the system SHALL return information specific to the selected school without requiring the school name in the query
3. WHEN filtering by school domain THEN the system SHALL use the source URI metadata field with a startsWith filter
4. WHEN no school is selected THEN the system SHALL search across all domains

### Requirement 4

**User Story:** As a developer, I want the code to work with Bedrock's standard metadata format, so that the system is maintainable and follows AWS best practices.

#### Acceptance Criteria

1. WHEN processing retrieval results THEN the system SHALL extract source URLs from the x-amz-bedrock-kb-source-uri metadata field
2. WHEN filtering by domain THEN the system SHALL use startsWith filter on the source URI field
3. WHEN displaying sources THEN the system SHALL extract domain information from the source URI

### Requirement 5

**User Story:** As a system administrator, I want to configure which domains and content types the crawler should index, so that only relevant content is captured.

#### Acceptance Criteria

1. WHEN configuring the web crawler THEN the system SHALL specify seed URLs for all school subdomains
2. WHEN configuring the web crawler THEN the system SHALL define inclusion patterns for allowed external domains
3. WHEN configuring the web crawler THEN the system SHALL set appropriate crawl depth and scope

### Requirement 6

**User Story:** As a user, I want the chatbot to find information that was previously missed by the custom scraper, so that I receive accurate answers to all questions.

#### Acceptance Criteria

1. WHEN the web crawler indexes content THEN the system SHALL capture staff directory information including all paginated entries
2. WHEN the web crawler indexes content THEN the system SHALL capture external PDFs linked from the website
3. WHEN the web crawler indexes content THEN the system SHALL capture JavaScript-rendered content

### Requirement 7

**User Story:** As a developer, I want to remove the custom webscraper code, so that the codebase is simpler and easier to maintain.

#### Acceptance Criteria

1. WHEN the migration is complete THEN the system SHALL no longer use the custom webscraper Lambda function
2. WHEN the migration is complete THEN the system SHALL no longer require the webscraper layer or dependencies
3. WHEN the migration is complete THEN the system SHALL document the web crawler configuration for future reference

### Requirement 8

**User Story:** As a customer administrator, I want to manage the web crawler through the AWS Console, so that I can update content and adjust settings without requiring developer assistance.

#### Acceptance Criteria

1. WHEN the customer needs to update content THEN the customer SHALL be able to trigger a web crawler sync job through the AWS Console
2. WHEN the customer needs to adjust crawler settings THEN the customer SHALL be able to modify inclusion/exclusion filters through the AWS Console
3. WHEN the migration is complete THEN the customer SHALL have documentation explaining how to manage the web crawler
4. WHEN the customer manages the crawler THEN the customer SHALL be able to monitor crawl job status and view logs
