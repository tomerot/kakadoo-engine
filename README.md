<p align="center">
  <img src="https://github.com/user-attachments/assets/f431a57c-1fd8-4d11-b8bd-fb82bbd61b79" alt="Kakadoo Engine"/>
</p>

<p align="center">
  <a href="https://www.youtube.com/watch?v=C0nFFuhSl3c">▶ Watch the demo video</a>
</p>

**Kakadoo Engine** is a search engine for AWS, the world's largest cloud provider. Type in a topic, such as "auto scaling" or "relational database", and it returns the pages from the AWS website that explain it, ranked by relevance. The results come from an index the engine builds itself by crawling the AWS website.

It runs inside a Jupyter notebook, but it doesn't look like one: the interface was designed to look and work like a website, with page navigation, a search bar, results tables, charts and an admin area.

The project has two goals:

- **Demonstrate the microservices architecture (the main goal)** – the design style commonly used for cloud applications, in which a system is built from small services that each have a single responsibility. Here, the microservices are separate Python classes that run together in one notebook rather than independently deployed services, so the project demonstrates how the logic is divided, not how it would be deployed (see [In Production](#in-production)).
- **Understand how search engines work** – by building every stage from scratch: crawling, indexing and ranking.

## Features

- **Search** – type a topic in plain English and get the matching AWS pages, sorted from most to least relevant, with each page's relevance score shown. The score starts from one signal and is boosted by two others:
  - **Base** – how often the search words appear on the page.
  - **Boost** – how many of the search words the page contains, so a page that matches the whole search beats one that repeats a single word.
  - **Strongest boost** – whether the search words appear in the page's address. AWS addresses are descriptive (for example, `aws.amazon.com/lambda/`), so this is the clearest sign of what the page is about, and it weighs the most.
- **Admin Panel** – manages the index, the data structure that search runs on. The index is built by crawling the AWS website and recording which words appear on which pages, and how often. It's an *inverted* index, organized by word rather than by page, so a search goes straight to the pages that contain each word instead of scanning every page. From the panel, an admin can rebuild the index from a fresh crawl to keep links and content up to date, browse the indexed words and the pages each one appears on, and delete a word or specific links manually.
- **Statistics** – charts about the index: the most and least common words, a random sample of words, and the pages that contain the most indexed words.
- **AWS Chatbot** – an LLM-based assistant that gives short, simple answers to questions about AWS, aimed at new AWS users. Off-topic questions are blocked by a two-layer guardrail: a keyword filter rejects questions with no AWS-related words before they reach the model, so no tokens are spent on them, and the model itself is instructed to refuse anything that isn't about AWS.
- **AWS Fun Facts** – shows a random fact about AWS and its history, such as that it launched in 2006 with just three services.

## Screenshots

**1. Main dashboard**
![Main dashboard](https://github.com/user-attachments/assets/9a087328-863b-4ef4-88cc-712427090216)

**2. Search results**
![Search results](https://github.com/user-attachments/assets/9be50de5-45dd-4568-a326-071f8bed61cf)

**3. Statistics – most referenced documents**
![Most referenced documents](https://github.com/user-attachments/assets/94aa5e57-9ac2-43c5-8730-6ceda1298c94)

**4. Statistics – most common words**
![Most common words](https://github.com/user-attachments/assets/39042a0e-543a-481e-a385-8ba7d7cf77f7)

**5. AWS Chatbot**
![AWS Chatbot](https://github.com/user-attachments/assets/b5235a48-e60b-4b22-9c59-93c7fb7175db)

**6. Admin Panel**
![Admin Panel](https://github.com/user-attachments/assets/fdc2eb1c-74c3-4a18-96b1-cef2d6d8605c)

## Architecture

The logic behind the interface is split into six microservices, each with a single responsibility:

| Microservice | Responsibility |
|---|---|
| Crawler | Collects up to 200 pages from aws.amazon.com, skipping other sites, non-English versions and irrelevant pages (sign-in, legal, careers, etc.). It crawls breadth-first: the homepage, then every page it links to, then the pages those link to. That way the page limit is spent on the main page of each service and topic, rather than on one long chain of links that drifts further from the main topics with every step. |
| Index Creator | Turns the crawled pages into the inverted index. It extracts each page's words, reduces them to their dictionary form, and removes stop words: common words like "the", plus generic AWS marketing words like "explore" and "build" that would match almost every page. It also drops words that appear fewer than 7 times on a page, then records each remaining word's count along with the page's address and title. |
| Data Fetcher | Reads from the database: the full index, or the entry for a single word. |
| Administration | Writes to the database: rebuilds the index (crawl → index → upload) and deletes words or pages. |
| Query | Converts the search text into index words the same way pages were processed, looks them up, and scores and sorts the matching pages. |
| Statistics | Computes statistics from the index and draws them as charts. |

### Why This Architecture?

- **Changes stay inside one microservice.** Each microservice has one responsibility and is used only through its methods, so changing how it works internally doesn't affect the microservices that depend on it. During the project, the ranking formula was reworked entirely inside the Query microservice; indexing, storage and the interface didn't change.
- **Teams can work in parallel.** Each microservice can be owned and developed by a different developer or team without touching anyone else's code.
- **Each part can be tested on its own.** Every microservice receives the microservices it depends on when it's created, so a test can pass in a stand-in. For example, the Query microservice's ranking can be tested with a fake Data Fetcher that returns fixed data, with no database or crawl involved.

### In Production

Running Kakadoo as real microservices would mean:

- deploying each microservice separately (for example, in its own container) with its own HTTP API, instead of calling it as a method inside the notebook;
- replacing the notebook interface with a web frontend that calls those APIs;
- running the crawl and index build as a scheduled background job (for example, nightly) instead of an admin button.

That is where the architecture pays off fully:

- **Independent scaling** – more copies of the Query microservice during heavy search traffic, while the crawler runs only when scheduled.
- **Fault isolation** – if the crawler fails, search keeps serving the existing index.
- **Independent releases** – a new ranking formula ships by redeploying the Query microservice alone.

On AWS, for example, the Query and Data Fetcher microservices could run as Lambda functions behind API Gateway, the crawl as a scheduled ECS task, and the index could be stored in DynamoDB.

## Tech Stack

| Tool | Purpose |
|---|---|
| Jupyter Notebook on Google Colab | Development and runtime environment. The notebook format lets each microservice be written, run and inspected in its own cells; Colab hosts the notebook in the cloud, so it runs on Google's machines with no local setup and can be shared across the team. |
| ipywidgets + IPython.display | Render interactive elements (buttons, text fields, dropdowns, tabs) styled with HTML and CSS, to simulate a website-like interface inside the notebook. |
| Figma | Design tool for the interface. The design was exported to HTML and CSS with Figma plugins, then adapted to ipywidgets. |
| Requests + BeautifulSoup | Requests downloads each page; BeautifulSoup parses its HTML, so the crawler can follow links and the index can be built from each page's text and title. |
| NLTK (WordNet lemmatizer) | Reduces words to their dictionary form, so "instances" and "instance" count as the same word. Lemmatization was chosen over stemming, which cuts words down to rule-based stems such as "servic" or "databas". The indexed words appear in the statistics charts and the admin panel, so they need to be real, readable words. |
| Firebase Realtime Database | Cloud-hosted database that stores the index. The index persists between sessions and is shared by everyone running the notebook, so it doesn't need to be rebuilt on every run. Accessed over its REST API with the `firebase` Python package. |
| Gemini API | Cloud LLM behind the AWS Chatbot. The project used `gemini-1.5-flash` through the `google-generativeai` package. |
| Matplotlib | Draws the statistics charts, which are rendered as images inside the dashboard. |

## Results

- **Responsive search** – results return in about 1–1.5 seconds, so searching feels immediate and users never wait long enough to wonder whether it's working.
- **Relevant results first** – combining word frequency, search coverage and address matching puts pages that are actually about the searched topic at the top, rather than pages that merely repeat its words.
- **Broad coverage** – thanks to the breadth-first crawl, the 200-page index covers a wide range of AWS services through their main pages, instead of going deep into one narrow corner of the site, as a depth-first crawl would.

## Limitations

- **No JavaScript-rendered pages.** AWS Documentation and re:Post hold a lot of valuable content, but their pages are built by JavaScript in the browser. The crawler downloads raw HTML with Requests and parses it with BeautifulSoup, and neither runs JavaScript, so these pages arrive mostly empty. Indexing them would require a headless browser such as Selenium or Playwright, so they were left out of scope.
- **Depends on the AWS website's structure.** The crawler was built around how aws.amazon.com worked when the project was developed, and it decides which pages to skip by words in their address (like `/careers/`, or `/es/` for Spanish pages). If AWS moves more pages to JavaScript (see above) or changes how its addresses look, important pages can be missed and irrelevant ones can get in, so the crawler needs ongoing maintenance to keep the results relevant.
- **English only.** The crawler skips non-English versions of pages, and the stop words and lemmatizer are English-only.

## Running It

1. Open [`src/kakadoo_engine.ipynb`](src/kakadoo_engine.ipynb) in Google Colab.
2. Create your own Firebase Realtime Database in test mode (the notebook connects without authentication).
3. Optionally, create your own Gemini API key in [Google AI Studio](https://aistudio.google.com/apikey) to enable the AWS Chatbot.
4. In Colab's **Secrets** panel (the key icon in the left sidebar), add:
   - `FIREBASE_DB_URL` (required) – your database URL. Without it, search, statistics and the admin panel are unavailable.
   - `GEMINI_API_KEY` (optional) – your Gemini API key. Without it, only the AWS Chatbot is unavailable.
5. If you added a Gemini key, set `GEMINI_MODEL` in the AWS Chatbot cell to the Gemini model you want to use. The project was built with `gemini-1.5-flash`, which may be retired by the time you run this; if so, choose any current Gemini model.
6. Run all cells. The dashboard appears under the last cell.
7. On the first run, click **Admin Page** (demo password: `123456`), then **Recreate Index** to crawl AWS and build the index. With the default limit of 200 pages, this took about 2 minutes on Colab when the project was developed; the time can change as the AWS website changes. To index more pages, set `max_urls` where the Crawler is created in the last cell, for example `CrawlerService(max_urls=500)`. More pages take longer to crawl.
