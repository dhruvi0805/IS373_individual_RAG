# Dopamenu | IS 373 Review 1 — Plan

## 2. RAG app + technical research

Dopamenu helps students choose an activity that fits their interests, available time, and budget. A reviewer will be able to see which source cards supported a recommendation. The first hosted version is planned around a fictional student profile, a curated activity catalog, and manually entered limits.

### Research review index

- [Research sources and comparison](#research-sources-and-comparison)
- [Permitted corpus and import](#permitted-corpus-and-import)
- [Proposed stack and hosting](#proposed-stack-and-hosting)
- [Source-to-answer trace](#source-to-answer-trace)
- [Planned evaluation](#planned-evaluation)
- [Changed, checked, and next](#changed-checked-and-next)

### Research sources and comparison

| Research source | Finding and implication |
| --- | --- |
| [Lewis et al., Retrieval-Augmented Generation](https://arxiv.org/abs/2005.11401) | RAG conditions generation on retrieved evidence. Dopamenu will pass the shortlisted records to the model and link the answer to those records. |
| [pgvector documentation](https://github.com/pgvector/pgvector) | PostgreSQL can store vector embeddings, rank them by cosine distance, and filter on ordinary columns. Dopamenu can rank by intent after applying numeric limits. |
| [Neo4j graph database concepts](https://neo4j.com/docs/getting-started/appendix/graphdb-concepts/) | A property graph represents entities as nodes and their connections as typed relationships. This could support questions that traverse several links between interests, activities, and places. |

| Approach | Proposed representation and import | Tradeoff for Dopamenu |
| --- | --- | --- |
| Keyword search with filters | Search titles and descriptions from normalized activity cards after applying time and budget filters. | Transparent baseline. A query such as “make something with my hands” may share few words with “paper collage.” |
| PostgreSQL + pgvector | Store card/section text and embeddings with source IDs and duration and cost fields. | Supports paraphrases and explicit constraints in one database. Similarity still needs evaluation. |
| Neo4j property graph | Import validated Activity, Interest, and Venue nodes and edges such as Activity HAS_INTEREST Interest and Activity TAKES_PLACE_AT Venue. Use stable IDs to merge duplicate nodes and edges. | Explicit relationships help with multi-hop questions, but curated edges and graph queries add work. |
| Vector + graph | Use vectors to find candidates and traverse graph relationships by shared activity ID. | Useful if later questions need both semantic matching and multi-hop evidence. Requires synchronization and more evaluation. |

**Decision** — Use PostgreSQL with pgvector and structured filters. Keep keyword search as the comparison baseline. The first questions require semantic matching, time and budget limits, and clear citations. The assumption that vectors handle paraphrases better will be tested on the same corpus and filters.

### Permitted corpus and import

The following original, fictional records are permitted for the class demo. They are the starter source manifest, version 1. Their text is shown here for review. The proposed import will extract them into Markdown cards. Prices are estimated USD costs for the example activities, assuming the listed household supplies are available.

| Source ID | Kind | Source text and fields |
| --- | --- | --- |
| PREF-01 | Fictional profile | Enjoys small-group craft activities and making things with everyday materials. |
| ACT-01 | Activity | Make a paper collage by cutting and arranging old magazines on scrap paper. Solo, 30 minutes, $0. |
| ACT-02 | Activity | Host a two-person sketch swap. Take turns drawing household objects with pencil and paper. Social, 15 minutes, $0. |
| ACT-03 | Activity | Take a neighborhood photo walk using a phone to photograph repeating shapes. Solo, 25 minutes, $0. |
| ACT-04 | Activity | Make a pinch pot from air-dry clay. This example includes a $12 clay purchase. Solo, 40 minutes, $12. |

Use one chunk per short activity card and one chunk per titled preference section. Keep each heading with its text. During a manual import, validate required IDs and numeric fields, normalize whitespace, and deduplicate by source ID and content hash. Store source ID, original text, record version, permission, import timestamp, and numeric metadata. Re-embed changed text and replace superseded chunks. Technical references above support engineering decisions and are separate from the recommendation corpus. Ticketmaster and FindLocal would require checking reuse, caching, and attribution terms before any import. Calendar availability and rating history are future structured context.

### Proposed stack and hosting

| Layer | Plan and reason |
| --- | --- |
| React interface | Accept a plain-language request plus explicit duration and budget fields. Show the recommendation, its source links, and retrieved record IDs for review. |
| Python FastAPI | Handle validation, import, query embedding, database retrieval, and generation server-side. Keep model and database credentials server-side. |
| PostgreSQL + pgvector | Store source metadata and vectors together. Filter activity rows by time and cost, then rank eligible rows by cosine distance. Retrieve preference evidence separately when personalization is relevant. Start with exact search for the small corpus. |
| Embedding and generation APIs | Use one approved embedding model for both cards and queries. Record its model name, dimensions, and generation date. Give a hosted language model only the shortlisted evidence and require source IDs that the API can verify against the shortlist. |
| Docker on the existing DigitalOcean droplet | Serve the frontend and API over HTTPS through Traefik. Check memory, disk space, and model API limits before choosing whether to run Postgres on the droplet or use managed Postgres. Persist the database outside the app container. Precompute card embeddings and cap the context sent to the model to limit resource use. |

### Source-to-answer trace

**Illustrative walkthrough.** ACT-01 becomes one chunk containing its title and description. Store its embedding alongside source ID ACT-01, duration 30, and cost 0. For “I want to make art from scraps with 45 minutes and $0,” the API would filter by time and cost and embed the intent. If ACT-01 ranks among the eligible matches and its text supports the request, the model could answer “Make a paper collage using old magazines. It takes 30 minutes and costs $0 [ACT-01].” The interface would resolve the citation to the original source text. If no eligible record supports the request, the answer would state that the catalog has no supported match.

### Planned evaluation

| User story | Evaluation question or action | Observable pass condition |
| --- | --- | --- |
| Find a named activity | “Can I make a paper collage in 45 minutes for $0?” | ACT-01 appears in the retrieved shortlist and the answer cites it with accurate time and cost. |
| Find an activity by intent | “I want to make art out of scraps without paying.” Use a 45-minute limit and a $0 budget. | ACT-01 ranks first. Record the first result or no match for each retrieval method. |
| Enforce hard limits | “Can I make a pinch pot in 30 minutes for $5?” | ACT-04 is excluded. The answer explains that there is no eligible pinch-pot activity. |
| Explain a personal match | “What can I make with one other person in 20 minutes for free?” | ACT-02 is cited. Any claim about the profile liking small-group crafts also cites PREF-01. |
| Handle missing evidence | “Find a free underwater welding class in Newark this weekend.” | State that the catalog lacks a supported match and return no fabricated class or link. |

For the paraphrase comparison, apply the same 45-minute and $0 filters to both methods. ACT-01, ACT-02, and ACT-03 qualify, so a top-three check would include every eligible vector result regardless of order. Score both methods at rank one. For the keyword baseline, rank by the number of distinct normalized content words shared by the query and the card title and description, with source ID as the tie-breaker. Return no keyword match when overlap is zero. Count a hit only when ACT-01 ranks first. If both methods hit or both miss, this query does not distinguish them. Keep the top-three shortlist for the other checks. Log the retrieved IDs and compare expected evidence, constraint violations, and citation accuracy. For the paraphrase question, also compare embedding the title alone with embedding the title and description. An ACT-01 result below rank one or absent altogether is a miss for the paraphrase check. An answer that misstates its duration despite retrieving ACT-01 indicates a generation or validation error.

### Changed, checked, and next

- **Changed** — Scoped the original Dopamenu specification to a curated catalog, fictional preferences, manual time and budget entry, and an inspectable recommendation trace.
- **Checked** — Reviewed the Dopamenu specification and the linked pgvector and Neo4j references for the database comparison. The trace and evaluation table are design examples. Runtime results are pending.
- **Peer feedback and decision** — The reviewer traced the $0 case and pointed out that a top-three cutoff cannot measure rank with three eligible cards. The paraphrase check now compares the first result from each method under the same limits. See [peer review #1](https://github.com/dhruvi0805/IS373_individual_RAG/issues/1).
- **Next** — At Build, confirm hosting resources, import the corpus, and capture reproducible retrieval and answer results.
