# 🌐 Tavily Search, Extract & Crawl Integration

## Why Tavily?

LLMs ke paas training data hota hai jo kuch time baad outdated ho sakta hai.

Example:

> User: Who is the Chief Minister of Tamil Nadu?

Agar model ke paas latest information nahi hai, to wo purana answer de sakta hai.

Is problem ko solve karne ke liye Tavily real-time internet search provide karta hai, jisse AI ko fresh aur updated information milti hai.

---

## Installation

```bash
npm install @tavily/core
```

---

## Basic Search Example

```javascript
const { tavily } = require("@tavily/core");

const tvly = tavily({
  apiKey: "tvly-YOUR_API_KEY",
});

const response = await tvly.search(
  "Who is Lionel Messi?"
);

console.log(response);
```

---

# 🔍 Tavily Search

Tavily Search internet se real-time information fetch karta hai aur AI ko updated context provide karta hai.

## Search Depth Options

### ultra-fast

* Lowest latency
* Fastest response
* Time-critical queries ke liye

### fast

* Low latency
* Good relevance
* General applications ke liye

### basic

* Balanced speed and accuracy
* Har URL ka NLP summary return karta hai

### advanced

* Highest relevance
* Detailed search results
* Research aur high-accuracy use cases ke liye

---

## Search Cost

| Mode       | Credits |
| ---------- | ------- |
| ultra-fast | 1       |
| fast       | 1       |
| basic      | 1       |
| advanced   | 2       |

---

## Search Parameters

| Parameter         | Type    | Default | Description                   |
| ----------------- | ------- | ------- | ----------------------------- |
| max_results       | Integer | 5       | Maximum sources returned      |
| topic             | String  | general | General web ya news search    |
| time_range        | String  | null    | day, week, month, year        |
| start_date        | String  | null    | Custom start date             |
| end_date          | String  | null    | Custom end date               |
| chunks_per_source | Integer | 1       | Per source extracted snippets |

---

## Advanced News Search Example

```javascript
const response = await tvly.search(
  "AI advancements in quantum computing",
  {
    topic: "news",
    max_results: 5,
    time_range: "month",
    start_date: "2026-05-01",
    end_date: "2026-06-01",
    chunks_per_source: 2
  }
);
```

---

## Additional Search Features

| Parameter                 | Purpose                     |
| ------------------------- | --------------------------- |
| include_answer            | AI-generated concise answer |
| include_raw_content       | Full scraped page content   |
| include_images            | Related image URLs          |
| include_image_description | Image descriptions          |
| include_favicon           | Website icon                |
| include_usage             | Credit/token usage          |
| auto_parameters           | Automatic optimization      |
| exact_match               | Exact keyword matching      |

---

# 📄 Tavily Extract

Agar specific webpage ka content extract karna ho to Tavily Extract use kiya jata hai.

## Example

```javascript
const { tavily } = require("@tavily/core");

const tvly = tavily({
  apiKey: "tvly-YOUR_API_KEY"
});

const response = await tvly.extract(
  "https://en.wikipedia.org/wiki/Artificial_intelligence"
);

console.log(response);
```

## Use Cases

* Blog content extraction
* Article summarization
* RAG pipelines
* Documentation ingestion
* Knowledge base generation

---
## hmm same parmaeter yha extract main v chla skte hain 

## Multiple URLs Extract

```javascript
const response = await tvly.extract([
  "https://openai.com",
  "https://anthropic.com"
]);

console.log(response);
```

---

# Tavily Crawl

Agar sirf ek page nahi balki poori website explore karni ho to Crawl API use hoti hai.

## Basic Crawl

```javascript
const response = await tvly.crawl(
  "https://docs.langchain.com"
);

console.log(response);
```

---

## Advanced Crawl

```javascript
const response = await tvly.crawl(
  "https://docs.langchain.com",
  {
    max_depth: 3,
    max_pages: 50,
    extract_depth: "advanced",
    include_images: true,
    include_raw_content: true
  }
);

console.log(response);
```

---

## Crawl Workflow

```text
Website URL
     ↓
Find Internal Links
     ↓
Visit Pages
     ↓
Extract Content
     ↓
Clean Text
     ↓
Return Structured Data
```

---

# Website Knowledge Map

Crawl ke baad website ka page relationship map banaya ja sakta hai.

## Example

```javascript
const crawlData = await tvly.crawl(
  "https://docs.langchain.com"
);

const map = crawlData.pages.map(page => ({
  title: page.title,
  url: page.url
}));

console.log(map);
```

Output:

```json
[
  {
    "title":"Getting Started",
    "url":"https://docs.langchain.com/start"
  },
  {
    "title":"Agents",
    "url":"https://docs.langchain.com/agents"
  }
]
```

---

# Tavily Map

Map API website ke saare discoverable URLs return karti hai.

## Basic Map

```javascript
const response = await tvly.map(
  "https://docs.langchain.com"
);

console.log(response);
```

---

## Advanced Map

```javascript
const response = await tvly.map(
  "https://docs.langchain.com",
  {
    max_depth: 5,
    max_pages: 100
  }
);

console.log(response);
```

---

## Map Workflow

```text
Root URL
    ↓
Discover Links
    ↓
Build Site Structure
    ↓
Return URL Tree
```

---

# AI Agent Architecture Using Tavily

```text
User Query
      ↓
Tavily Search
      ↓
Latest Sources
      ↓
Tavily Extract
      ↓
Detailed Content
      ↓
Tavily Crawl
      ↓
Website Knowledge
      ↓
Vector Database
      ↓
LLM Context
      ↓
Final Response
```

---

# RAG Architecture

```text
Web Sources
      ↓
Tavily Search
      ↓
Tavily Extract
      ↓
Tavily Crawl
      ↓
Chunking
      ↓
Embedding
      ↓
Vector DB
      ↓
Retriever
      ↓
LLM
      ↓
Answer
```

---

# Best Use Cases

* AI Agents
* Research Assistants
* News Monitoring
* RAG Systems
* Knowledge Base Creation
* Real-Time Chatbots
* Website Intelligence
* Documentation Search
* Competitor Analysis

---

# Benefits

✅ Real-time web data

✅ Latest news access

✅ Website crawling

✅ Knowledge mapping

✅ Better factual accuracy

✅ Reduced hallucinations

✅ Multi-source verification

✅ Production-ready AI applications
