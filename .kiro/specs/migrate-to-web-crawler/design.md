# Design Document

## Overview

This design migrates from a custom Lambda-based webscraper to AWS Bedrock's managed web crawler service. The web crawler provides automatic content ingestion, handles JavaScript rendering, follows external links, and eliminates the need for custom scraping code. The migration involves updating the Lambda code to work with Bedrock's standard metadata format and configuring the web crawler with appropriate scope and filters.

## Architecture

**Current Architecture:**
1. Custom Lambda webscraper → Scrapes website → Uploads to S3 → Bedrock indexes from S3
2. Custom metadata: `domain`, `source`, `meeting_date`
3. Manual trigger via scripts/run_webscraper.sh

**New Architecture:**
1. Bedrock Web Crawler → Crawls website directly → Indexes into Knowledge Base
2. Standard metadata: `x-amz-bedrock-kb-source-uri`, `x-amz-bedrock-kb-data-source-id`
3. Automatic or scheduled crawling via Bedrock

**Benefits:**
- No custom scraping code to maintain
- Automatic JavaScript rendering
- External PDF support built-in
- Managed service with automatic retries
- Simpler architecture

## Components and Interfaces

### Web Crawler Configuration

**Data Source Type:** Web Crawler

**Source URL:**
```
https://www.orcuttschools.net
```

**Sync Scope:** Subdomains
- Automatically discovers and crawls all subdomains (orcuttacademy, oahs, lakeview, ojhs, aliceshaw, joenightingale, olgareed, pattersonroad, pinegrove, ralphdunlap, osis)

**Authentication:** No Authentication

**Regex Include Patterns:**
```
None
```
**Important:** Leave include patterns empty. The Subdomains scope automatically handles all *.orcuttschools.net domains. Adding regex patterns can cause crawl failures.

**Regex Exclude Patterns:**
```
None
```
**Important:** Leave exclude patterns empty initially. Adding regex patterns can cause crawl failures. The semantic search naturally filters out less relevant content like calendar events.

**External Content:**
- ParentSquare PDFs cannot be crawled due to robots.txt restrictions on their domain
- The web crawler respects robots.txt and will skip blocked domains
- Main school website content (2800+ pages) is successfully indexed

**Parsing Strategy:** Amazon Bedrock Default Parser

**Chunking Strategy:** Semantic chunking
- **Buffer:** 1
- **Max tokens:** 300
- **Breakpoint percentile threshold:** 95

**Embeddings Model:** Titan Text Embeddings v2
- **Embedding type:** Float vector embeddings
- **Vector dimensions:** 1024

**Vector Store:** Quick create vector store (recommended)
- **Type:** Amazon OpenSearch Serverless

**Content Types:**
- HTML pages
- PDF documents

### Code Changes

**File:** `lambda/chatbot/lambda_function.py`

**Modified Method:** `query_knowledge_base_semantic()`
```python
def query_knowledge_base_semantic(self, query: str, knowledge_base_id: str, 
                                 domain_filter: str = None, number_of_results: int = 60) -> Dict:
    """Query Knowledge Base using hybrid search with optional domain filtering"""
    
    config = {
        'vectorSearchConfiguration': {
            'numberOfResults': number_of_results,
            'overrideSearchType': 'HYBRID'
        }
    }
    
    # Add domain filter if provided (for school-specific queries)
    if domain_filter:
        config['vectorSearchConfiguration']['filter'] = {
            'startsWith': {
                'key': 'x-amz-bedrock-kb-source-uri',
                'value': f'https://{domain_filter}'
            }
        }
    
    response = self.bedrock_agent_runtime.retrieve(
        knowledgeBaseId=knowledge_base_id,
        retrievalQuery={'text': query},
        retrievalConfiguration=config
    )
    return response
```

**Modified Method:** `process_knowledge_base_response()`
```python
def process_knowledge_base_response(self, kb_responses: List[Dict]) -> Tuple[str, List]:
    """Process KB responses using standard Bedrock metadata"""
    
    for result in kb_response['retrievalResults']:
        # Extract source URI from standard Bedrock metadata
        source_uri = result.get('metadata', {}).get('x-amz-bedrock-kb-source-uri', 'NA')
        
        # Extract domain from URI for display
        domain = urlparse(source_uri).netloc if source_uri else 'NA'
        
        # Map domain to school name if applicable
        school_name = next((k for k, v in school_url_dict.items() 
                          if v == domain), domain)
        
        context += f"[Source {counter}]: source_url: {source_uri} School: {school_name}\n{text}\n\n"
        
        sources.append({
            "filename": f"Source {counter}",
            "url": source_uri,
            "s3Uri": None  # Not applicable for web crawler
        })
```

### Metadata Mapping

| Custom Scraper | Web Crawler | Notes |
|----------------|-------------|-------|
| `metadata['domain']` | Extract from `x-amz-bedrock-kb-source-uri` | Parse netloc from URI |
| `metadata['source']` | `x-amz-bedrock-kb-source-uri` | Direct replacement |
| `metadata['meeting_date']` | Not available | Optional field, handle gracefully |
| `location['s3Location']['uri']` | Not applicable | Web crawler doesn't use S3 |

### Domain Filtering Logic

**How it works:**
1. User selects school from UI dropdown (e.g., "Pine Grove Elementary")
2. Frontend sends `selectedSchool` parameter to backend
3. Backend maps school name to domain and applies filter
4. User can ask questions without mentioning the school name

**For school-specific queries:**
```python
# User selects "Pine Grove Elementary" in UI
# User asks: "When does school start?" (no school name in query)
selected_school = "Pine Grove Elementary"
domain_filter = school_url_dict[selected_school]  # "pinegrove.orcuttschools.net"

# Filter becomes:
{
    'startsWith': {
        'key': 'x-amz-bedrock-kb-source-uri',
        'value': 'https://pinegrove.orcuttschools.net'
    }
}
# Results: Only content from pinegrove.orcuttschools.net
```

**For general queries:**
```python
# No school selected in UI
# User asks: "When does school start?"
selected_school = None
domain_filter = None  # No filter applied, search all content
# Results: Content from all school domains
```

## Data Models

### Retrieval Result Structure (Web Crawler)
```python
{
    'content': {
        'text': 'Page content...'
    },
    'location': {
        'type': 'WEB',
        'webLocation': {
            'url': 'https://www.orcuttschools.net/page'
        }
    },
    'metadata': {
        'x-amz-bedrock-kb-source-uri': 'https://www.orcuttschools.net/page',
        'x-amz-bedrock-kb-data-source-id': 'DATASOURCE123',
        # Other standard Bedrock fields
    },
    'score': 0.85
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system.*

### Property 1: Source URI Extraction
*For any* retrieval result from the web crawler, the source URL SHALL be extractable from the `x-amz-bedrock-kb-source-uri` metadata field.
**Validates: Requirements 4.1**

### Property 2: Domain Filtering
*For any* school-specific query, only results with source URIs starting with the school's domain SHALL be retrieved.
**Validates: Requirements 3.1, 3.2**

### Property 3: External Content Capture
*For any* external PDF linked from the website, the web crawler SHALL index that content if the domain is in the inclusion filters.
**Validates: Requirements 2.1, 2.2, 6.2**

### Property 4: JavaScript Content Capture
*For any* page with JavaScript-rendered content, the web crawler SHALL capture the rendered content.
**Validates: Requirements 2.3, 6.1**

## Error Handling

### Web Crawler Sync Failures
- **Scenario:** Web crawler job fails
- **Handling:** Check CloudWatch logs, retry sync job
- **User Impact:** Content may be stale until next successful sync

### Missing Metadata Fields
- **Scenario:** Result doesn't have expected metadata
- **Handling:** Use default values ("NA"), log warning
- **User Impact:** Source attribution may be incomplete

### Domain Filter Errors
- **Scenario:** Invalid domain filter provided
- **Handling:** Fall back to no filter, log error
- **User Impact:** May retrieve results from all domains

## Customer Management

### AWS Console Access

**Customer administrators will need:**
- AWS Console access with permissions for:
  - Bedrock Knowledge Bases (read/write)
  - CloudWatch Logs (read)
  - S3 (read, for viewing indexed content)

### Common Management Tasks

**1. Trigger Manual Content Update:**
```
AWS Console → Bedrock → Knowledge Bases → [KB Name] → Data Sources → [Web Crawler] → Sync
```
- Click "Sync" button to start a new crawl
- Monitor progress in the console
- Typical sync time: 30-60 minutes

**2. View Crawl Job Status:**
```
AWS Console → Bedrock → Knowledge Bases → [KB Name] → Data Sources → [Web Crawler] → Sync history
```
- View recent sync jobs
- Check status (In Progress, Completed, Failed)
- View number of documents indexed

**3. Modify Crawler Settings:**
```
AWS Console → Bedrock → Knowledge Bases → [KB Name] → Data Sources → [Web Crawler] → Edit
```
- Add/remove seed URLs
- Modify inclusion/exclusion filters
- Adjust crawl depth
- Change crawl rate

**4. View Logs:**
```
AWS Console → CloudWatch → Log Groups → /aws/bedrock/knowledgebases/[KB-ID]
```
- View crawl errors
- Check which URLs were indexed
- Troubleshoot issues

### Documentation for Customer

Create a separate document: `docs/WEB_CRAWLER_MANAGEMENT.md` with:
- Step-by-step instructions with screenshots
- Common scenarios (adding new pages, updating content)
- Troubleshooting guide
- Contact information for technical support

## Testing Strategy

### Manual Testing
- Create new KB with web crawler
- Configure seed URLs and filters
- Run initial crawl
- Test queries:
  - "Who is the Superintendent?" (should find Dr. Holly Edds)
  - "How do I sign up for a classroom pizza party?" (should find catering info)
  - "Who are the Executive Directors?" (should find team page)
  - School-specific: "When does school start at Pine Grove?" (should filter to Pine Grove)

### Validation
- Verify external PDFs are indexed (check for bus schedules)
- Verify staff directory is complete (check for all staff members)
- Verify school filtering works (test with each school)
- Compare result quality with custom scraper

### Rollback Plan
- Keep custom scraper KB ID in config
- If web crawler doesn't work, switch back to custom KB
- Document any issues found for future improvement
