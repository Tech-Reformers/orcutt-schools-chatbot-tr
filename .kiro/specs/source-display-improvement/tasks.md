# Implementation Plan

- [x] 1. Update backend to remove presigned URL generation
  - Modify `process_knowledge_base_response()` method in `lambda/chatbot/lambda_function.py`
  - Remove presigned URL generation logic
  - Ensure source objects only contain `url` field (not `presignedUrl`)
  - Keep `s3Uri` for internal reference but don't expose to frontend
  - _Requirements: 3.1, 3.2, 3.3_

- [x] 2. Update frontend Sidebar component
  - Modify `Sidebar.js` to display sources as `🔗 [URL]` format
  - Remove `handleFilenameClick` function (presignedUrl handler)
  - Update `handleLinkClick` to be the primary click handler
  - Change source display to show link icon + URL as hyperlink
  - Handle sources without URLs gracefully (display filename only)
  - _Requirements: 1.1, 1.2, 1.3, 2.1, 2.2, 2.3_

- [x] 3. Update frontend ChatInterface component
  - Modify `ChatInterface.js` to use URL-only click handling
  - Update `handleSourceClick` to only check for `source.url`
  - Remove presignedUrl fallback logic
  - _Requirements: 1.2_

- [x] 4. Test and deploy
  - Deploy backend changes
  - Rebuild and deploy frontend
  - Test with various question types (web content, PDFs)
  - Verify all source links open correct website URLs (not S3)
  - Verify sources without URLs display appropriately
  - _Requirements: All_
