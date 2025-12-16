# Requirements Document

## Introduction

This spec addresses two critical webscraper limitations:
1. **External PDFs**: The webscraper cannot download PDFs hosted on external domains (like ParentSquare/SmartSites) that are linked from the main website, missing important documents like bus schedules and forms.
2. **JavaScript/Dynamic Content**: The webscraper cannot capture content that loads dynamically via JavaScript, including paginated staff directories and AJAX-loaded content.

## Glossary

- **Webscraper**: The Lambda function that crawls the Orcutt Schools website and downloads content to S3
- **External PDF**: A PDF document hosted on a different domain (e.g., files.smartsites.parentsquare.com) but linked from the main website
- **Same-Domain Content**: Content hosted directly on orcuttschools.net or its subdomains
- **Linked Resource**: A file or document that is referenced via a hyperlink on the main website
- **Allowed External Domain**: A trusted external domain from which PDFs should be downloaded (e.g., ParentSquare, SmartSites)

## Requirements

### Requirement 1

**User Story:** As a user, I want to access information from all documents linked on the website, so that I can get complete answers to my questions.

#### Acceptance Criteria

1. WHEN the webscraper encounters a link to a PDF on an allowed external domain THEN the system SHALL download that PDF
2. WHEN the webscraper downloads an external PDF THEN the system SHALL store it in S3 with appropriate metadata indicating the source URL
3. WHEN the webscraper processes a page with external PDF links THEN the system SHALL follow those links and download the PDFs

### Requirement 2

**User Story:** As a system administrator, I want to control which external domains are trusted for PDF downloads, so that the system doesn't download malicious or irrelevant content.

#### Acceptance Criteria

1. WHEN the webscraper is configured THEN the system SHALL maintain a whitelist of allowed external domains
2. WHEN the webscraper encounters a PDF link THEN the system SHALL check if the domain is in the allowed list before downloading
3. WHEN an external PDF is from a non-whitelisted domain THEN the system SHALL skip that PDF and log the skipped URL

### Requirement 3

**User Story:** As a user, I want external PDFs to be treated as authoritative sources, so that I receive accurate information from official district documents.

#### Acceptance Criteria

1. WHEN an external PDF is downloaded THEN the system SHALL create metadata indicating it is an official district document
2. WHEN an external PDF is stored THEN the system SHALL preserve the original URL in metadata for source attribution
3. WHEN an external PDF is processed THEN the system SHALL extract and store the same metadata as same-domain PDFs (meeting dates, document type, etc.)

### Requirement 4

**User Story:** As a system administrator, I want the webscraper to handle external PDF downloads gracefully, so that failures don't break the entire scraping process.

#### Acceptance Criteria

1. WHEN an external PDF download fails THEN the system SHALL log the error and continue processing other links
2. WHEN an external PDF is too large THEN the system SHALL skip it and log a warning
3. WHEN an external PDF URL is invalid or unreachable THEN the system SHALL handle the error without crashing the scraper

### Requirement 5

**User Story:** As a user, I want the webscraper to find PDFs linked from the website regardless of how they are linked, so that no important documents are missed.

#### Acceptance Criteria

1. WHEN a PDF is linked via a direct href THEN the system SHALL download it
2. WHEN a PDF is linked via a button or download link THEN the system SHALL download it
3. WHEN a PDF is embedded or referenced in JavaScript THEN the system SHALL attempt to extract and download it

### Requirement 6

**User Story:** As a user, I want the webscraper to capture all staff directory information, so that I can ask about any staff member by name.

#### Acceptance Criteria

1. WHEN the webscraper encounters a paginated directory THEN the system SHALL navigate through all pages to capture complete content
2. WHEN content loads dynamically via JavaScript THEN the system SHALL execute JavaScript to render the full page before scraping
3. WHEN the webscraper captures a directory page THEN the system SHALL include all staff names and contact information in the scraped content

### Requirement 7

**User Story:** As a system administrator, I want the webscraper to handle JavaScript-rendered content, so that modern web pages are fully captured.

#### Acceptance Criteria

1. WHEN the webscraper encounters a page with JavaScript content THEN the system SHALL use a headless browser to render the page
2. WHEN JavaScript content takes time to load THEN the system SHALL wait for content to be fully rendered before scraping
3. WHEN JavaScript rendering fails THEN the system SHALL fall back to static HTML scraping and log the issue
