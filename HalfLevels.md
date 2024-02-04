Half the database items that involve levels:

```
# item level requirements, replace the level part of the requirement list, level requirement is the condition 5
UPDATE ch_unitydatadb.item_templates
SET requirements = REGEXP_REPLACE(requirements, '(5\^)(\d+)', E'\\1' || (CAST('\\2' AS INTEGER) / 2)::TEXT)
WHERE requirements ~ '5\^\d+';

# mob levels
UPDATE ch_unitydatadb.mob_templates SET level = level / 2;

# quest levels
UPDATE ch_unitydatadb.quest_templates SET level_required = level_required / 2;

# not going to change cooking

# signpost LevelGreaterThan(=7), FirstTimeComplete(=40) is coded to fail on player level > 19 for some reason so that's a code change
UPDATE ch_unitydatadb.signpost_conditions SET params = params / 2 WHERE condition_type_id = 7;
```
