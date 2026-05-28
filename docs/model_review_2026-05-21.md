-- Active: 1776473653729@@kmappprodread.c4xsp663lj2z.us-east-1.rds.amazonaws.com@3306@knowledge_management
# Omni Model Review & Recommendations

**Date:** 2026-05-21
**Reviewer:** GitHub Copilot (Claude Opus 4.7)
**Scope:** Full review of `omni/TPZ_KM_Production/` — `model.yaml`, `relationships.yaml`, all `*.topic.yaml`, `*.query.view.yaml`, and `knowledge_management/*.view.yaml` files. Goal: identify opportunities to improve **query performance** and **AI (Omni AI / agent) context quality**.

> No model files were modified during this review. This document is recommendations only.

---

## 1. Bugs / correctness issues (fix first)

These affect both performance and AI accuracy and should be addressed before broader refactors.

1. **Typo breaks a join.** `relationships.yaml` references `${peson_relationships.relationship_type_id}` (missing the `r`). The join from `person_relationships` to `relationship_types` is currently invalid.
2. **Stray SQL token.** In `omni__clean_attendance_query_view.query.view.yaml`, the `effective_course_days` measure has `sql: COUNT(DISTINCT attendance_date) END` — the trailing `END` is not valid SQL in context.
3. **Truncated SQL with trailing `-- `.** Same file, `count_drops` measure ends in `-- `, suggesting an unfinished edit. It also embeds a correlated subquery that bypasses Omni's join graph.
4. **`numeric_value_weighted` has duplicated branches.** In `omni__all_survey_data_with_demos.query.view.yaml`, both halves of the outer `CASE WHEN ${question_taxonomy.weight} = -1` evaluate the same 0/1/2/3 mapping; only the `4 -` prefix performs reverse scoring. It works today, but ~60 lines of duplication will drift.
5. **`dap_context_category` dimension contains `AVG(...)`.** Aggregates inside a dimension's `sql:` cause Omni to treat it as a measure context; it also duplicates the measure `dap_context_score_label`. Remove the dimension.
6. **Stale duplicate file.** `omni__clean_attendance_query_view Copy.topic.yaml` is a 165-line fork of the production topic. High risk that one silently diverges.
7. **PII leakage in alumni CSV.** `alumni_data_clean_final.view.yaml` exposes `first_name`, `last_name`, `student_phone`, `student_email`, `IP.Address`, `street_address`, primary phone, etc., none marked `hidden: true`. This violates the rule stated in `model.yaml` ("Never expose first_name/last_name in aggregate reporting").
8. **Mis-stated cardinality.** `student_drop_patterns_by_term_data → terms` is declared `one_to_one` on `term_id`, but many students share a term. Should be `many_to_one`. A few other relationships likely have the same issue and should be audited.

---

## 2. Performance recommendations

1. **Materialize the heavy query views.** `omni__all_survey_data_with_demos`, `omni__clean_attendance_query_view`, `student_attendance_rates`, and `question_taxonomy` are large multi-join SQL views executed on every query. In Omni, add a `materialization:` block with a `datagroup_trigger` (e.g., nightly) — these are read-mostly analytic surfaces.
2. **Replace correlated subqueries with PDTs.** `is_true_drop` (dimension) runs an `EXISTS` subquery against `registrations` for every attendance row; `count_drops` (measure) does similar. Precompute a `student_course_drops` view (one row per `person × course`) and join.
3. **Move the Likert score CASE block to a lookup view.** The 30+ row mapping repeated twice in `numeric_value` and `numeric_value_weighted` belongs in a small `survey_answer_score_lookup` view joined on `answer_value_clean`. Faster to evaluate, single source of truth, easier to extend.
4. **Replace string-key joins with id joins.** Multiple relationships join on `term_name`: `omni__clean_attendance_query_view.term_id = terms.id` already exists, but `most_recent_ef_by_student.term_name = terms.name`, `enrollment_forms.term_name = terms.name`, and `student_attendance_rates.term_name = terms.name` use the name. Switch to `term_id` everywhere (or store the id upstream).
5. **Push `question_match_key` upstream.** The nested `REGEXP_REPLACE` / `SUBSTRING` / `LOCATE` chain is evaluated on every row of survey data per query. Persist it as a column in dbt/SQL, or at minimum in the materialized version of the view.
6. **Drop `always_left` where not required.** Forcing `always_left` on every relationship blocks Omni's join-elimination optimizer. Reserve it for joins that are semantically required; for lookup/type tables, leave the default behavior.
7. **Tiered cache policies.** Today there is only `cache_for_a_day`. Add `cache_for_an_hour` (operational dashboards) and `cache_for_a_week` (historical/research topics on materialized views), and apply per topic.
8. **Trim base-view column lists.** `omni__all_survey_data_with_demos`'s base `SELECT` pulls `first_name`, `last_name`, `current_grade`, school columns, races, etc., that are also available via joins. Pulling them in the base scans those tables on every query even when the field isn't needed. Move them to topic-level joins.
9. **`order_by_field`.** Ensure `term_name` and similar dimensions use `terms.start_date` (some already do; `most_recent_ef_by_student.term_name` and a few others don't) so Omni can push sort into the warehouse.

---

## 3. AI-context recommendations

1. **View-level `ai_context` is missing on most views.** `people.view.yaml`, `student_profiles.view.yaml`, `terms.view.yaml`, `courses.view.yaml`, `registrations.view.yaml`, and almost all `*_types.view.yaml` files have no view-level description. Add a 1–2 sentence purpose statement to each — these are the views the AI joins to most often.
    - DONE
2. **Use `ai_fields` on every topic.** Only `omni__clean_attendance_query_view.topic.yaml` uses it. `ai_fields` is the strongest lever for narrowing AI exploration — apply it to all topics, especially `omni__all_survey_data_with_demos` and `student_attendance_rates`.
    - DONE
3. **Hide PII consistently.** Across the model, set `hidden: true` on `first_name`, `last_name`, `email`, `phone`, IP, address, and DOB unless explicitly required. `people.full_name` currently violates the `model.yaml` rule.
    - DONE
4. **Add `format: ID` to every `_id` column.** Several `_id` fields in `student_profiles`, `enrollment_forms`, etc., lack `format: ID`, causing the AI to treat them as numeric measures.
    - DONE
5. **Add `synonyms`** for business vocabulary: "drop"/"withdrawal", "EShip"/"entrepreneurship", "Pathways", "WBL", "DAP", "ES", "EC", "term"/"quarter"/"semester", "stipend", "alumni".
6. **Add `sample_values`** to enum-like dimensions (`enrollment_form_most_recent_status`, `current_grade`, `school_tier`, `course_name_groups`, `gender`, `ethnicity`, `category`). This is done well on `omni__all_survey_data_with_demos.survey_name` — propagate the pattern.
7. **Encode model rules as `default_filters` on topics.** `model.yaml` says "filter out `active=FALSE` or `deleted=TRUE`" and "exclude when `school_release_consent` is FALSE". Put these in topic-level `default_filters` so the AI doesn't have to remember.
8. **Exclude noise from the AI surface.** Add to `ignored_views`: `fake_*`, `temp_*`, `*_translations` (unless needed), `ar_internal_metadata`, `schema_migrations`, `queued_emails`, `ahoy_*` (if not used for AI questions). CSV uploads like `alumni_data_clean_final` should also be hidden unless intentionally queryable.
9. **Mark AGGREGATE_EXPRESSION measures clearly.** You do this on `dap_context_score`, `dap_asset_score`, `dap_context_score_label`. Apply the same `ai_context` note to every measure that cannot appear in `GROUP BY` / cannot be used as a dimension. The AI mis-uses these frequently.
10. **Consolidate duplicate topics.** Delete the "Copy" file; keep either `omni__clean_attendance_query_view.topic.yaml` or `student_attendance_rates.topic.yaml` as the primary attendance topic and have the other reference subset fields via `ai_fields`.
11. **Glossary in `model.yaml`.** Extend `ai_context` with concrete definitions: "student" (a `people` row joined to `student_profiles`), "active student in term X" (`registrations.active=TRUE` + `courses.term_id`), "drop" (true-drop logic), "EShip courses" (`course_name_groups='EShip'`). This block is loaded into every AI request.
12. **Per-topic `ai_context` everywhere.** Most topics lack one. `omni__all_survey_data_with_demos` is a strong template (grain + survey types + scoring rules + AGGREGATE_EXPRESSION warnings). Add at minimum a grain + intended-use sentence to every topic.
    - DONE
13. **Don't expose both `dap_context_category` (dimension) and `dap_context_score_label` (measure)** to the AI — duplicates confuse it.
14. **Replace generic field labels.** `people.calculation` (already hidden) and similar auto-generated names should be removed or aliased. Anything not hidden becomes part of the AI surface.

---

## 4. Recommended priority order

1. Bugs **1–8** above.
2. `ignored_views` expansion + PII hiding (~1 hr; big AI-quality and compliance win).
3. Add `ai_fields` + topic-level `ai_context` + `default_filters` to all topics.
4. Materialize heavy query views.
5. Replace string joins with id joins + restructure `numeric_value` lookup.
6. Add view-level descriptions and synonyms across the `knowledge_management/` directory.

---

## 5. Quick reference: files reviewed

- Top-level: `model.yaml`, `relationships.yaml`, `alumni_data_clean_final.view.yaml`, `omni__all_survey_data_with_demos.topic.yaml`, `omni__clean_attendance_query_view.query.view.yaml`, `omni__clean_attendance_query_view.topic.yaml`, `omni__clean_attendance_query_view Copy.topic.yaml`, `omni__survey_response_rate_query.query.view.yaml`, `student_drop_patterns_by_term_data.query.view.yaml`, and others.
- `knowledge_management/`: `omni__all_survey_data_with_demos.query.view.yaml`, `question_taxonomy.query.view.yaml`, `student_attendance_rates.query.view.yaml`, `enrollment_forms.view.yaml`, `people.view.yaml`, `student_profiles.view.yaml`, plus structural scan of remaining 90+ view files.
