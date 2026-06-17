# FitFindr — planning.md

> Complete this document before writing any implementation code.
> Your spec and agent diagram are what you'll use to direct AI tools (Claude, Copilot, etc.) to generate your implementation — the more specific they are, the more useful the generated code will be.
> Your planning.md will be reviewed as part of your submission.
> Update it before starting any stretch features.

---

## Tools

List every tool your agent will use. For each tool, fill in all four fields.
You must have at least 3 tools. The three required tools are listed — add any additional tools below them.

### Tool 1: search_listings

**What it does:**
Searches the mock listings dataset for secondhand items that match the user's item description, optional size, and optional maximum price.

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `description` (str): Keywords describing what the user is looking for, such as "vintage graphic tee".
- `size` (str): The requested size, such as "M", or `None` if the user does not provide a size.
- `max_price` (float): The requested size, such as "M", or `None` if the user does not provide a size.

**What it returns:**
Returns a list of listing dictionaries sorted by relevance. Each listing contains the following fields: `id`, `title`, `description`, `category`, `style_tags`, `size`, `condition`, `price`, `colors`, `brand`, and `platform`. If there are no matching listings, it returns an empty list `[]`.

**What happens if it fails or returns nothing:**
If no listings match, the tool returns `[]`. The agent should stop the workflow, store an error message in the session, and tell the user to try a broader description, higher budget, or removing the size filter.

---

### Tool 2: suggest_outfit

**What it does:**
Suggests one or more outfit combinations using the selected thrifted item and the user's existing wardrobe. If the wardrobe contains items, it creates personalized outfit recommendations. If the wardrobe is empty, it provides general styling advice.

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `new_item` (dict): The thrifted clothing item selected from the search results.
- `wardrobe` (dict): A wardrobe dictionary containing an `items` list of the user's existing clothing pieces.


**What it returns:**
Returns a non-empty string containing outfit suggestions. The suggestions should incorporate the selected thrifted item and, when possible, specific items from the user's wardrobe.

**What happens if it fails or returns nothing:**
If the wardrobe is empty, the tool returns general styling advice rather than an error. If an outfit cannot be generated, the agent informs the user and provides a basic recommendation based on the item's category and style.

---

### Tool 3: create_fit_card

**What it does:**
Generates a short, social-media-style caption describing the outfit and thrifted item. The caption should feel natural, casual, and shareable.

**Input parameters:**
- `outfit` (str): The outfit suggestion returned by `suggest_outfit`.
- `new_item` (dict): The thrifted item selected from the listings.

**What it returns:**
Returns a 2–4 sentence caption that references the thrifted item, its price, platform, and overall outfit vibe.

**What happens if it fails or returns nothing:**
If the outfit string is empty or incomplete, the tool returns a descriptive error message instead of raising an exception. The agent displays the error and stops the workflow.

---

### Additional Tools (if any)

<!-- Copy the block above for any tools beyond the required three -->

---

## Planning Loop

**How does your agent decide which tool to call next?**
The agent uses the current session state to decide which tool to call next. It does not call all tools automatically if an earlier step fails.

1. The agent starts with the user's query and extracts the item description, size, max price, and wardrobe information.
2. The agent first calls `search_listings(description, size, max_price)`.
3. After search runs, the agent checks whether any listings were returned.
   - If `search_results` is empty, the agent stores an error message in `session["error"]`, sets `session["fit_card"] = None`, and returns early. It does not call `suggest_outfit` or `create_fit_card`.
   - If listings are found, the agent stores the full list in `session["search_results"]`, selects the first result, and stores it in `session["selected_item"]`.
4. Next, the agent calls `suggest_outfit(session["selected_item"], wardrobe)`.
5. The outfit suggestion is stored in `session["outfit_suggestion"]`.
6. If the outfit suggestion is empty or contains an error, the agent stores an error message in `session["error"]` and returns early.
7. If the outfit suggestion is valid, the agent calls `create_fit_card(session["outfit_suggestion"], session["selected_item"])`.
8. The fit card is stored in `session["fit_card"]`.
9. The agent returns the full session with the selected item, outfit suggestion, fit card, and any error message.

The loop is conditional because each step depends on what happened in the previous step. For example, if `search_listings` returns no results, the agent stops instead of continuing with empty data.

---

## State Management

**How does information from one tool get passed to the next?**
The agent stores information in a session dictionary while it runs. Each tool saves its output into the session so the next tool can use it without asking the user to re-enter the same information.

The session tracks:
- `session["query"]`: the user's original request
- `session["search_results"]`: the list of listings returned by `search_listings`
- `session["selected_item"]`: the first/best listing selected from the search results
- `session["wardrobe"]`: the wardrobe data used for styling
- `session["outfit_suggestion"]`: the outfit text returned by `suggest_outfit`
- `session["fit_card"]`: the caption returned by `create_fit_card`
- `session["error"]`: an error message if something goes wrong

The output from `search_listings` becomes the input for `suggest_outfit`. Then the output from `suggest_outfit` becomes the input for `create_fit_card`.
---

## Error Handling

For each tool, describe the specific failure mode you're handling and what the agent does in response.

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | |
| suggest_outfit | Wardrobe is empty | |
| create_fit_card | Outfit input is missing or incomplete | |

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | Store an error in `session["error"]`, return early, and tell the user to try a broader description, higher budget, or removing the size filter. |
| suggest_outfit | Wardrobe is empty | Still return general styling advice based on the selected item instead of crashing. |
| create_fit_card | Outfit input is missing or incomplete | Return a descriptive error message and do not crash the agent. |

---

## Architecture

<!-- Draw a diagram of your agent showing how the components connect:
     User input → Planning Loop → Tools (search_listings, suggest_outfit, create_fit_card)
                                                                          ↕
                                                                   State / Session
     Show what triggers each tool, how state flows between them, and where error paths branch off.
     Use ASCII art or a Mermaid diagram (https://mermaid.js.org/syntax/flowchart.html).
     Do NOT embed an image — graders need to read your diagram directly in the file;
     an embedded image or screenshot cannot be evaluated.
     You'll share this diagram with an AI tool when asking it to implement
     the planning loop and each individual tool. -->
```text
User query
    |
    v
Planning Loop
    |
    |-- calls search_listings(description, size, max_price)
    |       |
    |       |-- if results = []
    |       |       |
    |       |       v
    |       |   session["error"] = "No listings found..."
    |       |   return session
    |       |
    |       |-- if results found
    |               |
    |               v
    |          session["search_results"] = results
    |          session["selected_item"] = results[0]
    |
    |-- calls suggest_outfit(session["selected_item"], wardrobe)
    |       |
    |       |-- if wardrobe is empty
    |       |       |
    |       |       v
    |       |   return general styling advice
    |       |
    |       v
    |   session["outfit_suggestion"] = outfit text
    |
    |-- calls create_fit_card(session["outfit_suggestion"], session["selected_item"])
            |
            |-- if outfit is empty
            |       |
            |       v
            |   return descriptive error message
            |
            v
        session["fit_card"] = caption text

Final output:
- selected listing
- outfit suggestion
- fit card
- error message if something failed
---

## AI Tool Plan

<!-- For each part of the implementation below, describe:
     - Which AI tool you plan to use (Claude, Copilot, ChatGPT, etc.)
     - What you'll give it as input (which sections of this planning.md, your agent diagram)
     - What you expect it to produce
     - How you'll verify the output matches your spec before moving on

     "I'll use AI to help me code" is not a plan.
     "I'll give Claude my Tool 1 spec (inputs, return value, failure mode) and ask it to implement
     search_listings() using load_listings() from the data loader — then test it against 3 queries
     before trusting it" is a plan. -->

**Milestone 3 — Individual tool implementations:**
I will use Claude to help implement each tool one at a time. For `search_listings`, I will give Claude the Tool 1 section of this planning document and ask it to implement the function in `tools.py` using `load_listings()` from `utils/data_loader.py`. I will verify the code by checking that it filters by description, size, and max price, returns a list of dictionaries, sorts matches by relevance, and returns `[]` when no listings match.

For `suggest_outfit`, I will give Claude the Tool 2 section and ask it to implement the function using the Groq client already provided in `tools.py`. I will verify that it handles both a normal wardrobe and an empty wardrobe. I will test it with `get_example_wardrobe()` and `get_empty_wardrobe()`.

For `create_fit_card`, I will give Claude the Tool 3 section and ask it to implement the function using the selected item and outfit suggestion. I will verify that it returns a caption-style string, mentions the item naturally, and returns a clear error message if the outfit input is empty.

**Milestone 4 — Planning loop and state management:**
I will use Claude to help implement `run_agent()` in `agent.py`. I will provide the Planning Loop, State Management, Error Handling, and Architecture sections from this planning document. I expect Claude to produce code that calls tools conditionally, stores tool outputs in the session dictionary, and returns early when search results are empty.

Before trusting the generated code, I will check that the agent does not call all three tools regardless of the results. I will test one successful query and one no-results query to confirm that state is passed correctly between tools and that the session dictionary updates as described in the specification.
---

## A Complete Interaction (Step by Step)

Write out what a full user interaction looks like from start to finish — tool call by tool call. Use a specific example query.

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1:**
<!-- What does the agent do first? Which tool is called? With what input? -->
The agent starts by identifying the item request from the user query. It calls:
search_listings(description="vintage graphic tee", size=None, max_price=30.0)
**Step 2:**
<!-- What happens next? What was returned from step 1? What tool is called now? -->
search_listings returns a list of matching listing dictionaries. The agent stores the full list in:
session["search_results"] = results
Then it selects the first/best result and stores it in:
session["selected_item"] = results[0]
**Step 3:**
<!-- Continue until the full interaction is complete -->\
The agent calls:
suggest_outfit(session["selected_item"], wardrobe)
The wardrobe may come from get_example_wardrobe() during testing. The outfit suggestion is stored in:
session["outfit_suggestion"] = outfit_suggestion
**Step 4:**
The agent calls:
create_fit_card(session["outfit_suggestion"], session["selected_item"])
The returned caption is stored in:
The returned caption is stored in:
**Final output to user:**
<!-- What does the user actually see at the end? -->
The user sees the selected thrift listing, an outfit suggestion using their wardrobe, and a short shareable fit card caption. If no listings were found, the user instead sees a helpful message suggesting that they broaden the search, increase the budget, or remove the size filter.