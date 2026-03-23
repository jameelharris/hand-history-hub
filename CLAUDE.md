# CLAUDE.md — Poker Hand History Hub

> **Source of truth:** All schema and architecture decisions derive from `./docs/poker-erd.drawio.svg` (data model) and `./docs/system-arch.svg` (system architecture). When in doubt, defer to those diagrams over any other documentation.

---

## MVP-1 Scope

MVP-1 is strictly limited to **JSON object validation** within BigQuery using dbt.

| Step | Description |
|---|---|
| **Source** | Read JSON objects from `hand_history.raw_ai_hands` in BigQuery |
| **Validation** | dbt logic to assert all required fields are present (per the ERD) |
| **Metadata Lookups** | dbt cross-references lookup tables to validate JSON metadata values |
| **Cleanse** | Logic to identify and delete invalid objects from the raw table |

**Do NOT scaffold BigQuery tables from the ERD.** Table creation is out of scope for MVP-1; the focus is validating raw JSON before any structured tables exist.

**Future GCP services (Cloud Run, Eventarc, Scheduler, Dataform, Gemini, Cloud Storage) are currently out of scope.** See the System Architecture section for context.

---

## Data Model (Source of Truth: `./docs/poker-erd.drawio.svg`)

### Broadcast & Clip Layer

**`Broadcasts`**
- `broadcast_id` PK (auto int)
- `registry_status_id` FK → `Registry_Status_LU`
- `stream_id` (int)
- `stream_title` (char 50)
- `broadcast_start_time` (time)
- `broadcast_end_time` (time)
- `broadcast_date` (date)
- `game_type` (date)

**`Clips`**
- `clip_id` PK (auto int)
- `broadcast_id` FK → `Broadcasts`
- `registry_status_id` FK → `Registry_Status_LU`
- `clip_start_time` (time)
- `clip_end_time` (time)

**`Hands`**
- `hand_sequence_id` PK (auto int)
- `clip_id` FK → `Clips`
- `game_variant_id` FK → `Games_LU`
- `hand_start_time` (time)
- `hand_end_time` (time)

### Lookup / Reference Tables

**`Registry_Status_LU`**
- `registry_status_id` PK (auto int)
- `registry_status_type` (char 50)

**`Games_LU`**
- `game_variant_id` PK (auto int)
- `game_variant_name` (char 50)
- `game_variant_format` (char 50)
- `is_limit` (bool)
- `max_seats` (int)
- `min_seats` (int)

**`Streets_LU`**
- `street_id` PK (auto int)
- `street_name` (char 50)

**`Actions_LU`**
- `action_id` PK (auto int)
- `action_type` (char 50)

**`Sizings_LU`**
- `bet_sizing_id` PK (auto int)
- `bet_sizing_type` (char 50)
- `bet_sizing_percentages` (float array)

**`Limits_LU`**
- `limit_id` PK (auto int)
- `limit_type` (char 50)

**`Modes_LU`**
- `modes_id` PK (auto int)
- `straddle_type` (char 50)
- `max_straddles` (enum)
- `straddle_amount_multiplier` (int)
- `is_straddle_on` (bool)
- `is_bounty_on` (bool)
- `max_players` (int)
- `is_multiple_straddle` (bool)

**`Positions_LU`**
- `position_id` PK (auto int)
- `position_name` (char 50)
- `position_label` (char 50)
- `position_type` (char 50)

**`States_LU`**
- `states_id` PK (auto int)
- `state_change_type` (char 50)

**`Player_Status_LU`**
- `player_status_id` PK (auto int)
- `status_type` (enum)

**`Nodes_LU`**
- `node_id` PK (auto int)
- `node_name` (char 50)

**`Connectedness_LU`**
- `connectedness_id` PK (auto int)
- `connectedness_type` (char 50)
- `connectedness_name` (char 50)

**`Suitedness_LU`**
- `suitedness_id` PK (auto int)
- `suitedness_type` (char 50)
- `suitedness_name` (char 50)

**`Paired_LU`**
- `paired_id` PK (auto int)
- `paired_type` (char 50)
- `paired_name` (char 50)

**`Players_LU`**
- `player_id` PK (auto int)
- `player_name` (char 50)
- `archetype` (char 50)

### Game Configuration Tables

**`Games_Rules`**
- `games_rules_id` PK (auto int)
- `street_id` FK → `Streets_LU`
- `game_variant_id` FK → `Games_LU`
- `num_visible_cards_dealt` (int)
- `num_invisible_cards_dealt` (int)
- `lead_determination_type` (enum)
- `max_num_raises` (enum)
- `max_num_boards` (int)
- `cards_dealt_per_player` (int)

**`Games_Modes`**
- `games_modes_id` PK (auto int)
- `modes_id` FK → `Modes_LU`
- `game_variant_id` FK → `Games_LU`

**`Games_Stakes`**
- `games_rules_id` PK (auto int)
- `game_variant_id` FK → `Games_LU`
- `limit_type_id` FK → `Limits_LU`
- `rake_percentage` (float)
- `bring_in_amount` (int)
- `big_bet` (float)
- `small_blind` (float)
- `big_blind` (float)
- `ante` (float)
- `min_straddle` (float)

**`Games_Positions`**
- `games_position_id` PK (auto int)
- `game_variant_id` FK → `Games_LU`
- `position_id` FK → `Positions_LU`

**`Games_Configurations`**
- `games_configurations_id` PK (auto int)
- `games_modes_id` FK → `Games_Modes`
- `games_rules_id` FK → `Games_Rules`
- `games_stakes_id` FK → `Games_Stakes`
- `games_variant_id` FK → `Games_LU`
- `description` (char 50)

### Street & Action Tables

**`Streets`**
- `street_instance_id` PK (auto int)
- `states_id` FK → `States_LU`
- `initial_aggressor_id` FK
- `cards_instance_id` FK → `Cards`
- `range_profiles_id` FK → `Range_Profiles`
- `is_heads_up` (bool)
- `effective_stack_size` (int)
- `starting_spr` (float)
- `starting_pot_size` (float, default '0.0')
- `street_sequence_index` (int)
- `cumulative_cards_dealt` (varchar 255 array)

**`Streets_Players`**
- `streets_players_id` PK (auto int)
- `cards_instance_id` FK → `Cards`
- `participant_id` (char 50) FK
- `states_id` (char 50) FK → `States_LU`
- `street_instance_id` FK → `Streets`
- `participant_position_id_id` FK → `Positions_LU`
- `player_status_id` FK → `Player_Status_LU`
- `relative_action_order` (int)
- `starting_stack_size` (int)
- `player_status` (char 50)
- `board_valuation_score` (int)
- `straddle_index` (char 50)
- `cumulative_cards_dealt` (varchar 255 array)

**`Players_Actions`**
- `players_action_id` PK (auto int)
- `street_players_id` FK → `Streets_Players`
- `action_id` FK → `Actions_LU`
- `sizing_id` FK → `Sizings_LU`
- `action_sequence_index` (int)
- `is_aggressive` (bool)
- `bet_amount` (float)
- `seconds_to_act` (time)
- `is_all_in` (bool)
- `pot_before_action` (int)

### Cards & Classification Tables

**`Cards`**
- `cards_instance_id` PK (auto int)
- `cards_class_id` FK → `Classes_LU`
- `hand_sequence_id` FK → `Hands`
- `game_configurations_id` FK → `Games_Configurations`
- `cards_dealt` (char 50)
- `is_visible` (bool)
- `owner_type` (char 50)

**`Classes_LU`**
- `class_id` PK (auto int)
- `connectedness_id` FK → `Connectedness_LU`
- `suitedness_id` FK → `Suitedness_LU`
- `paired` (int) FK → `Paired_LU`
- `scope` (char 50)

### Range & Formation Tables

**`Formations_LU`**
- `formation_id` PK (auto int)
- `defender_position_ids` (varchar 255 array) FK → `Positions_LU`
- `aggressor_position_id` FK → `Positions_LU`
- `formation_type` (char 50)
- `is_multiway` (bool)

**`Range_Profiles`**
- `range_profiles_id` PK (auto int)
- `game_variant_id` FK → `Games_LU`
- `node_id` FK → `Nodes_LU` (default NULL)
- `formation_id` FK → `Formations_LU` (default NULL)
- `aggressor_range_shape` (char 50, default NULL)
- `defender_range_shape` (varchar 255 array, default NULL)

---

## System Architecture (Source of Truth: `./docs/system-arch.svg`)

The full pipeline is GCP-native. For MVP-1, only **BigQuery** and **dbt** are in scope.

### Full Pipeline (for context only — future scope)

| Component | GCP Service | Role |
|---|---|---|
| Downloader Trigger | Cloud Scheduler | Schedules download jobs every 30 min |
| Downloader | Cloud Run | Downloads broadcasts; uploads to Storage; writes to Clip Registry |
| Video Clipper Trigger | Eventarc | Fires on new video assets |
| Video Clipper | Cloud Run | Clips broadcasts to hand-level granularity |
| AI Transformation | Dataform + Gemini (BQML) | Decomposes clips into structured hand records |
| Clip Registry | BigQuery | Stores clip metadata |
| Hand History | BigQuery | Final structured hand history store |
| Video Assets | Cloud Storage | Raw broadcast video |
| Clip Assets | Cloud Storage | Hand-level video clips |

**Data flow:** Poker Broadcast Registry → Downloader → Video Assets + Clip Registry → Video Clipper → Clip Registry → AI Transformation → `hand_history` dataset in BigQuery.

### MVP-1 Relevant Components

- **`hand_history.raw_ai_hands`** (BigQuery) — source table containing raw JSON objects produced by the AI transformation step.
- **dbt** — validation, lookup cross-referencing, and cleansing logic runs here against BigQuery.

---

## dbt & BigQuery Commands

### Setup

```bash
# Install dbt with BigQuery adapter
pip install dbt-bigquery

# Authenticate with GCP
gcloud auth application-default login

# Verify dbt connection
dbt debug
```

### Development

```bash
# Run all models
dbt run

# Run a specific model
dbt run --select <model_name>

# Run models tagged for MVP-1
dbt run --select tag:mvp1
```

### Testing (Validation Logic)

```bash
# Run all tests (schema + custom data tests)
dbt test

# Run tests for a specific model
dbt test --select <model_name>

# Run only schema tests
dbt test --select test_type:generic

# Run only custom singular tests
dbt test --select test_type:singular
```

### Inspection & Lineage

```bash
# Compile models without executing (useful for reviewing SQL)
dbt compile

# Generate and serve documentation with lineage graph
dbt docs generate && dbt docs serve
```

### Profiles

dbt connects to BigQuery via `~/.dbt/profiles.yml`. The target project and dataset must reference the GCP project containing `hand_history.raw_ai_hands`.

---

## Key Conventions for MVP-1

- **Raw source:** `hand_history.raw_ai_hands` — treat as append-only; do not mutate rows directly from dbt models. Deletion of invalid records is handled via a separate cleanse step.
- **Validation tests live in dbt** — use `schema.yml` for generic field-presence tests and `tests/` for custom singular SQL tests that cross-reference lookup tables.
- **Lookup validation:** When the JSON contains a metadata field (e.g. `game_variant_id`, `registry_status_id`, `action_type`), the dbt test must confirm that value exists in the corresponding `_LU` table.
- **No DDL in dbt for MVP-1** — models should use `{{ source() }}` references only; do not define or create target tables.
- **Invalid record identification** — output of validation tests should produce a list of `hand_sequence_id` values (or equivalent JSON object identifiers) that fail validation; cleanse logic operates on that list.
