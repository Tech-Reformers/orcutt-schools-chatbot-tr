# Requirements Document

## Introduction

This spec addresses a critical accuracy issue where the chatbot retrieves and uses information from outdated PDF documents instead of current website content. When users ask questions like "Who are the Executive Directors?", the chatbot should use the current staff directory from the website, not old board minutes or archived PDFs.

## Glossary

- **Website Content**: Current information scraped from orcuttschools.net domain pages (HTML content)
- **PDF Content**: Archived documents like board minutes, policies, and historical records stored as PDFs
- **Knowledge Base Retrieval**: The process of searching and ranking relevant content from the knowledge base
- **Semantic Search**: Vector-based search that finds content based on meaning similarity
- **Source Ranking**: The order in which retrieved sources are presented to the AI for response generation

## Requirements

### Requirement 1

**User Story:** As a user, I want to receive current, accurate information from the district website, so that I can make decisions based on up-to-date facts.

#### Acceptance Criteria

1. WHEN a user asks a question about current staff, programs, or policies THEN the system SHALL prioritize website content over PDF documents
2. WHEN both website content and PDF content are available for a query THEN the system SHALL rank website sources higher than PDF sources
3. WHEN the system retrieves sources from the knowledge base THEN the system SHALL apply a boost factor to website sources to increase their relevance scores

### Requirement 2

**User Story:** As a user, I want consistent answers regardless of how I phrase my question, so that I can trust the information provided.

#### Acceptance Criteria

1. WHEN a user asks the same question with different wording THEN the system SHALL retrieve the same source types (website vs PDF) based on content relevance, not phrasing
2. WHEN website content exists for a topic THEN the system SHALL prioritize that content in retrieval results
3. WHEN the system retrieves sources THEN the system SHALL apply consistent ranking logic that favors website sources over PDF sources

### Requirement 3

**User Story:** As a user, I want the chatbot to find information that is clearly available on the website, so that I receive accurate answers to basic questions.

#### Acceptance Criteria

1. WHEN a user asks about staff members listed on the website THEN the system SHALL retrieve and use that staff directory information
2. WHEN a user asks about current programs or services THEN the system SHALL retrieve information from the relevant website pages
3. WHEN the system retrieves multiple sources THEN the system SHALL rank website sources higher than PDF sources in the results

### Requirement 4

**User Story:** As a user, I want to receive information from the most relevant sources, so that my questions are answered accurately.

#### Acceptance Criteria

1. WHEN both website and PDF sources contain relevant information THEN the system SHALL prefer website sources for the response
2. WHEN only PDF sources contain the needed information THEN the system SHALL use those PDF sources
3. WHEN website sources are insufficient THEN the system SHALL supplement with PDF sources as needed

### Requirement 5

**User Story:** As a user asking about upcoming events or schedules, I want to see future dates rather than past dates, so that the information is relevant to my needs.

#### Acceptance Criteria

1. WHEN a user asks about events, schedules, or dates THEN the system SHALL use the current date to filter and prioritize relevant information
2. WHEN a source contains both past and future dates THEN the system SHALL use that source and focus on the future dates in the response
3. WHEN a source contains only past dates THEN the system SHALL rank it lower than sources with future or current dates
4. WHEN generating a response about dates THEN the system SHALL reference the current date to provide context (e.g., "The next parent-teacher conferences are...")

### Requirement 6

**User Story:** As a user asking about specific terms or phrases, I want the system to find pages containing those exact words, so that I receive accurate answers to direct questions.

#### Acceptance Criteria

1. WHEN a user asks a question containing specific terms or phrases THEN the system SHALL retrieve sources that contain those exact terms
2. WHEN a source contains exact keyword matches for the query THEN the system SHALL rank that source higher than sources with only semantic similarity
3. WHEN the system performs knowledge base retrieval THEN the system SHALL use hybrid search combining semantic similarity and keyword matching
