## Data Structures and Algorithms 2025 Semester Project


# VHS Course Matcher

**VHS Course Matcher** is a web application that helps users interact with the **Volkshochschule Berlin (VHS)** course database in English. It helps users discover and match with courses based on their interests, budget, and identity preferences, while also aligning with the platform's goals—like filling under-enrolled or sponsored courses and promoting diversity in course participation.

---

## Features

- Search courses by keyword (supports English and German)
- Automatic query translation and language detection
- Budget-aware matching and fuzzy search ranking
- Dual ranking: User preferences and platform priorities
- Consensus ranking using the Kemeny-Young algorithm
- Smart platform matching: prioritizes courses that need participants, maintain gender balance, and match user-identified target groups
- Team page to meet the creators

---

## Setup & Installation

### 1. Clone the Repository

```bash
git clone https://github.com/olivepol/DSA2025_birds.git
cd flask_app
```

### 2. Create a Virtual Environment

```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

### 3. Install Dependencies

```bash
pip install -r requirements.txt
```

> Alternatively, use the minimal production-ready list:

```txt
Flask==3.1.0
Werkzeug==3.1.3
pandas==2.2.3
numpy==2.2.4
langdetect==1.0.9
deep-translator==1.11.4
RapidFuzz==3.13.0
PuLP==3.1.1
```

### 4. Run the Application

```bash
cd flask_app
python flask_app.py
```

Then visit <http://127.0.0.1:5000> in your browser.

---

## Matching Algorithm & Design Rationale

### 1. Language Detection & Translation

User search queries can be written in English or German. The `langdetect` package identifies the input language. If not in German, the query is translated using Google’s deep translator (via `deep-translator`) to ensure consistent matching with the primarily German-language dataset.

### 2. Fuzzy String Matching

After experimenting with Levenshtein edit‑distance (too typo‑sensitive) and sentence‑transformer embeddings (computationally heavy, inconsistent for local German terms), we settled on RapidFuzz’s partial and token‑set ratios for their balance of speed and robustness.

**How we match**

1. The query is lower‑cased and split on whitespace.
2. If it contains *more than one* token we use `fuzz.partial_ratio`; otherwise we use `fuzz.token_set_ratio`.
3. We compute the score against three columns (`course_name_german`, `course_name_translated`, `search_text`).
4. A course is kept if *any* column passes the threshold (75 for partial, 60 for token‑set).

**Complexity overview**

- **`token_set_ratio`** Tokenises both strings into unique sets, builds intersections/differences, then runs a handful of Levenshtein distance checks on much shorter strings. Practical complexity ≈ O(n log n + m log m).
- **`token_set_partial_ratio`** Same preprocessing but uses `partial_ratio` to allow substring alignment; big‑O remains O(n log n + m log m) with slightly larger constants.

### 3. Budget filter

If the user specifies a budget, we filter to courses within ±30 % of it. We then compute a price penalty:

- The farther from the budget, the higher the penalty
- Final score = 65 % fuzzy match + 35 % budget closeness

### 4. Platform Preference Ranker

The platform uses additional metadata about each course:

- Number of participants enrolled vs. minimum required
- Remaining occupancy
- Gender balance (targeting ≤ 80 % one-gender dominance)
- Binary boosts for:
  - Sponsored courses
  - Courses targeting user-identified groups (e.g., women, migrants)

These are used to assign a platform-side score with numeric and boosting components.

### 5. Kemeny‑Young Aggregation

We blend the user‑centric and platform‑centric rankings with a Kemeny‑Young consensus solved by Integer Linear Programming (ILP).

**Core idea**

- For every pair of courses (*i, j*) we decide whether *i* should appear before *j* (binary variable x_{i,j}).
- The ILP minimises the total pairwise disagreement with the two input rankings.

**Problem size**

| Component                | Order of magnitude  |
| ------------------------ | ------------------- |
| Pairwise variables       | O(n²)  (n choose 2) |
| Transitivity constraints | O(n³)  (n choose 3) |

With the list capped at **n ≤ 20**, this translates to 190 variables and 1 140 constraints—well within the capabilities of standard MILP solvers.

If we ever need to rank larger sets, we can switch to approximation heuristics (greedy merge, local search) without changing the surrounding code.

---

## Data preparation

A separate utility pipeline (now archived) was used to build the dataframe consumed by the matcher.

1. **Raw source** — We downloaded the complete course schedule from the Volkshochschule Berlin (VHS) from Berlin Open Data as of **March 2025**. The data was in the form of a JSON file.
2. **Translation** — Columns were translated, and course names were machine‑translated to English via Google Translate and kept in parallel columns to the German version.
3. **Feature engineering and cleaning** — We added working columns such as:
   - `prop_occupancy_left` and `prop_minimum_to_reach`
   - binary flags for sponsorship and target groups
   - gender‑gap measures against the 80 % rule
   - Already fully enrolled courses were removed
4. **Serialization** — The final tidy dataset is stored as `Processed_data_for_app.pkl` inside `flask_app/app/data/`.

All scripts and the original zipped JSON dump live in:

```
DSA2025_birds/depreciated_data_prep/
```

They remain available for reproducibility or for refreshing the dataset in future.

---

## Alternative model

Early in development we maintained an experimental branch located at

```
DSA2025_birds/depreciated_flask_app_with_transformer_and_embeddings/
```

That variant used **semantic embeddings** rather than fuzzy string matching.

1. **Levenshtein edit‑distance** was our first attempt but proved too sensitive to minor typos and accent marks in a multilingual setting.
2. We then built a pipeline around **Sentence‑Transformers** to encode both user queries and course descriptions into vectors and compared them with cosine similarity.
   - Pros: captured deeper intent beyond exact wording.
   - Cons: large model weights (> 400 MB), slower inference, and inconsistent results for regional German course jargon.
3. We ultimately moved to **RapidFuzz** (`token_set_ratio`, `partial_ratio`) which was lighter, faster, and more robust to spelling variation across English and German.

For reference, the deprecated code—including preprocessing, embedding generation, and cosine‑similarity ranking—remains in the folder above if you wish to explore or benchmark it.

---

## Project Structure
```
flask_app/
│-- flask_app.py
│-- test_algorithm.py
│-- static/
│-- templates/
│   ├-- layout.html
│   ├-- home.html
│   ├-- courses.html
│   └-- about.html
│
├-- app/
│   ├-- __init__.py
│   ├-- assets_loader.py
│   ├-- processor.py
│   ├-- data/
│   │   └-- Processed_data_for_app.pkl
│   └-- models/
│       ├-- __init__.py
│       ├-- consensus_ranker.py
│       ├-- matching.py
│       └-- platform_ranker.py
```
- The test cases for this project are located in `test_algorithm.py`.
---

## Authors
Developed for the **DSA 2025** course by **Team Birds**

- Elena Lopez  
- Fanus Ghorjani  
- Hanna Getachew  
- Matheus Galiza  
- Oliver Pollex  
- Padmavathi Reddy  
- Sauradeep Bhattacharyya (see bwlow)
  
**Individual Write-up – Sauradeep Bhattacharyya**
  
For our DSA 2025 project, I was primarily responsible for the algorithm implementation and the
backend integration of the VHS Course Finder application. When we started, we had very limited
context on how algorithms are applied in real-world projects—especially in programming—and how
they differ from more generic deterministic methods. Initially, our approach leaned heavily toward
simple string matching. However, through discussions with our lab instructor and you, we began to
explore a more ambitious framing of the problem.

Instead of just finding course matches through keyword comparison, we reimagined the task as a
ranking problem. We realized that there were two distinct but related preferences at play: one from
the user (what they want, their budget, etc.), and one from the platform (which courses need to be
filled, where gender balance is skewed, etc.). To bridge these, I proposed using the Kemeny-Young
rank aggregation algorithm, which uses integer linear programming (ILP) to generate a
consensus ranking by minimizing pairwise disagreements between two rankings. This formed the core
algorithm of our app.

*Key Contributions*

**1. Kemeny-Young ILP Formulation**
I was responsible for translating this mathematical model into code—writing out the binary
decision variables, antisymmetry and transitivity constraints, and setting up the objective
function that minimizes disagreement. To ensure it remained solvable, I capped the rankings
at 20 entries. I felt this was a strong demonstration of how a theoretical algorithm could
power a real product.

**2. String Matching Logic**
Since the dataset was in German, I implemented a preprocessing step that combined
langdetect and deep-translator to handle multilingual queries. I realized that for optimal string
matching, we needed a split approach: token_set_ratio worked best for one-word queries,
while partial_ratio performed better on multi-word input. I modified our matcher
accordingly, making the match logic dynamic depending on the structure of the input.

**3. Platform-Side Ranking**
The platform had its own goals: prioritize under-enrolled courses, ensure a gender balance,
and boost sponsored offerings. I designed a ranking logic that reflected this. First, courses
were ranked based on numeric variables like occupancy and participant minimums. Then, I
applied a “ranking boost” based on binary categorical indicators (sponsorship, matching user
target groups). The boost was scaled using inter-rank dispersion, so that it meaningfully
shifted course positions.

**4. Initial Flask Integration**
While my teammates worked on refining the interface, I was responsible for setting up the
initial Flask app structure and doing the first level of integration. This included organizing
the codebase into reusable modules, setting up the route logic, and connecting it to
placeholder HTML pages. Coming into this project with almost no experience in Python or
Flask, I was genuinely proud to have debugged issues and made the whole pipeline—from
form submission to ranked course display—work as a coherent app.

**What Could Be Improved**

If I had more time, the next big improvement I’d push for would be
integrating a hosted database with user registration. That would allow us to save user preferences,
record course interest, and feed that data back into our ranking system. It would also let us update
courses in real-time, show trending courses, or even notify users when a class they're interested in is
about to fill. While our current version shows a successful use case of a 3rd party webapp that
integrates open source VHS course data to generate an English interface for users of all backgrounds
to interact and access these courses, further integration was slightly out of reach given our timelines
and our team's early-stage experience with backend systems.

---

