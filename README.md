# FTEC5660 Homework 3: Fisher's 15-Point Investment Agent

Build a LangChain agent that researches a company with tools and applies the
framework from Philip Fisher's *Common Stocks and Uncommon Profits* (Part One,
Chapter 3 "What to buy: the fifteen points to look for in a common stock", and
Chapter 4 "What to buy: applying this to your own needs"). You choose 3 of
Fisher's 15 points and decide yourself what PASS means for each. For every
company your agent gives a PASS/FAIL verdict on your 3 points and a short
summary.

The companies are fictional, so the model cannot answer from memory. Unlike a
pile of reports, the evidence is spread over three sources, and your agent has
to go and get it with tools, the way Fisher's "scuttlebutt" method sends an
investor to talk to customers, competitors, suppliers and staff.

## Data and tools

`public_test/` has one folder per company. Your code receives four tools per
company (built by the provided code in `hw3.py`):

| Tool | What it gives you |
| --- | --- |
| `list_documents()` | the company's six documents and their dates |
| `read_document(name)` | one document: the 2025 letter to shareholders, four earnings calls (Q3 2025 to Q2 2026) and a news file (July 2025 to July 2026) |
| `sql_query(query)` | one read-only SQLite query on the company's database; at most 50 rows come back, so use `SUM`, `GROUP BY` and `WHERE` |
| `interview(role, topic)` | what one person says about the company; at most **10 interviews per company** |

The documents are what management says. Management's own claims (such as "we
have never cut research" or "our people are our greatest strength") are not
evidence on their own: check them against the database and the interviews.

**Database tables** (amounts in USD thousands; dates `YYYY-MM-DD`, months
`YYYY-MM`, quarters like `2025Q3`; the fiscal year is the calendar year):

| Table | Columns |
| --- | --- |
| `company` | name, industry |
| `industry_medians` | industry, revenue_growth_2y_pct, new_product_share_pct, new_product_sales_per_rd, operating_margin_pct (one row per industry; use the row for the company's own industry) |
| `orders` | order_id, order_date, product, customer, amount, note (about 3,000 to 4,000 orders from January 2023 to June 2026; revenue is the sum of order amounts) |
| `products` | product, launch_date |
| `monthly_costs` | month, cost_of_sales, selling_admin, research_development |
| `quarterly_cash` | quarter, operating_cash_flow, capex, cash_end |
| `capex_plan` | year, planned_capex (2026 to 2028) |
| `returns` | month, units_shipped, units_returned |
| `insider_trades` | trade_date, person, title, side (`buy` or `sell`), shares, price |

**Terms used in the data.**

| Term | Meaning |
| --- | --- |
| FY2025 | Fiscal year 2025. Here the fiscal year is the calendar year, so FY2025 is January to December 2025. |
| Quarter | Three months of the year: Q1 is January to March, Q2 April to June, Q3 July to September and Q4 October to December. The database writes `2025Q3`; the documents write "third quarter 2025". |
| Amounts | The database is in **USD thousands** (676,100 means US$676.1 million). The documents use USD millions. |
| Revenue | Money from sales: the sum of `amount` in `orders`. |
| Cost of sales | The cost of making the products that were sold. |
| Selling and admin | The cost of selling and of running the company (sales staff, offices, management). |
| R&D | Research and development spending, mainly on new products. |
| Operating margin | Operating profit as a share of revenue: (revenue − cost of sales − selling and admin − R&D) ÷ revenue. |
| Operating cash flow | Cash that the day-to-day business brings in during a quarter. |
| Cash at the end of a quarter | The cash the company holds on the last day of that quarter (`cash_end`). |
| Capex | Capital expenditure: spending on factories, machines and other long-term assets. `capex_plan` holds the planned capex for 2026 to 2028. |
| Return rate | Units returned ÷ units shipped, from the `returns` table. |
| Industry median | The middle value for companies in the same industry, from `industry_medians`. |
| Insider | A director or senior manager of the company. `insider_trades` lists the shares they bought (`buy`) or sold (`sell`). |
| Earnings call | A meeting each quarter where management presents the quarter's results to investors. Each call quotes that quarter's revenue. |

**Interviews.** Roles: `customer`, `current_employee`, `former_employee`,
`competitor`, `supplier`. Topics: `sales_team`, `working_conditions`,
`top_management`, `decision_making`, `competitive_edge`. Only some people know
about each topic; the others say they can't comment, and that answer still
uses up one interview.

| Topic | Who knows about it |
| --- | --- |
| `sales_team` | customer, competitor |
| `working_conditions` | current_employee, former_employee |
| `top_management` | former_employee, supplier |
| `decision_making` | supplier, customer |
| `competitive_edge` | competitor, customer |

People who know about a topic always agree with each other, so one good answer
is enough.

## Choose your 3 points and your rules

Pick any 3 of Fisher's 15 points and write them in `CHOSEN_POINTS` near the
top of `hw3.py`, e.g. `CHOSEN_POINTS = [1, 7, 15]`. The full questions are in
`FISHER_POINTS` in `hw3.py` and in the book.

| # | Fisher's point | Where the evidence is |
| --- | --- | --- |
| 1 | Market potential for sales growth | database |
| 2 | New products as current lines mature | database |
| 3 | Effective research and development | database |
| 4 | Above-average sales organization | interviews (`sales_team`) |
| 5 | Worthwhile profit margin | database |
| 6 | Maintaining or improving margins | database |
| 7 | Labour and personnel relations | interviews (`working_conditions`) |
| 8 | Executive relations | interviews (`top_management`) |
| 9 | Depth of management | interviews (`decision_making`) |
| 10 | Cost analysis and accounting controls | earnings calls, checked against the database |
| 11 | Advantage over competitors | interviews (`competitive_edge`) |
| 12 | Long-range outlook on profits | database |
| 13 | Growth without dilutive equity financing | database |
| 14 | Management talks freely about troubles | earnings calls, checked against the database |
| 15 | Management of unquestionable integrity | news file, checked against the database |

For each of your points, **you design the rule**: which evidence to use, what
to compute or whom to ask, and where the line between PASS and FAIL lies. PASS
means the company looks favourable on that point, as Fisher would see it.

We grade your verdicts against our own answer key. It uses one fixed rule per
point, which we do not publish, and the data never puts a company close to
the line of that rule, so a sensible reading of Fisher usually gives the same
answer. **Test your rules on the public answers**:
`public_test/ground_truth.json` gives the expected verdict on all 15 points
for each of the 20 public companies, and every point fails in 6 of them. If
your rule disagrees with the public answers, rethink it.

**Tip: calculate with tools, not in your head.** Do every sum, ratio and
comparison in SQL or Python, including the number of days between two dates
(SQLite's `julianday()` gives a date as a number of days).

## Student task

Fill in the three places in `hw3.py` marked `### YOUR CODE HERE`. You may add
imports, constants and helper functions above them, but do not change the
provided code below them:

- `CHOSEN_POINTS` lists your 3 points.
- `build_agent()` creates your model and anything else you need once.
- `analyze_companies(agent, companies)` receives
  `{company_folder: [list_documents, read_document, sql_query, interview]}`
  and returns, for every company,
  `{"points": {1: "PASS", 7: "FAIL", 15: "PASS"}, "summary": "..."}` (here with
  `CHOSEN_POINTS = [1, 7, 15]`).

Use the course techniques that fit your points: tool use for the database and
the interviews, planning which interviews to spend, routing each point to the
right source, prompt chaining, parallel calls, reflection to check your
answers, or a combination. Your code must read company data **only through
the four tools**; do not open `company.db` or `interviews.dat` yourself. Do
not hard-code folder names, company names or the public answers.

## Setup and public test

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
echo "DEEPSEEK_API_KEY=your_key_here" > .env
python3 hw3.py --data-folder public_test
```

Use your own DeepSeek API key (from https://platform.deepseek.com). The
required model is `deepseek-v4-flash`. We grade with our own key, so never put
a key in your code; `.env` must stay out of git.

The program creates `results.csv` with the columns `company`, `points`,
`points_correct`, `point_errors`, `interviews_used` and `summary`, and prints
the point score on your 3 points (the percentage after "balanced"). The point
score is the average of two shares: the share of truly passing verdicts that
you marked PASS and the share of truly failing verdicts that you marked FAIL.
For example, 90% and 60% give (90% + 60%) / 2 = 75%, and marking every point
PASS gives (100% + 0%) / 2 = 50%. Once `CHOSEN_POINTS` is set, the starter
runs before you add any API code (it returns no verdicts, so it scores 0%).

`public_test/ground_truth.json` gives the expected verdicts on all 15 points
for each public company. Grading uses a private set of about 20 unseen
companies with the same tools and the same answer key rules, but different
names and wording, and scores your 3 points only.

## Homework 3 solution:
> to students: For this implementation, my starting question was who would actually be able to answer each of Fisher’s questions. A customer can describe the help received from a sales team, an employee can describe working conditions, and a competitor can identify capabilities that are difficult to match. This led me to Points 4, 7 and 11.

I also tried the combination of Points 3, 7 and 14, which produced lower public-test accuracy. I retained the original combination because its evidence sources and decision rules were more straightforward to check. I would need to examine the individual mismatches before attributing the lower score to a particular rule or model behaviour.

### My 3 points and rules
> for each point: 

Point 4: Above-average sales organisation

I chose this point to distinguish having salespeople from having a sales organisation that adds value for customers. Sending a quotation is useful, but it says relatively little about whether the team understands the customer’s problem.

My rule is PASS when a knowledgeable interviewee describes useful advice, an understanding of customer needs or effective sales support. It is FAIL when the evidence describes a mainly price-list-based service, poor advice, or customers turning to rivals for better support.

The agent interviews a customer about sales_team. A competitor is the alternative source.

Point 7: Labour and personnel relations

This point gives employees a place in the assessment. Their account can reveal whether the company’s treatment of staff supports the positive picture management might present.

My rule is PASS when the interview supports strong personnel relations through fair treatment, good working conditions, constructive communication or employee trust. It is FAIL when the account describes persistent poor treatment, unsafe conditions, serious dissatisfaction or hostile labour relations.

The primary source is a current_employee answering questions about working_conditions. A former_employee is available if further evidence is needed.

Point 11: Advantage over competitors

I chose this point because satisfactory current performance does not explain why customers should continue buying from the company. I wanted the assessment to identify a specific reason for preferring its business over an alternative.

My rule is PASS when the interview identifies a meaningful competitive advantage, such as specialist expertise, differentiated products, customer loyalty or switching costs. It is FAIL when the offering is easily interchangeable with rivals or customers choose mainly on price without another meaningful advantage.

The agent interviews a competitor about competitive_edge, with a customer as the alternative source.
### Design
```mermaid
flowchart TD
    A["Initialize ChatDeepSeek once<br/>deepseek-v4-flash"] --> B["Start a company workflow<br/>Up to 4 companies concurrently"]
    B --> C["Apply the fixed interview plan"]

    subgraph Interviews["Retrieve evidence through the interview tool"]
        D["Point 4: Sales organisation<br/>customer / sales_team"]
        E["Point 7: Personnel relations<br/>current_employee / working_conditions"]
        F["Point 11: Competitive advantage<br/>competitor / competitive_edge"]
    end

    C --> D
    C --> E
    C --> F

    D --> G["Collect usable interview answers<br/>Use an alternative person if unavailable"]
    E --> G
    F --> G

    G --> H["Evaluate evidence against the rules<br/>Return verdict, reason and exact quote"]
    H --> I{"Valid label and<br/>quote found in the source?"}

    I -->|Yes| J["Keep the validated assessment"]
    I -->|No| K{"Model attempts remaining?<br/>Maximum 3 calls per company"}

    K -->|Yes| L["Repair unresolved points only"]
    L -->|Format or quote issue| H
    L -->|More evidence requested| M["Consult the alternative knowledgeable person<br/>Only if not already interviewed"]
    M --> H

    K -->|No| N["Keep completed assessments<br/>Mark unresolved points as FAIL fallbacks"]

    J --> O["After processing finishes:<br/>assemble three verdicts and supporting summary"]
    N --> O
    O --> P["Provided runner writes results.csv"]

    classDef setup fill:#E2E8F0,stroke:#64748B,color:#0F172A;
    classDef evidence fill:#DBEAFE,stroke:#3B82F6,color:#1E3A8A;
    classDef model fill:#EDE9FE,stroke:#8B5CF6,color:#4C1D95;
    classDef repair fill:#FEF3C7,stroke:#F59E0B,color:#78350F;
    classDef output fill:#D1FAE5,stroke:#10B981,color:#064E3B;
    classDef fallback fill:#FEE2E2,stroke:#EF4444,color:#7F1D1D;

    class A,B,C setup;
    class D,E,F,G,M evidence;
    class H,I model;
    class K,L repair;
    class J,O,P output;
    class N fallback;
```

The workflow separates evidence collection from judgment. The interview roles and topics are fixed before the model sees a company’s evidence. This avoids spending the interview budget on people who cannot answer the selected questions. The agent normally obtains three interviews, one per point; an alternative knowledgeable person is consulted when the first answer is unavailable or the model identifies insufficient evidence. LangChain’s ChatDeepSeek, using deepseek-v4-flash, then evaluates the retrieved material against the written rules. Each response must contain a verdict, an explanation and an exact supporting quote. Python checks the response structure, the allowed labels and whether the quote occurs in the interview. Accepted assessments are kept while unresolved points receive up to two further model attempts. Each company can use at most six interviews and has a 180-second analysis limit. Up to four companies are processed concurrently, with separate evidence for each. The provided runner receives the completed results and writes results.csv.

There are two distinctions in this design that matter to the interpretation of its output. First, finding a quote in an interview verifies its source, but does not prove that the model interpreted it correctly. Second, the code uses a labelled FAIL fallback when it cannot obtain a validated assessment. That fallback preserves a complete output structure, but should be read as a processing failure rather than evidence that the company performs badly.
### Results
> The public-test run produced the following output: Processed 20 companies. Wrote results.csv. Your points: 4, 7, 11. Point accuracy: 60/60 (balanced: 100.0%).
All 60 verdicts agreed with the public answers. There were therefore no disagreement cases to explain for this configuration. Both the true-PASS and true-FAIL cases were identified correctly, producing a balanced point accuracy of 100.0%.

This result supports the selected rules on the public dataset. It does not establish how consistently the model will interpret unfamiliar wording. The main remaining checks are performance on unseen companies and consistency across repeated runs. I have not reported private-test results or repeated-run statistics because those are not available.
