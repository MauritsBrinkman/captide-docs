---
sidebar_position: 2
---

# Frontend Integration

This guide explains how to integrate Captide's document viewer and SEC Q&A capabilities into your React frontend application.

## Important Security Note

**Never expose your Captide API key in frontend code!** Always use a backend proxy as shown in the [Backend Integration](./backend) guide.

## React Integration

### Basic Document Viewer Setup

Here's how to set up the Captide document viewer component in a React application:

```jsx
import React, { useState } from 'react';
import { DocumentViewer, DocumentViewerProvider } from 'captide';

function App() {
  // Function to fetch document through your backend API
  const fetchDocument = async (sourceLink) => {
    const response = await fetch('/api/document', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ source_link: sourceLink })
    });
    
    return response.json();
  };

  return (
    <div className="app">
      <h1>Captide Document Explorer</h1>
      
      <DocumentViewerProvider fetchDocumentFn={fetchDocument}>
        <DocumentExplorerDemo />
      </DocumentViewerProvider>
    </div>
  );
}

function DocumentExplorerDemo() {
  const [answer, setAnswer] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const [query, setQuery] = useState('');
  
  // Create a simple form for asking questions
  const handleSubmit = async (e) => {
    e.preventDefault();
    setIsLoading(true);
    
    try {
      // Call your backend API that proxies to Captide
      const response = await fetch('/api/document-snippets', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ query })
      });
      
      const data = await response.json();
      // Process and display the response
      // This is a simplified example - you'd typically render the snippets
      setAnswer(JSON.stringify(data, null, 2));
    } catch (error) {
      console.error('Error:', error);
    } finally {
      setIsLoading(false);
    }
  };
  
  return (
    <div>
      <form onSubmit={handleSubmit}>
        <input
          type="text"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="Ask about SEC filings (e.g., What's AAPL's revenue?)"
        />
        <button type="submit" disabled={isLoading}>
          {isLoading ? 'Loading...' : 'Ask'}
        </button>
      </form>
      
      {answer && (
        <pre>{answer}</pre>
      )}
      
      {/* The DocumentViewer needs to be sized explicitly */}
      <div style={{ height: '600px', marginTop: '20px', border: '1px solid #ccc' }}>
        <DocumentViewer />
      </div>
    </div>
  );
}

export default App;
```

### Streaming Response Example

For handling streaming responses from the AI agent:

```jsx
import React, { useState, useEffect, useRef } from 'react';
import { DocumentViewer, DocumentViewerProvider, useDocumentViewer } from 'captide';

function StreamingDemo() {
  const [query, setQuery] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const [streamedText, setStreamedText] = useState('');
  const [idMapping, setIdMapping] = useState({});
  const { loadDocument } = useDocumentViewer();
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    setIsLoading(true);
    setStreamedText('');
    
    try {
      // Call your backend API that proxies the streaming endpoint
      const response = await fetch('/api/query-stream', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ query })
      });
      
      // Create a reader for the stream
      const reader = response.body.getReader();
      const decoder = new TextDecoder();
      
      while (true) {
        const { value, done } = await reader.read();
        if (done) break;
        
        const text = decoder.decode(value);
        // Process the SSE data
        const lines = text.split('\n');
        
        for (const line of lines) {
          if (line.startsWith('data: ')) {
            try {
              const data = JSON.parse(line.substring(6));
              
              // Handle different event types
              if (data.type === 'id_mapping') {
                setIdMapping(data.mapping);
              } else if (data.type === 'markdown_chunk') {
                setStreamedText(prev => prev + data.content);
              } else if (data.type === 'full_answer') {
                setStreamedText(data.content);
              }
            } catch (e) {
              console.error('Error parsing SSE data:', e);
            }
          }
        }
      }
    } catch (error) {
      console.error('Error with streaming:', error);
    } finally {
      setIsLoading(false);
    }
  };
  
  // Function to handle source link clicks
  const handleSourceClick = async (sourceId) => {
    // Extract source info from the mapping
    const sourceInfo = idMapping[sourceId];
    if (sourceInfo && sourceInfo.sourceLink) {
      await loadDocument(sourceInfo.sourceLink, sourceId);
    }
  };
  
  // Simple regex to detect source references like [#123abc]
  const renderWithSourceLinks = (text) => {
    if (!text) return '';
    
    // Split by references [#id] and render as clickable spans
    const parts = text.split(/(\[#[a-z0-9]+\])/g);
    
    return parts.map((part, index) => {
      // Check if this part is a reference
      const match = part.match(/\[#([a-z0-9]+)\]/);
      
      if (match) {
        const sourceId = `#${match[1]}`;
        return (
          <span 
            key={index}
            className="source-link"
            onClick={() => handleSourceClick(sourceId)}
            style={{ cursor: 'pointer', color: 'blue', textDecoration: 'underline' }}
          >
            [{index + 1}]
          </span>
        );
      }
      
      // Regular text
      return <span key={index}>{part}</span>;
    });
  };
  
  return (
    <div>
      <form onSubmit={handleSubmit}>
        <input
          type="text"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="Ask about SEC filings (e.g., What's AAPL's revenue?)"
          style={{ width: '400px' }}
        />
        <button type="submit" disabled={isLoading}>
          {isLoading ? 'Loading...' : 'Ask'}
        </button>
      </form>
      
      <div className="answer-container">
        {renderWithSourceLinks(streamedText)}
      </div>
      
      <div style={{ height: '600px', marginTop: '20px', border: '1px solid #ccc' }}>
        <DocumentViewer />
      </div>
    </div>
  );
}

function App() {
  const fetchDocument = async (sourceLink) => {
    const response = await fetch('/api/document', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ source_link: sourceLink })
    });
    
    return response.json();
  };

  return (
    <DocumentViewerProvider fetchDocumentFn={fetchDocument}>
      <StreamingDemo />
    </DocumentViewerProvider>
  );
}

export default App;
```

## Next.js Integration

If you're using Next.js, you can create API routes to proxy requests to Captide:

```javascript
// pages/api/document.js
export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }

  try {
    const { source_link } = req.body;
    
    const response = await fetch(source_link, {
      headers: {
        'Authorization': `Bearer ${process.env.CAPTIDE_API_KEY}`
      }
    });
    
    if (!response.ok) {
      throw new Error(`API error: ${response.status}`);
    }
    
    const data = await response.json();
    res.json(data);
  } catch (error) {
    console.error('Error fetching document:', error);
    res.status(500).json({ error: 'Failed to fetch document' });
  }
}
```

## Live Example

To see a complete implementation of Captide's capabilities, visit [app.captide.co/chat](https://app.captide.co/chat).

## Next Steps

- For production use, obtain a license by [contacting our team](https://www.captide.co/demo)
- Access in-depth API documentation in the [API Reference](/docs/api) 