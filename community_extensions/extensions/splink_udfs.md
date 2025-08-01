---
warning: DO NOT CHANGE THIS MANUALLY, THIS IS GENERATED BY https://github/duckdb/community-extensions repository, check README there
title: splink_udfs
excerpt: |
  DuckDB Community Extensions
  Phonetic, text normalization and fuzzy matching functions for record linkage.

extension:
  name: splink_udfs
  description: Phonetic, text normalization and fuzzy matching functions for record linkage.
  version: 0.0.5
  language: C++
  build: cmake
  license: MIT
  maintainers:
    - RobinL

repo:
  github: moj-analytical-services/splink_udfs
  ref: cb048a1cdb5bf85d1c7db032e7bdaba3fc5876ac

docs:
  hello_world: |
    LOAD splink_udfs;
    SELECT soundex(unaccent('Jürgen'));  -- returns 'J625'
  extended_description: |
    The splink_udfs extension provides functions for data cleaning and phonetic matching.

    Includes `soundex(str)`, `strip_diacritics(str)`, `unaccent(str)`,
    `ngrams(list,n)`, `double_metaphone(str)`
    and faster versions of `levenshtein` and `damerau_levenshtein`.

extension_star_count: 5
extension_star_count_pretty: 5
extension_download_count: null
extension_download_count_pretty: n/a
image: '/images/community_extensions/social_preview/preview_community_extension_splink_udfs.png'
layout: community_extension_doc
---

### Installing and Loading
```sql
INSTALL {{ page.extension.name }} FROM community;
LOAD {{ page.extension.name }};
```

{% if page.docs.hello_world %}
### Example
```sql
{{ page.docs.hello_world }}```
{% endif %}

{% if page.docs.extended_description %}
### About {{ page.extension.name }}
{{ page.docs.extended_description }}
{% endif %}

### Added Functions

<div class="extension_functions_table"></div>

|  function_name   | function_type | description | comment | examples |
|------------------|---------------|-------------|---------|----------|
| double_metaphone | scalar        | NULL        | NULL    |          |
| ngrams           | scalar        | NULL        | NULL    |          |
| soundex          | scalar        | NULL        | NULL    |          |
| strip_diacritics | scalar        | NULL        | NULL    |          |
| unaccent         | scalar        | NULL        | NULL    |          |

### Overloaded Functions

<div class="extension_functions_table"></div>

|    function_name    | function_type |                                                                                                                                                     description                                                                                                                                                      | comment |                examples                 |
|---------------------|---------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------|-----------------------------------------|
| damerau_levenshtein | scalar        | Extension of Levenshtein distance to also include transposition of adjacent characters as an allowed edit operation. In other words, the minimum number of edit operations (insertions, deletions, substitutions or transpositions) required to change one string to another. Different case is considered different | NULL    | [damerau_levenshtein('hello', 'world')] |
| levenshtein         | scalar        | The minimum number of single-character edits (insertions, deletions or substitutions) required to change one string to the other. Different case is considered different                                                                                                                                             | NULL    | [levenshtein('duck','db')]              |


