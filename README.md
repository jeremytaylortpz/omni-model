# omni-model

Omni semantic model for **The Possible Zone (TPZ)** analytics environment.

## What is in this repository

This repository contains a single Omni model under `/home/runner/work/omni-model/omni-model/omni/TPZ_KM_Production` with:

- `model.yaml` — top-level model configuration, cache policy, ignored views, and global AI context
- `relationships.yaml` — cross-view join definitions
- `*.view.yaml` files — base views and query-backed views
- `*.topic.yaml` files — analyst-facing topics built on top of the views
- `knowledge_management/` — the core semantic layer for the KM database

## Model overview

The model is centered on TPZ student, enrollment, attendance, and survey analysis.

Important global rules already encoded in `model.yaml`:

- Use `people.person_id` / `student_profiles.person_id` as the student identifier for joins and counts
- Do **not** expose student first/last names in aggregate reporting
- Filter out `active = FALSE` or `deleted = TRUE` records
- Exclude students with `student_profiles.school_release_consent = FALSE` from external reporting
- TPZ fiscal year runs **July through June**

## Key high-value model areas

- `knowledge_management/people.view.yaml` — person-level identifiers and student counting helpers
- `knowledge_management/omni__all_survey_data_with_demos.query.view.yaml` — survey response fact table with demographics pre-joined
- `knowledge_management/question_taxonomy.query.view.yaml` — normalized question taxonomy used to group survey items
- `student_attendance_rates.topic.yaml` / attendance query views — attendance and persistence analysis
- `Alumni Survey Data.topic.yaml` — alumni survey analysis using `survey_responses.sm_id` as the respondent key

## Working with the model

There is no application build/test harness in this repository. A simple validation pass is to parse the YAML files:

```bash
cd /home/runner/work/omni-model/omni-model
ruby -e 'require "yaml"; require "date"; files = Dir["**/*.{yaml,yml}"].sort; files.each { |f| YAML.load_file(f, permitted_classes: [Date, Time, Symbol], aliases: true) }; puts "Parsed #{files.length} YAML files successfully"'
```

## Performance review notes

These are recommended follow-up improvements based on the current model files:

1. **Tune cache policies for expensive query views**  
   `model.yaml` currently defines a single day-long cache policy for the whole model. Heavier query views such as `knowledge_management/omni__all_survey_data_with_demos.query.view.yaml` and `omni__survey_response_rate_query.query.view.yaml` are good candidates for explicit per-view cache policies if freshness requirements allow it.

2. **Reduce repeated question-text normalization logic**  
   `knowledge_management/question_taxonomy.query.view.yaml` repeats the same expensive `REPLACE` / `REGEXP_REPLACE` / `SUBSTRING` normalization logic multiple times. Moving that normalization into one reusable helper layer would reduce repeated computation and make the logic easier to maintain.

3. **Review large pre-joined survey views for selective use**  
   `knowledge_management/omni__all_survey_data_with_demos.query.view.yaml` is intentionally rich and convenient, but it joins many demographic and survey tables at once. For performance-sensitive dashboards, consider narrower topic/view entry points when all columns are not required.

4. **Verify assumed relationship cardinality**  
   `relationships.yaml` contains several `assumed_many_to_one` joins. Confirming the actual cardinality of those joins can improve model reliability and reduce the risk of accidental fanout in downstream analysis.

## AI context review notes

Recommended follow-up improvements for analyst and AI usability:

1. **Standardize topic-level `ai_context` coverage**  
   Some topics are richly documented, while others are very sparse. Every topic should ideally describe:
   - the grain
   - the preferred identifier
   - common filters
   - any scoring or interpretation rules

2. **Document grain consistently on complex query views**  
   The strongest example is `omni__all_survey_data_with_demos`, which already describes its row grain well. Applying that same pattern across other topics and query views will improve both analyst self-service and AI answer quality.

3. **Call out canonical fields more aggressively**  
   The model already contains strong guidance like preferring canonical question text and distinct student-count measures. Expanding those notes to more dimensions/measures will help AI choose safer fields by default.

4. **Keep duplicate or backup artifacts out of the model tree**  
   Files like `omni__clean_attendance_query_view Copy.topic.yaml` add noise to the semantic layer and to AI context retrieval. Removing obsolete copies in a future cleanup would make the model easier to navigate.
