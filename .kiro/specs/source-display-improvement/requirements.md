# Requirements Document

## Introduction

This spec addresses improving how source citations are displayed in the chatbot interface. Currently, sources show S3 URLs to scraped content (which return Access Denied errors) alongside small link icons to the original websites. Users should see clear, clickable links to the original website sources instead.

## Glossary

- **Source Citation**: A reference to where the chatbot retrieved information from
- **S3 Presigned URL**: A temporary URL to access scraped content stored in AWS S3
- **Original Website URL**: The actual web page URL where content was originally published
- **Sources Sidebar**: The right-side panel showing "Sources Used" for the current response

## Requirements

### Requirement 1

**User Story:** As a user, I want to click on source citations and be taken to the original website, so that I can verify information and learn more.

#### Acceptance Criteria

1. WHEN a source has an original website URL THEN the system SHALL display that URL as a clickable hyperlink
2. WHEN a user clicks on a source citation THEN the system SHALL open the original website URL in a new browser tab
3. WHEN a source citation is displayed THEN the system SHALL show a link icon (🔗) followed by the clickable URL text
4. WHEN a source has no original website URL (PDF documents) THEN the system SHALL display the filename only without a clickable link

### Requirement 2

**User Story:** As a user, I want source citations to be clearly readable and accessible, so that I can easily navigate to referenced materials.

#### Acceptance Criteria

1. WHEN viewing the Sources sidebar THEN the system SHALL display each source on its own line
2. WHEN a source URL is displayed THEN the system SHALL format it as a standard hyperlink with appropriate styling
3. WHEN hovering over a source link THEN the system SHALL provide visual feedback (cursor change, underline, etc.)

### Requirement 3

**User Story:** As a user, I want to avoid encountering broken or inaccessible links, so that my experience is smooth and frustration-free.

#### Acceptance Criteria

1. WHEN the backend processes sources THEN the system SHALL NOT generate S3 presigned URLs for any content
2. WHEN the backend processes sources THEN the system SHALL use original website URLs for all sources (including PDFs)
3. WHEN a source has an original URL in metadata THEN the system SHALL always use that URL instead of S3 storage locations
