# Design Document

## Overview

This design improves the source citation display in the Orcutt Schools Chatbot by removing broken S3 presigned URLs and showing only original website URLs. The changes span both backend (Lambda) and frontend (React) components.

## Architecture

The system currently has a two-layer approach to source handling:

1. **Backend (Lambda)**: Processes knowledge base results and creates source metadata objects
2. **Frontend (React)**: Displays source metadata in the sidebar

The change will modify both layers to eliminate S3 presigned URLs and prioritize original website URLs.

## Components and Interfaces

### Backend Changes

**File**: `lambda/chatbot/lambda_function.py`

**Method**: `process_knowledge_base_response()`

Current behavior:
- Generates presigned URLs for all S3 content
- Stores both `url` (website) and `presignedUrl` (S3) in source metadata

New behavior:
- Only stores `url` (original website URL)
- Removes presigned URL generation entirely
- Simplifies source metadata structure

**Source Metadata Structure**:
```python
source_info = {
    "filename": str,  # Display name for the source
    "url": str,       # Original website URL (always present)
    "s3Uri": str      # S3 location (for internal reference only, not displayed)
}
```

### Frontend Changes

**Files**: 
- `frontend/src/components/Sidebar.js`
- `frontend/src/components/ChatInterface.js`

Current behavior:
- Filename is clickable → opens presignedUrl (broken)
- Small 🔗 icon → opens url (works)

New behavior:
- Display format: `🔗 [URL as hyperlink]`
- Single click handler that opens `source.url`
- Remove presignedUrl handling entirely

## Data Models

### Source Object (Backend to Frontend)

**Before**:
```javascript
{
  filename: "orcuttschools_34729_3.txt",
  url: "http://orcuttschools.net/34729_3",
  s3Uri: "s3://bucket/orcuttschools_34729_3.txt",
  presignedUrl: "https://bucket.s3.amazonaws.com/..."
}
```

**After**:
```javascript
{
  filename: "Child Nutrition Services",
  url: "http://orcuttschools.net/34729_3",
  s3Uri: "s3://bucket/orcuttschools_34729_3.txt"
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system-essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: URL Presence
*For any* source object returned by the backend, if the source has metadata with an original URL, then the source object SHALL contain a `url` field with that URL value.
**Validates: Requirements 1.1, 3.2, 3.3**

### Property 2: No Presigned URLs
*For any* source object returned by the backend, the source object SHALL NOT contain a `presignedUrl` field.
**Validates: Requirements 3.1**

### Property 3: Clickable Links
*For any* source displayed in the frontend with a `url` field, clicking on that source SHALL open the URL in a new browser tab.
**Validates: Requirements 1.2**

### Property 4: Display Format
*For any* source displayed in the frontend, the display SHALL show a link icon followed by the URL as a clickable hyperlink.
**Validates: Requirements 1.3, 2.2**

## Error Handling

### Missing URL
- **Scenario**: Source has no original URL in metadata
- **Handling**: Display filename only without link icon or hyperlink
- **User Impact**: User sees source reference but cannot click through

### Invalid URL
- **Scenario**: URL in metadata is malformed
- **Handling**: Display as plain text, log warning
- **User Impact**: User sees URL but link may not work

## Testing Strategy

### Unit Tests
- Test `process_knowledge_base_response()` with various source types
- Verify source objects contain `url` but not `presignedUrl`
- Test frontend rendering with and without URLs

### Integration Tests
- End-to-end test: Ask question → verify sources display correctly
- Click test: Verify clicking source opens correct URL in new tab
- Edge case: Source without URL displays appropriately

### Manual Testing
- Test with web-scraped content (should show website URL)
- Test with PDF content (should show original PDF URL if available)
- Verify no "Access Denied" errors when clicking sources
