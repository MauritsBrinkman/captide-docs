---
sidebar_position: 2
---

# Quickstart Guide

This guide will help you get started with Captide quickly, showing you how to set up both the REST API and the JavaScript SDK to integrate financial document Q&A into your application.

## Prerequisites

Before you begin, make sure you have:

- A Captide API key (request one at [app.captide.co/api-dashboard](https://www.captide.ai/contact/api-request))

## Using the REST API

### Authentication

All API requests require authentication using your API key in the header:

```bash
curl -X POST "https://rest-api-captide.co/api/v1/rag/agent-query-stream" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What's the latest revenue of AAPL?"
  }'
```

### Example 1: Streaming Agent Response

Use this endpoint when you want to receive a complete AI-generated answer based on SEC documents:

```bash
curl -X POST "https://rest-api-captide.co/api/v1/rag/agent-query-stream" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What's the latest revenue of AAPL?"
  }'
```

This endpoint returns a Server-Sent Events (SSE) stream with:

1. An initial `id_mapping` event containing document references
2. Stream of `markdown_chunk` events with the answer text
3. A final `full_answer` event with the complete response

Example response (simplified):
```
data: {"type":"id_mapping","mapping":{"#a060daba":{"sourceLink":"...","sourceMetadata":{...}},...}}

data: {"type":"markdown_chunk","content":"Apple"}
data: {"type":"markdown_chunk","content":" Inc"}
...

data: {"type":"full_answer","content":"Apple Inc.'s latest revenue was $124,300 million for Q1 2025..."}

data: {"type":"done"}
```

### Example 2: Document Snippets Only

Use this endpoint when you want to receive only the relevant document snippets:

```bash
curl -X POST "https://rest-api-captide.co/api/v1/rag/chunks" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Whats revenue of aapl last quarter"
  }'
```

This returns JSON with an array of document snippets including:
- Full text content with HTML element IDs
- Document metadata (filing type, date, ticker, etc.)
- Source links for retrieving the full document

## Using the JavaScript SDK

### Installation

Install the Captide SDK in your project:

```bash
npm install captide
# or
yarn add captide
```

### Integration Example

Here's how to integrate the document viewer into your React application:

```jsx
import React from 'react';
import { DocumentViewer, DocumentViewerProvider, useDocumentViewer } from 'captide';

// Function to fetch document content from your backend that calls our API
const fetchDocument = async (sourceLink) => {
  const response = await fetch('/your-backend/document', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ source_link: sourceLink })
  });
  
  return response.json();
};

// Example component using the DocumentViewer
function App() {
  return (
    <DocumentViewerProvider fetchDocumentFn={fetchDocument}>
      <DocumentViewerDemo />
    </DocumentViewerProvider>
  );
}

function DocumentViewerDemo() {
  const { loadDocument } = useDocumentViewer();
  
  // IMPORTANT: sourceLink and elementId come from your API response
  // - sourceLink: The 'sourceLink' field from the id_mapping in the streaming response
  //   or the 'source_link' field in the chunks response
  // - elementId: The element ID reference (like '#a060daba') from either the streaming 
  //   or chunks response that you want to highlight
  const handleSourceLinkClick = async (sourceLink, elementId) => {
    // Load document and highlight specific element
    await loadDocument(sourceLink, elementId);
  };
  
  return (
    <div>
      <button 
        onClick={() => handleSourceLinkClick(
          'https://rest-api.captide.co/api/v1/document?source_type=10-Q&document_id=69443120-e3a3-4ebb-91b1-a55ff2afe141',
          '#ab12ef34'
        )}
      >
        View Source
      </button>
      
      <div style={{ height: '600px', width: '100%', border: '1px solid #ccc' }}>
        <DocumentViewer />
      </div>
    </div>
  );
}

export default App;
```

## Live Example

For a live example of Captide in action, visit [app.captide.co/chat](https://app.captide.co/chat).

## Next Steps

- Explore the [API Reference](/api) for detailed documentation
- Obtain a liscence for production or commercial use: [Contact our team](https://www.captide.ai/contact/api-request) 