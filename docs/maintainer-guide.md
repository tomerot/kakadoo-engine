# Maintainer Guide

This guide is for developers who maintain or extend Kakadoo Engine. It covers setup, how the notebook is organized, the microservices and their main functions, the interface code, the database layout, and what to adjust when the AWS website changes. For an overview of the project, see the [README](../README.md). For using the system, see the [User Guide](user-guide.md).

The whole application is a single Jupyter notebook, [`src/kakadoo_engine.ipynb`](../src/kakadoo_engine.ipynb), developed and run in Google Colab.

## Contents

- [Setup](#setup)
- [Notebook Structure](#notebook-structure)
- [Code Conventions](#code-conventions)
- [Microservices](#microservices)
  - [Crawler](#crawler--crawlerservice)
  - [Index Creator](#index-creator--indexcreatorservice)
  - [Data Fetcher](#data-fetcher--datafetcherservice)
  - [Administration](#administration--administrationservice)
  - [Query](#query--queryservice)
  - [Statistics](#statistics--statisticservice)
- [Interface](#interface)
- [Implementation Notes](#implementation-notes)
- [Database Structure](#database-structure)
- [Maintenance](#maintenance)
- [Libraries](#libraries)

## Setup

1. **Open the notebook in Google Colab.**
2. **Create a Firebase Realtime Database.** Create a Firebase project, add a Realtime Database and start it in test mode. The notebook connects without authentication, through the database's REST API. Test-mode rules expire after 30 days, so update the database rules to keep access after that.
3. **Add Colab Secrets** in the **Secrets** panel (the key icon in the left sidebar):

   | Secret | Required | Value |
   |---|---|---|
   | `FIREBASE_DB_URL` | Yes | The database URL, for example `https://<database-name>.<region>.firebasedatabase.app/` |
   | `GEMINI_API_KEY` | No | A Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey). Only the AWS Chatbot needs it. |

   On the first run, Colab asks you to allow the notebook to read these secrets.
4. **Set the Gemini model.** If you added a Gemini key, set `GEMINI_MODEL` at the top of the AWS Chatbot GUI cell. The project was built with `gemini-1.5-flash`, which may be retired by the time you read this; if so, use any current Gemini model.
5. **Run all cells.** The dashboard appears under the last cell. On the first run, build the index from the admin panel: see [Managing the Index](user-guide.md#managing-the-index).

**Missing secrets don't crash the app:**
- **Without `FIREBASE_DB_URL`:** the database connection (`FBconn`) is `None`. Search, Statistics and the Admin Panel show "The search index isn't available." AWS Fun Facts and the chatbot keep working.
- **Without `GEMINI_API_KEY`:** the chatbot answers "The chatbot isn't configured." and everything else keeps working.

## Notebook Structure

The cells are grouped under headings, in this order. Run them top to bottom: later cells use classes defined in earlier ones, and the last cell starts the app.

| Section | Contents |
|---|---|
| Libraries and Installations | Imports and installations. |
| Crawler Microservice | `CrawlerService` |
| Index Creator Microservice | `IndexCreatorService` |
| Data Fetcher Microservice | `DataFetcherService` |
| Administration Microservice | `AdministrationService` |
| Query Microservice | `QueryService` |
| Statistics Microservice | `StatisticService` |
| HTML & CSS | CSS injection and reusable styled components. |
| Main View GUI | Navigation buttons, the base view and the main dashboard. |
| Admin Page GUI | The admin panel. |
| Enter Query GUI | The search page, including AWS Fun Facts. |
| Query Results GUI | The search results page. |
| AWS Chatbot GUI | The chatbot's model setup, guardrail and page. |
| Statistics GUI | The statistics page. |
| Controllers | Button handling, navigation and the dashboard's startup function. |
| Main | Connects to the database, creates the microservices and starts the dashboard. |

## Code Conventions

- **One method per cell.** Each class cell defines only the constructor. Every method is written as a standalone function in its own cell, attached to the class (for example `CrawlerService.crawl = crawl`), and the standalone name is then deleted. This keeps cells short and lets you edit and re-run a single method without redefining the class.
- **Internal helpers** start with a double underscore, for example `__normalize_url`. Because they're attached outside the class body, Python doesn't rename them, so the prefix is only a convention marking them as internal.
- **Dependency injection.** Each microservice receives the services it depends on, and the database connection, through its constructor. The Main cell is the only place where they're created and connected, so any of them can be replaced, for example with a stand-in during testing, without changing the others.
- **Views.** Each page is a class that inherits from `BaseView`, a vertical layout that also adds the decorative cloud image. `MainDashboardView` holds the content area and swaps pages in and out, and `DashboardController` connects the main page's buttons to their handlers. The views use the microservice instances created in the Main cell (`query_service`, `administration_service`, `statistics_service`, `data_fetcher_service`) as global variables.

## Microservices

### Crawler – `CrawlerService`

Constructor parameters: `domain` (default `https://aws.amazon.com/`), `max_urls` (default 200), `non_relevant_language_codes` and `non_relevant_keywords` (defaults are defined in the constructor).

| Function | Description |
|---|---|
| `__is_within_domain(url)` | Whether the address is on the crawled domain. |
| `__contains_non_english_language_codes(url)` | Whether the address contains a language code such as `/es/` or `/ja/`. |
| `__contains_non_relevant_keywords(url)` | Whether the address contains an irrelevant keyword such as `careers`, `legal` or `/feed/`. |
| `__not_relevant_url(url)` | Combines the three checks above. |
| `__normalize_url(url)` | Removes query parameters and fragments and adds a trailing slash, so one page isn't crawled twice under different addresses. |
| `crawl()` | A breadth-first crawl from the domain's homepage, using a queue. Each page is downloaded with Requests and parsed with BeautifulSoup. Each link on it is converted to a full address with `urljoin`, normalized, and added to the queue if it hasn't been visited. Stops after `max_urls` pages and returns `{url: parsed page}`. |

### Index Creator – `IndexCreatorService`

Constructor parameters: `crawler_service`, `stop_words` (an AWS-tuned default list) and `freq_threshold` (default 7).

| Function | Description |
|---|---|
| `__create_terms_histogram(soup)` | Counts every word on the page, in lowercase. Purely numeric words are skipped. |
| `__apply_lemmatization(term, lemmatizer)` | Returns the word's dictionary form, trying it as a noun, then a verb, then an adjective. |
| `__normalize_terms(histogram)` | Lemmatizes every word, merging the counts of forms that reduce to the same word. |
| `__remove_stop_words(histogram)` | Removes the stop words. |
| `__remove_low_freqs(histogram, threshold)` | Removes words that appear fewer than `threshold` times on the page. |
| `__fetch_doc_title(url, soup)` | The page's `<title>`, otherwise its first `<h1>`, otherwise "No Title Available". |
| `__add_term(index, term)` / `__add_doc(...)` | Add a word to the index, and add a page's entry under a word. |
| `create_index()` | Runs the crawler, then processes each page in this order: count words, lemmatize, remove stop words, remove rare words, add to the index. Returns `{"index": index}`. |

### Data Fetcher – `DataFetcherService`

Constructor parameter: `FBconn`, the database connection, or `None` when no database is configured.

| Function | Description |
|---|---|
| `is_available()` | Whether a database connection is configured. The interface uses it to decide whether search, statistics and the admin panel can run. |
| `fetch_index()` | The whole index; empty if no database is configured or the read fails. |
| `fetch_term_docs(term)` | The pages stored under one word; empty if none. |

### Administration – `AdministrationService`

Constructor parameters: `FBconn`, `index_creator_service` and `data_fetcher_service`. At startup it loads a local copy of the index, which `fetch_terms` and `fetch_urls` read from and the delete functions keep up to date.

| Function | Description |
|---|---|
| `delete_index()` | Deletes the whole index from the database. |
| `recreate_index()` | Deletes the index, builds a new one with the Index Creator and uploads it. Returns a status message. Because the old index is deleted first, a failed rebuild leaves the database without an index. |
| `fetch_terms()` | All words in the index. |
| `fetch_urls(term)` | The pages stored under a word, with their counts. |
| `delete_docs(term, docs)` | Deletes pages from a word. If no pages remain, the word itself is deleted. |
| `delete_term(term)` | Deletes a word and all its pages. |

### Query – `QueryService`

Constructor parameter: `data_fetcher_service`.

| Function | Description |
|---|---|
| `__normalize_query(query)` | Lowercases the query, splits it into words and lemmatizes them, the same way pages are processed. |
| `__url_contains_terms(url, query_terms)` | How many of the query's words appear in a page's address. |
| `__update_maps(...)` | Accumulates, per page, its address, title, total count of query words and number of matched query words. |
| `__calculate_ranks(...)` | Computes each page's score (below). |
| `__fetch_results(...)` | Sorts the pages by score and returns their titles, addresses and scores. |
| `process_query(query)` | Runs the steps above for a query. |

Each page's score is:

```
score = (sum of the query words' counts on the page) × (matched query words + 2 × query words found in the address) / (number of query words)
```

### Statistics – `StatisticService`

Constructor parameter: `data_fetcher_service`. The index is loaded once at startup, so after rebuilding the index, re-run the Main cell to update the statistics.

| Function | Description |
|---|---|
| `get_most_common_words(n)` | The `n` words with the highest total count across all pages (`n` between 3 and 10). |
| `get_least_common_words(n)` | One random word from each of the `n` lowest total counts. |
| `get_random_words(n)` | `n` random words with their total counts. |
| `get_common_docs(n)` | The `n` pages that appear under the most words. |
| `generate_bar_chart(...)` / `generate_pie_chart(...)` | Draw a chart in the site's blue color scheme and return it as an HTML image. |
| `_convert_fig_to_html(fig)` | Converts a Matplotlib figure into an embedded image. |
| `generate_line_chart(...)` | A line chart; currently not used. |

## Interface

| Section | Main parts |
|---|---|
| HTML & CSS | `inject_css` defines the shared styles: button colors (default, warning, danger), the background gradient and the cloud image. `create_header`, `create_blur_divider`, `create_ResultView_page`, `create_admin_page` and `create_statistics_chart_page` return reusable styled components. |
| Main View GUI | `create_navigation_buttons` creates the **Admin Page** button and the **Run Query**, **Statistics** and **AWS Chatbot** buttons. `create_photo_container`, `BaseView`, `MainDashboardView` (`show_main_buttons`, `show_view`), and `UnavailableView`, which is shown instead of a page that needs the index when no database is configured. |
| Admin Page GUI | `AdminPageView`: `on_search` (the **Display Term Information** button), `update_table_data`, `on_select_all_change`, `on_delete_url`, `on_delete_term`, `on_recreate_index` and `on_return`. |
| Enter Query GUI | `EnterQueryView`: `on_search_clicked`, which shows an error for an empty query, a message when the index isn't available, and otherwise the results page; `on_fun_fact_clicked`, which holds the list of fun facts; and `on_return_main_clicked`. |
| Query Results GUI | `ResultView` runs the query through `query_service` and shows the results table. `on_return_to_query` returns to the search page. |
| AWS Chatbot GUI | `GEMINI_MODEL`; `SYSTEM_INSTRUCTION`, the rule given to the model; `model`, which is `None` without an API key; `AWS_KEYWORDS` and `AWS_KEYWORDS_PATTERN`, which match whole words with an optional plural "s"; `ask_aws_chatbot`; and `AWSChatbotView`. |
| Statistics GUI | `StatisticsView` with its four tabs: `create_*_tab`, `generate_*_chart`, `refresh_random_chart` (the **Random Again** button) and `on_return`. |
| Controllers | `DashboardController`: `setup_callbacks` connects the four main-page buttons to `handle_admin_page`, `handle_enter_query`, `handle_statistics` and `handle_aws_chatbot`. `display_main_dashboard()` builds the page and injects the CSS. |
| Main | Reads `FIREBASE_DB_URL`, connects to the database (or sets `FBconn = None`), creates the six microservices, connects them and calls `display_main_dashboard()`. |

## Implementation Notes

**The chatbot's guardrail** works in `ask_aws_chatbot`, in this order:
1. Without a model, it returns "The chatbot isn't configured."
2. A question with no word from `AWS_KEYWORDS` is rejected without calling the model, so no tokens are spent.
3. Otherwise, the question goes to the model, whose system instruction tells it to refuse anything that isn't about AWS.

**The admin password check** waits 0.2 seconds before reading the password box. The text box can update its value slightly after typing, so a password that was typed or pasted quickly and immediately submitted could otherwise be read incomplete.

## Database Structure

The index is stored under `index` in the Realtime Database:

```
index
└── <word>
    └── DocIDs
        └── doc_<n>
            ├── url
            ├── title
            └── count
```

For example (the values are illustrative):

```json
{
  "index": {
    "lambda": {
      "DocIDs": {
        "doc_12": {
          "url": "https://aws.amazon.com/lambda/",
          "title": "Serverless Computing - AWS Lambda - Amazon Web Services",
          "count": 41
        }
      }
    }
  }
}
```

- `<n>` is the page's position in the crawl, so within one build of the index, the same page has the same `doc_<n>` under every word.
- `count` is how many times the word appears on that page.
- Words contain only letters and digits, so they're always valid Firebase keys.

## Maintenance

The crawler depends on how the AWS website is built (see [Limitations](../README.md#limitations)), so expect to adjust it over time:

| Symptom | What to adjust |
|---|---|
| Relevant pages are missing | Raise `max_urls` in the Main cell, for example `CrawlerService(max_urls=500)`. Also check that AWS still writes the links to those pages in the page's HTML; links added by JavaScript aren't visible to the crawler. |
| Irrelevant pages appear in results | Add a keyword from their address to `non_relevant_keywords` in `CrawlerService`. |
| Pages in other languages appear | Add the language's address code to `non_english_language_codes`. |
| Meaningless words dominate the statistics | Add them to `stop_words` in `IndexCreatorService`. |
| The index is too large or too small | Adjust `freq_threshold` in `IndexCreatorService`. |
| The chatbot reports a model error | Set `GEMINI_MODEL` to a current Gemini model. |
| The chatbot rejects valid AWS questions | Add the missing terms to `AWS_KEYWORDS`. |
| The admin password needs changing | Change it in `handle_admin_page` in the Controllers section. |

## Libraries

| Library | Used for |
|---|---|
| `ipywidgets` | Interactive interface elements: buttons, text boxes, dropdowns, tabs and layout containers. |
| `IPython.display` | Displaying HTML, injecting CSS and clearing cell output. |
| `firebase` | REST client for the Firebase Realtime Database, installed in the first cell with `!pip install firebase`. |
| `requests` | Downloading pages during the crawl. |
| `bs4` (BeautifulSoup) | Parsing HTML: extracting links, text and page titles. |
| `nltk` | The WordNet lemmatizer, which reduces words to their dictionary form. The WordNet data is downloaded in the first cell. |
| `re` | Splitting text into words, and the chatbot's keyword filter. |
| `urllib.parse` | `urlparse` for domain checks and URL normalization; `urljoin` for converting relative links into full addresses. |
| `collections` | `deque` as the crawler's breadth-first queue; `Counter` for statistics. |
| `random` | Random fun facts, random search-box placeholders and random-word statistics. |
| `matplotlib`, `io`, `base64` | Drawing the statistics charts and embedding them in the page as images. |
| `time` | A short delay in the admin password check. |
| `google.generativeai` | The Gemini API client used by the AWS Chatbot. |
| `google.colab.userdata` | Reading Colab Secrets. |
