# System 1 Agentic Loop Evidence

- Model: `claude-haiku-4-5-20251001`
- Fixtures processed: 8
- Estimated total cost: **$0.0973** USD

| claim_id | outcome | claim_type | severity | turns | clarifications_asked | input_tokens | output_tokens | est_cost_usd | elapsed_s |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| claim_01_kitchen_fire | escalated | property_damage | medium | 3 | 0 | 9491 | 539 | 0.0122 |  |
| claim_02_stolen_bike | escalated | theft | medium | 2 | 0 | 6158 | 430 | 0.0083 |  |
| claim_03_water_damage | escalated | property_damage | medium | 2 | 0 | 6109 | 395 | 0.0081 |  |
| claim_04_neighbor_injury | escalated | liability | medium | 2 | 0 | 6180 | 460 | 0.0085 |  |
| claim_05_auto_collision | routed | auto | high | 4 | 0 | 14962 | 960 | 0.0198 |  |
| claim_06_low_confidence_escalation | escalated | unknown | unknown | 3 | 0 | 9433 | 583 | 0.0123 |  |
| claim_07_tree_falls_on_car | escalated | auto | medium | 3 | 0 | 10131 | 675 | 0.0135 |  |
| claim_08_minor_porch_damage | escalated | property_damage | low | 3 | 0 | 10474 | 835 | 0.0146 |  |

## Verification

- All 8 fixtures reached a terminal outcome: `routed` or `escalated`.
- The loop continues on `stop_reason == "tool_use"` and returns on `stop_reason == "end_turn"`.
- No integer-literal iteration cap is used as the primary loop-control mechanism.
- Terminal routing/escalation state is recorded in the per-claim session.