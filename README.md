
# Paper Trail

Paper Trail is an n8n workflow automation project for collecting and analyzing research-paper metadata.
It retrieves papers from Semantic Scholar, extracts structured claims from abstracts, and stores validated records in PostgreSQL.
It also generates author-level contradiction events, topic-level consensus summaries, and pipeline notifications.

## What the project does

Research papers can discuss the same topic while reporting different findings or positions. Paper Trail provides a workflow for collecting paper metadata and organizing selected information for further analysis.

The project uses a master workflow and eight supporting workflows. The master workflow coordinates the stages, while individual workflows handle specific tasks.

## Pipeline architecture

The main workflow follows this sequence:

1. **Database setup:** Checks that the papers table exists and ensures a duration column on the pipeline runs table when that table exists.
2. **Run initialization:** Initializes run-level counters and summary data.
3. **Ingestion:** Searches Semantic Scholar for papers matching three configured topics.
4. **Extraction:** Uses GPT-4o mini to extract structured information from each paper abstract.
5. **Validation:** Checks required fields, allowed stance values, confidence ranges, and methodology categories.
6. **Storage:** Inserts valid records into PostgreSQL and skips conflicting paper IDs.
7. **Contradiction detection:** Groups stored papers by topic and normalized author, then identifies changes in recorded stance between adjacent papers.
8. **Consensus calculation:** Calculates methodology-score-weighted stance percentages for each topic and year.
9. **Notification:** Sends a run summary to a configured webhook.
10. **Run logging:** Records run information in the PostgreSQL pipeline_runs table.

The master workflow also includes separate handling for storage failures and pipeline logging failures.

## Workflow diagram

```text
Manual Master Trigger
        |
        v
Database Setup
        |
        v
Initialize Run
        |
        v
Ingestion
        |
        v
Extraction
        |
        v
Validation
        |
        v
Storage
        |
        v
Storage Successful?
       / \
     Yes  No
      |    |
      v    v
Contradiction  Storage Failure
Detection      Notification
      |
      v
Consensus Engine
      |
      v
Notification
      |
      v
Pipeline Run Logging
      |
      v
Pipeline Complete
```

## Data ingestion

The ingestion workflow uses the Semantic Scholar Graph API:

`https://api.semanticscholar.org/graph/v1/paper/search`

The current configured topics are:

- Coffee consumption and mortality risk
- Red meat consumption and cancer risk
- Screen time and adolescent mental health

For each topic, the workflow requests up to 100 papers from the previous 15 years.

The requested fields include:

- Title
- Abstract
- Authors
- Publication year
- Venue
- Citation count
- Paper ID
- External identifiers
- Publication types

Records without abstracts are excluded. Existing paper IDs are checked against PostgreSQL before downstream processing.

## Information extraction

The extraction workflow uses the OpenAI `gpt-4o-mini` model.

It receives each paper's title and abstract and requests structured JSON containing:

- Claim
- Stance
- Confidence
- Methodology type
- Sample size
- Key limitation
- Normalized authors

The extraction instructions restrict the model to information contained in the supplied abstract. Uncertain fields are assigned neutral or empty values.

The original paper metadata is merged with the extracted fields before validation.

## Validation and methodology scoring

The validation workflow checks:

- Whether a paper ID is present.
- Whether the extracted claim is non-empty.
- Whether stance is one of `supports`, `contradicts`, `neutral`, or `mixed`.
- Whether confidence is between 0 and 1.
- Whether methodology type belongs to the configured categories.

The workflow assigns the following methodology scores:

| Methodology | Score |
|---|---:|
| Randomized trial | 95 |
| Meta-analysis | 92 |
| Longitudinal | 85 |
| Cross-sectional | 60 |
| Animal study | 40 |
| Other | 30 |
| Opinion review | 10 |

Invalid records are returned with validation errors. Valid records receive a methodology score and continue to storage.

## Storage

PostgreSQL is used for persistent storage.

The database setup workflow creates the `papers` table if it does not exist. The master workflow also records pipeline run information.

The storage workflow:

1. Keeps records marked as valid.
2. Maps the fields to the database columns.
3. Inserts records into the `papers` table.
4. Uses the paper ID as the conflict-matching column.
5. Skips records that conflict with existing paper IDs.

## Contradiction detection

The contradiction detection workflow reads stored papers and groups them by topic and normalized author.

Within each group, records are ordered by publication year. The workflow compares adjacent records and creates an event when their recorded stances differ.

The resulting records include the author, topic, paper identifiers, years, and the two stances.

This is a rule-based comparison of extracted stance labels. It does not establish that the papers make logically incompatible claims or that one paper disproves another.

## Consensus engine

The consensus workflow groups records by topic and publication year.

It calculates weighted percentages for four stance categories:

- Supports
- Contradicts
- Neutral
- Mixed

The methodology score assigned during validation is used as the weight.

The resulting summaries are stored in the `consensus_by_year` table. Existing topic-year records are updated through an upsert.

These percentages represent the project's scoring method. They should not be interpreted as established scientific consensus.

## Notifications and run logging

The notification workflow sends a JSON payload containing a message and run summary to a configured webhook.

The master workflow also prepares run information, including:

- Papers fetched
- Papers inserted
- Papers skipped
- Validation errors
- Contradiction summary
- Consensus summary
- Start and finish timestamps
- Run duration

Run information is written to the PostgreSQL `pipeline_runs` table.

## Project structure

```text
Paper-Trail/
│
├── workflows/
│   ├── Paper Trail - Master Pipeline.json
│   ├── Paper Trail - Database Setup.json
│   ├── Paper Trail - Ingestion.json
│   ├── Paper Trail - Extraction.json
│   ├── Paper Trail - Validation.json
│   ├── Paper Trail - Storage.json
│   ├── Paper Trail - Contradiction Detection.json
│   ├── Paper Trail - Consensus Engine.json
│   └── Paper Trail - Notifications.json
│
├── .github/
│   └── workflows/
│       ├── author-audit.yml
│       ├── backup-before-rewrite.yml
│       └── rewrite-history.yml
│
├── DEPLOYMENT.md
├── .gitignore
└── README.md
```

The `.github/workflows/` directory contains GitHub Actions workflows for repository auditing, history backup, and history rewriting. These are separate from the n8n application workflows.

## Setup and execution

### Requirements

The documented deployment uses:

- n8n Cloud
- PostgreSQL, with Neon used as the documented provider
- OpenAI API credentials
- Semantic Scholar Graph API access
- A notification webhook endpoint
- GitHub for version control

### Installation

1. Open your n8n instance.
2. Import all eight JSON workflow files from the `workflows/` directory.
3. Configure the PostgreSQL credential.
4. Configure the OpenAI credential.
5. Configure the notification webhook environment variable.
6. Verify that the master workflow references the correct imported child workflows.
7. Verify that the required database tables and columns exist.
8. Publish the child workflows and then the master workflow.
9. Execute the master workflow for an end-to-end run.

### Required configuration

The notification workflow reads this environment variable:

`PAPER_TRAIL_NOTIFICATION_WEBHOOK_URL`

It is used as the destination for the notification HTTP request.

The PostgreSQL credential must contain the connection details for the configured database, including host, port, database, username, password, and SSL settings.

The OpenAI credential must be configured for the extraction workflow.

Do not commit API keys, passwords, database credentials, or webhook secrets to the repository.

### Database prerequisites

The database setup workflow creates the `papers` table and conditionally adds `duration_ms` to an existing `pipeline_runs` table.

The other workflow queries also reference:

- `pipeline_runs`
- `contradictions`
- `consensus_by_year`

These tables and their required columns must be available in the configured PostgreSQL database. The current database setup workflow does not create all of these tables from scratch.

## Known limitations

- The three research topics are hardcoded in the ingestion workflow.
- The ingestion workflow requests a maximum of 100 papers per topic and filters out papers without abstracts.
- Extraction depends on the information available in the abstract. Full paper text is not supplied to the extraction model.
- Extracted claims, stances, confidence values, and methodology categories are model-generated and are not independently verified.
- Methodology scores are predefined values, not an independently validated assessment of research quality.
- Contradiction detection identifies changes in stance labels for an author and topic. It does not perform full semantic comparison of paper claims.
- Consensus calculations depend on the extracted stance labels and predefined methodology scores.
- The database setup workflow does not create every table referenced by the other workflows.
- Workflow execution depends on correctly configured n8n credentials, workflow references, database structures, and webhook settings.
- The repository contains workflow exports and deployment documentation. It does not contain a standalone application or local installation script.

## Deployment documentation

See [DEPLOYMENT.md](DEPLOYMENT.md) for deployment status, credential requirements, database configuration, previously encountered issues, and remaining work.

## Future improvements

- Make research topics configurable instead of hardcoding them.
- Add database migrations for all required tables and indexes.
- Improve semantic comparison beyond changes in extracted stance labels.
- Add validation against full-text sources where licensing and access permit.
- Add evidence references linking extracted claims to their source text.
- Add retry and error-routing improvements for external API failures.
- Add reproducible execution logs and test cases for extraction and analysis logic.
