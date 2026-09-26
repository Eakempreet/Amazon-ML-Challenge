# Dataset inventory and model rationale

Prepared from the supplied local files on 25 September 2026. Full-corpus counts below were recomputed; pilot-only findings are explicitly marked. The machine-readable `artifacts/dataset_audit/dataset_audit.json` contains every count, length histogram, percentile, fingerprint statistic, ID range, and SHA-256 collected by the audit. No external business lookup was used.

## Why multilingual DistilBERT is proposed

The proposed checkpoint is [`distilbert/distilbert-base-multilingual-cased`](https://huggingface.co/distilbert/distilbert-base-multilingual-cased). Its model card describes a 134M-parameter, six-layer, 768-dimensional model with 12 attention heads, pretrained on Wikipedia in 104 languages, under Apache 2.0. Its [configuration](https://huggingface.co/distilbert/distilbert-base-multilingual-cased/raw/main/config.json) allows 512 positions; 128 tokens is our initial runtime setting, not its architectural limit. Token-length/truncation rates on this dataset have not been measured yet.

The intended role is a supervised cross-encoder: put both business names and addresses in one paired input, train a binary classification head on supplied match/nonmatch labels, and score difficult candidates. Multilingual pretraining is a reason to test it against cross-script names and French text; it does not guarantee transliteration matching, French entity-resolution accuracy, or superiority to the tree baseline. Semantic similarity alone does not establish business identity. Keeping number/lexical evidence and validating precision remain necessary.

DistilBERT is still a proposal. Only LightGBM has actually been trained and evaluated here. The transformer is not a chat model, an external business lookup, an already-fine-tuned entity matcher, or the current blocking engine. The compact size is a practical first experiment under the 60-hour deadline; a larger cloud GPU does not by itself justify a more complex ensemble.

## Files, schema, and meaning

| File | Rows excluding header | Exact bytes | MiB |
| --- | --- | --- | --- |
| train_source1.tsv | 2,206,821 | 210,069,713 | 200.338 |
| train_source2.tsv | 5,034,616 | 489,301,488 | 466.634 |
| train_source3.tsv | 5,285,603 | 503,705,637 | 480.371 |
| test_source1.tsv | 1,732,544 | 175,022,086 | 166.914 |
| test_source2.tsv | 4,887,273 | 509,456,422 | 485.856 |
| test_source3.tsv | 5,082,316 | 506,002,772 | 482.562 |
| train_ground_truth.tsv | 2,206,821 | 127,015,583 | 121.131 |

Total record rows across six source files: **24,229,173**. All seven TSVs together: **2,520,573,701 bytes** (2.5206 decimal GB; 2.3475 GiB). Label rows describe anchors and are not additional business records.

| Column | Meaning | How to use it |
| --- | --- | --- |
| entity_id | Unique record key with S1-/S2-/S3- prefix | Join/output key; numeric suffix is not a prediction feature |
| business_name | Raw business name | Preserve raw text and create additional normalized views |
| business_address | One free-text address field | May be blank, abbreviated, reordered, partially missing, or noisy |
| country | Open string label: US, India, France observed | Useful partition/context; do not limit test processing to training labels |

All six source files have exactly those four columns and parse as UTF-8 TSV. There is no explicit source column, structured city/state/postcode column, phone, email, latitude/longitude, registration number, or stable shared cross-source business identifier. URL-like text sometimes occurs inside existing fields, but there is no structured website column. The raw TSVs are authoritative; `__MACOSX`, `._*`, and `.DS_Store` are archive/system metadata, not extra datasets.

Source 1 is the reference list, described as deduplicated at the business level. Sources 2 and 3 are noisy fragments. A Source 1 entity can have zero, one, or multiple matches in either source. Different Source 1 entities can share a business-name string or an address string. The task is to return sets of S2/S3 IDs for each S1 ID, not to cluster every record independently or impose one result per source.

## Countries and train/test shift

| File | US | India | France |
| --- | --- | --- | --- |
| train_source1 | 1,323,633 | 883,188 | 0 |
| train_source2 | 3,016,817 | 2,017,799 | 0 |
| train_source3 | 3,170,056 | 2,115,547 | 0 |
| test_source1 | 663,106 | 809,986 | 259,452 |
| test_source2 | 1,871,330 | 2,312,565 | 703,378 |
| test_source3 | 1,945,701 | 2,405,000 | 731,615 |

| Reference split | Country | Records | Share |
| --- | --- | --- | --- |
| train | US | 1,323,633 | 59.979% |
| train | India | 883,188 | 40.021% |
| test | US | 663,106 | 38.274% |
| test | France | 259,452 | 14.975% |
| test | India | 809,986 | 46.751% |

France has no supplied labelled examples. Its singleton rate, true match counts, matching difficulty, and score cannot be measured from test text alone. Test country proportions also differ from training. Country labels are not language labels: Indian records can mix Latin transliterations and Indic scripts, and non-ASCII characters can be accents or punctuation rather than a different language.

## Missing fields and Unicode

| File | Blank names | Blank addresses | Blank address rate | Non-ASCII names | Non-ASCII addresses |
| --- | --- | --- | --- | --- | --- |
| train_source1 | 0 | 0 | 0.000% | 0 | 554 |
| train_source2 | 0 | 168,967 | 3.356% | 764,608 | 478,453 |
| train_source3 | 0 | 175,916 | 3.328% | 606,737 | 476,588 |
| test_source1 | 0 | 0 | 0.000% | 40,789 | 73,800 |
| test_source2 | 0 | 129,408 | 2.648% | 928,158 | 720,665 |
| test_source3 | 0 | 136,098 | 2.678% | 737,515 | 729,222 |

The following table breaks missing addresses and non-ASCII text down by country. Empty means an empty or whitespace-only string; literal placeholder strings are separately counted in the small-formatting-details tables below.

| File | Country | Rows | Blank addresses | Non-ASCII names | Non-ASCII addresses |
| --- | --- | --- | --- | --- | --- |
| train_source1 | US | 1,323,633 | 0 | 0 | 0 |
| train_source1 | India | 883,188 | 0 | 0 | 554 |
| train_source2 | India | 2,017,799 | 57,846 | 562,437 | 478,396 |
| train_source2 | US | 3,016,817 | 111,121 | 202,171 | 57 |
| train_source3 | US | 3,170,056 | 110,968 | 215,565 | 52 |
| train_source3 | India | 2,115,547 | 64,948 | 391,172 | 476,536 |
| test_source1 | US | 663,106 | 0 | 0 | 0 |
| test_source1 | France | 259,452 | 0 | 40,789 | 73,335 |
| test_source1 | India | 809,986 | 0 | 0 | 465 |
| test_source2 | India | 2,312,565 | 52,764 | 638,614 | 551,461 |
| test_source2 | France | 703,378 | 21,537 | 172,617 | 169,163 |
| test_source2 | US | 1,871,330 | 55,107 | 116,927 | 41 |
| test_source3 | India | 2,405,000 | 59,240 | 437,903 | 551,274 |
| test_source3 | France | 731,615 | 21,541 | 175,185 | 177,914 |
| test_source3 | US | 1,945,701 | 55,317 | 124,427 | 34 |

## Scripts actually detected

Counts mean records containing at least one character in the tested script blocks, not language identification. Script categories can overlap within a record. The audit checks Devanagari, Bengali, Gurmukhi, Gujarati, Odia, Tamil, Telugu, Kannada, Malayalam, Arabic, Cyrillic, Greek and CJK; untested scripts and Latin accents are not silently classified as one of these.

| File | Country | Field | Detected script: row count |
| --- | --- | --- | --- |
| train_source1 | US | business_name | None of the tested non-Latin blocks |
| train_source1 | US | business_address | None of the tested non-Latin blocks |
| train_source1 | India | business_name | None of the tested non-Latin blocks |
| train_source1 | India | business_address | None of the tested non-Latin blocks |
| train_source2 | India | business_name | Devanagari: 269,424; Tamil: 33,781; Gujarati: 30,929; Kannada: 37,211; Bengali: 30,723; Telugu: 39,323; Malayalam: 18,773; Odia: 7,493; Gurmukhi: 6,688 |
| train_source2 | India | business_address | Devanagari: 277,523; Bengali: 31,770; Gujarati: 31,543; Tamil: 34,777; Kannada: 38,560; Telugu: 34,382; Malayalam: 15,680; Odia: 6,157; Gurmukhi: 7,129 |
| train_source2 | US | business_name | None of the tested non-Latin blocks |
| train_source2 | US | business_address | None of the tested non-Latin blocks |
| train_source3 | US | business_name | None of the tested non-Latin blocks |
| train_source3 | US | business_address | None of the tested non-Latin blocks |
| train_source3 | India | business_name | Tamil: 19,790; Malayalam: 11,116; Devanagari: 158,003; Telugu: 23,033; Gujarati: 18,020; Kannada: 21,995; Odia: 4,317; Bengali: 18,144; Gurmukhi: 4,106 |
| train_source3 | India | business_address | Kannada: 38,974; Devanagari: 275,861; Gujarati: 31,517; Telugu: 34,324; Tamil: 34,978; Bengali: 31,448; Malayalam: 15,379; Gurmukhi: 7,195; Odia: 6,102 |
| test_source1 | US | business_name | None of the tested non-Latin blocks |
| test_source1 | US | business_address | None of the tested non-Latin blocks |
| test_source1 | France | business_name | None of the tested non-Latin blocks |
| test_source1 | France | business_address | None of the tested non-Latin blocks |
| test_source1 | India | business_name | None of the tested non-Latin blocks |
| test_source1 | India | business_address | None of the tested non-Latin blocks |
| test_source2 | India | business_name | Telugu: 45,171; Gujarati: 35,321; Gurmukhi: 7,903; Bengali: 35,546; Malayalam: 22,033; Devanagari: 309,103; Kannada: 43,906; Tamil: 38,953; Odia: 8,670 |
| test_source2 | India | business_address | Gujarati: 36,709; Devanagari: 318,369; Gurmukhi: 8,195; Odia: 7,088; Tamil: 40,621; Malayalam: 18,129; Bengali: 36,832; Kannada: 45,006; Telugu: 39,624 |
| test_source2 | France | business_name | None of the tested non-Latin blocks |
| test_source2 | France | business_address | None of the tested non-Latin blocks |
| test_source2 | US | business_name | None of the tested non-Latin blocks |
| test_source2 | US | business_address | None of the tested non-Latin blocks |
| test_source3 | India | business_name | Devanagari: 181,068; Tamil: 22,808; Kannada: 25,832; Gujarati: 20,570; Gurmukhi: 4,634; Bengali: 20,872; Malayalam: 13,095; Telugu: 26,698; Odia: 5,062 |
| test_source3 | India | business_address | Devanagari: 319,073; Kannada: 45,164; Bengali: 36,427; Tamil: 40,544; Gujarati: 36,732; Gurmukhi: 7,992; Telugu: 39,430; Malayalam: 17,933; Odia: 7,188 |
| test_source3 | France | business_name | None of the tested non-Latin blocks |
| test_source3 | France | business_address | None of the tested non-Latin blocks |
| test_source3 | US | business_name | None of the tested non-Latin blocks |
| test_source3 | US | business_address | None of the tested non-Latin blocks |

## Text lengths

Lengths below are Unicode characters, including spaces and punctuation. Word counts in the JSON use whitespace splitting. Neither is a transformer-token count. Zeros from missing fields are included in distributions.

| File/country | Field | Min | Median | P95 | P99 | Max | Mean |
| --- | --- | --- | --- | --- | --- | --- | --- |
| train_source1/US | business_name | 3 | 22 | 35 | 41 | 68 | 22.47 |
| train_source1/US | business_address | 11 | 34 | 48 | 55 | 93 | 34.96 |
| train_source1/India | business_name | 5 | 27 | 38 | 43 | 105 | 26.39 |
| train_source1/India | business_address | 14 | 76 | 116 | 134 | 256 | 77.70 |
| train_source2/India | business_name | 2 | 27 | 42 | 49 | 104 | 27.42 |
| train_source2/India | business_address | 0 | 68 | 110 | 128 | 249 | 68.20 |
| train_source2/US | business_name | 2 | 23 | 39 | 46 | 87 | 23.55 |
| train_source2/US | business_address | 0 | 32 | 43 | 49 | 73 | 31.53 |
| train_source3/US | business_name | 2 | 23 | 40 | 49 | 88 | 23.98 |
| train_source3/US | business_address | 0 | 39 | 53 | 60 | 104 | 38.45 |
| train_source3/India | business_name | 2 | 27 | 43 | 51 | 123 | 27.04 |
| train_source3/India | business_address | 0 | 58 | 106 | 126 | 240 | 59.11 |
| test_source1/US | business_name | 3 | 22 | 35 | 41 | 66 | 22.46 |
| test_source1/US | business_address | 13 | 34 | 48 | 55 | 92 | 34.97 |
| test_source1/France | business_name | 5 | 19 | 29 | 33 | 50 | 19.42 |
| test_source1/France | business_address | 22 | 48 | 65 | 78 | 147 | 50.07 |
| test_source1/India | business_name | 5 | 27 | 38 | 43 | 92 | 26.37 |
| test_source1/India | business_address | 11 | 76 | 116 | 134 | 268 | 77.71 |
| test_source2/India | business_name | 2 | 28 | 44 | 51 | 102 | 28.25 |
| test_source2/India | business_address | 0 | 68 | 110 | 128 | 269 | 68.75 |
| test_source2/France | business_name | 2 | 20 | 35 | 41 | 60 | 21.10 |
| test_source2/France | business_address | 0 | 40 | 59 | 69 | 136 | 39.55 |
| test_source2/US | business_name | 2 | 24 | 39 | 47 | 85 | 24.29 |
| test_source2/US | business_address | 0 | 32 | 43 | 49 | 75 | 31.84 |
| test_source3/India | business_name | 2 | 28 | 44 | 52 | 103 | 27.87 |
| test_source3/India | business_address | 0 | 59 | 106 | 126 | 267 | 59.42 |
| test_source3/France | business_name | 2 | 20 | 36 | 43 | 69 | 21.14 |
| test_source3/France | business_address | 0 | 41 | 60 | 70 | 139 | 39.97 |
| test_source3/US | business_name | 2 | 24 | 41 | 49 | 85 | 24.62 |
| test_source3/US | business_address | 0 | 39 | 53 | 60 | 107 | 38.84 |

## Ground truth and match multiplicity

Labels have exactly `source1_entity_id` and `matched_entity_ids`. A row contains the complete comma-separated match list for one anchor; an empty list denotes a singleton. The source file and label file must be joined by ID, not row position. There is no separate supplied negative-pair file or validation split.

| Scope | Anchors | Positive links | Singletons | Singleton rate | Mean links/anchor | Mean links/non-singleton |
| --- | --- | --- | --- | --- | --- | --- |
| All | 2,206,821 | 7,638,365 | 123,247 | 5.5848% | 3.4613 | 3.6660 |
| US | 1,323,633 | 4,578,522 | 73,896 | 5.5828% | 3.4591 | 3.6636 |
| India | 883,188 | 3,059,843 | 49,351 | 5.5878% | 3.4645 | 3.6696 |

| True matches for an anchor | Anchors | Share |
| --- | --- | --- |
| 0 | 123,247 | 5.585% |
| 1 | 119,157 | 5.399% |
| 2 | 375,212 | 17.002% |
| 3 | 530,841 | 24.055% |
| 4 | 484,115 | 21.937% |
| 5 | 321,957 | 14.589% |
| 6 | 164,868 | 7.471% |
| 7 | 63,968 | 2.899% |
| 8 | 18,680 | 0.846% |
| 9 | 4,205 | 0.191% |
| 10 | 534 | 0.024% |
| 11 | 37 | 0.002% |

| Scope | Neither source | Only S2 | Only S3 | Both S2 and S3 |
| --- | --- | --- | --- | --- |
| All | 123,247 | 143,029 | 164,498 | 1,776,047 |
| US | 73,896 | 85,827 | 98,848 | 1,065,062 |
| India | 49,351 | 57,202 | 65,650 | 710,985 |

Per-anchor source2 match-count distribution:

| Matches in this source | Anchors |
| --- | --- |
| 0 | 287,745 |
| 1 | 789,108 |
| 2 | 652,779 |
| 3 | 333,957 |
| 4 | 119,078 |
| 5 | 24,154 |

Per-anchor source3 match-count distribution:

| Matches in this source | Anchors |
| --- | --- |
| 0 | 266,276 |
| 1 | 716,417 |
| 2 | 668,375 |
| 3 | 372,443 |
| 4 | 145,116 |
| 5 | 35,378 |
| 6 | 2,816 |

The JSON additionally contains the full joint S2-count/S3-count table, separately for each training country. The observed maximum is a training-data fact, not a rule permitting us to truncate test predictions to that size.

## Label integrity and fragments without a reference link

| Check | Count |
| --- | --- |
| Malformed label rows | 0 |
| Duplicate Source 1 label rows | 0 |
| Reference IDs missing a label row | 0 |
| Label rows repeating a target ID | 0 |

| Target source | Labelled links | Unique referenced targets | Unreferenced targets | Missing target IDs | Targets referenced more than once | Cross-country true links |
| --- | --- | --- | --- | --- | --- | --- |
| S2 | 3,693,619 | 3,693,619 | 1,340,997 | 0 | 0 | 0 |
| S3 | 3,944,746 | 3,944,746 | 1,340,857 | 0 | 0 | 0 |

| Source | Country | Records | Referenced | Unreferenced |
| --- | --- | --- | --- | --- |
| S2 | US | 3,016,817 | 2,213,074 | 803,743 |
| S2 | India | 2,017,799 | 1,480,545 | 537,254 |
| S3 | US | 3,170,056 | 2,365,448 | 804,608 |
| S3 | India | 2,115,547 | 1,579,298 | 536,249 |

| Source | Referenced targets with blank address | Referenced non-ASCII names | Referenced tested non-Latin names | Unreferenced targets with blank address |
| --- | --- | --- | --- | --- |
| S2 | 165,184 | 583,520 | 344,448 | 3,783 |
| S3 | 171,834 | 477,261 | 206,792 | 4,082 |

Unreferenced means absent from all provided Source 1 match lists; it does not mean the record is invalid or describes no real business. Such records are important distractors. Any reverse uniqueness observed in training is a data property, not an explicit instruction to force a global one-to-one assignment on test.

## IDs, repeated text, and train/test overlap

| File | Duplicate ID rows | Malformed data rows | Numeric suffix min | Numeric suffix max | ID length: count |
| --- | --- | --- | --- | --- | --- |
| train_source1 | 0 | 0 | 217 | 999998822 | 12: 1,986,420; 11: 198,190; 10: 19,912; 9: 2,072; 8: 203; 6: 4; 7: 20 |
| train_source2 | 0 | 0 | 576 | 999999727 | 12: 4,530,567; 11: 453,700; 10: 45,433; 9: 4,404; 8: 469; 7: 40; 6: 3 |
| train_source3 | 0 | 0 | 10 | 999999866 | 12: 4,755,781; 11: 477,291; 10: 47,337; 9: 4,682; 8: 462; 7: 47; 6: 2; 5: 1 |
| test_source1 | 0 | 0 | 401 | 999998909 | 12: 1,559,869; 11: 155,513; 10: 15,466; 9: 1,524; 8: 154; 7: 15; 6: 3 |
| test_source2 | 0 | 0 | 208 | 999999982 | 12: 4,398,891; 11: 439,602; 10: 43,823; 9: 4,500; 8: 426; 7: 28; 6: 3 |
| test_source3 | 0 | 0 | 5 | 999999995 | 12: 4,573,479; 11: 457,859; 10: 45,776; 9: 4,691; 8: 453; 7: 49; 6: 8; 4: 1 |

Every ID used by this completed audit passed its expected source prefix and canonical ASCII-decimal suffix check, including no leading-zero ambiguity. The numeric ranges above are diagnostics only. They do not supply identity features or justify ID-based shortcuts.

| File | Raw value compared | Distinct values | Repeated groups | Rows in repeated groups | Extra rows after first | Largest group |
| --- | --- | --- | --- | --- | --- | --- |
| train_source1 | raw_name_address_country | 2,206,821 | 0 | 0 | 0 | 1 |
| train_source1 | raw_business_name | 1,539,229 | 177,793 | 845,385 | 667,592 | 253 |
| train_source1 | raw_business_address | 2,130,606 | 40,089 | 116,304 | 76,215 | 14 |
| train_source2 | raw_name_address_country | 5,008,743 | 25,060 | 50,933 | 25,873 | 5 |
| train_source2 | raw_business_name | 4,402,009 | 239,779 | 872,386 | 632,607 | 320 |
| train_source2 | raw_business_address | 4,337,262 | 421,473 | 1,118,827 | 697,354 | 168,967 |
| train_source3 | raw_name_address_country | 5,266,743 | 18,381 | 37,241 | 18,860 | 4 |
| train_source3 | raw_business_name | 4,651,609 | 258,276 | 892,270 | 633,994 | 421 |
| train_source3 | raw_business_address | 4,632,765 | 382,891 | 1,035,729 | 652,838 | 175,916 |
| test_source1 | raw_name_address_country | 1,732,544 | 0 | 0 | 0 | 1 |
| test_source1 | raw_business_name | 1,238,867 | 129,955 | 623,632 | 493,677 | 205 |
| test_source1 | raw_business_address | 1,677,483 | 31,396 | 86,457 | 55,061 | 99 |
| test_source2 | raw_name_address_country | 4,864,632 | 21,909 | 44,550 | 22,641 | 5 |
| test_source2 | raw_business_name | 4,311,041 | 223,103 | 799,335 | 576,232 | 302 |
| test_source2 | raw_business_address | 4,224,784 | 444,699 | 1,107,188 | 662,489 | 129,408 |
| test_source3 | raw_name_address_country | 5,066,023 | 15,861 | 32,154 | 16,293 | 4 |
| test_source3 | raw_business_name | 4,521,929 | 238,156 | 798,543 | 560,387 | 387 |
| test_source3 | raw_business_address | 4,456,436 | 404,757 | 1,030,637 | 625,880 | 136,098 |

Text repetition uses BLAKE2b-128 fingerprints, with no independent collision verification. It compares raw field content, not normalized strings or real-world identities. Blank addresses are included and can dominate the largest repeated-address group. Entity-ID uniqueness checks are exact and do not use those fingerprints.

| Source | Shared train/test record IDs | Shared unique raw name+address+country fingerprints |
| --- | --- | --- |
| S1 | 0 | 0 |
| S2 | 0 | 0 |
| S3 | 0 | 0 |

No shared raw signatures, if observed, rules out only that narrow exact-overlap case. It does not prove that train and test lack related companies, near-duplicates, shared brands, or address neighborhoods.

## Small formatting details

These are per-record presence counts. Uppercase/lowercase tests refer only to cased letters; they are not language labels. URL-like matches are heuristic and may include false positives. No detected URL was opened.

**train_source1**

| Property | US/business_name | US/business_address | India/business_name | India/business_address |
| --- | --- | --- | --- | --- |
| literal_placeholder | 0 | 0 | 0 | 0 |
| leading_or_trailing_whitespace | 0 | 0 | 0 | 0 |
| repeated_whitespace | 0 | 968 | 0 | 0 |
| has_digit | 34,494 | 1,323,632 | 1,091 | 806,152 |
| has_ampersand | 74,036 | 146 | 37,861 | 19,268 |
| has_comma | 168,736 | 1,323,633 | 0 | 883,188 |
| has_hyphen | 11,326 | 14,864 | 2,191 | 340,798 |
| has_slash | 5,127 | 2,484 | 49 | 300,821 |
| has_hash | 1,399 | 4,670 | 3 | 11,971 |
| has_parenthesis | 41 | 818 | 45,447 | 46,370 |
| has_apostrophe | 46,592 | 1,132 | 14 | 4,233 |
| uppercase_only_cased_letters | 0 | 4 | 0 | 0 |
| lowercase_only_cased_letters | 0 | 0 | 0 | 0 |
| url_like | 0 | 0 | 0 | 33 |
| embedded_tab_or_newline | 0 | 0 | 0 | 0 |

**train_source2**

| Property | India/business_name | India/business_address | US/business_name | US/business_address |
| --- | --- | --- | --- | --- |
| literal_placeholder | 0 | 0 | 6 | 0 |
| leading_or_trailing_whitespace | 0 | 0 | 0 | 0 |
| repeated_whitespace | 202,528 | 24,620 | 351,864 | 82,954 |
| has_digit | 62,595 | 1,842,584 | 193,041 | 2,721,091 |
| has_ampersand | 68,006 | 40,427 | 141,070 | 235 |
| has_comma | 0 | 1,959,953 | 345,864 | 2,905,696 |
| has_hyphen | 88,263 | 844,699 | 177,390 | 222,140 |
| has_slash | 25,759 | 715,124 | 10,912 | 91,854 |
| has_hash | 10,625 | 224,277 | 23,523 | 157,460 |
| has_parenthesis | 146,601 | 92,166 | 104,583 | 1,169 |
| has_apostrophe | 34 | 8,577 | 96,115 | 2,307 |
| uppercase_only_cased_letters | 301,505 | 481,567 | 650,081 | 2,709,537 |
| lowercase_only_cased_letters | 88,058 | 0 | 201,567 | 0 |
| url_like | 68,573 | 63 | 132,693 | 0 |
| embedded_tab_or_newline | 0 | 0 | 0 | 0 |

**train_source3**

| Property | US/business_name | US/business_address | India/business_name | India/business_address |
| --- | --- | --- | --- | --- |
| literal_placeholder | 11 | 0 | 7 | 0 |
| leading_or_trailing_whitespace | 0 | 0 | 0 | 0 |
| repeated_whitespace | 352,717 | 4,874 | 222,899 | 0 |
| has_digit | 199,652 | 2,894,131 | 71,195 | 1,905,166 |
| has_ampersand | 147,716 | 320 | 69,221 | 36,323 |
| has_comma | 359,325 | 3,059,088 | 0 | 2,050,598 |
| has_hyphen | 191,629 | 228,171 | 103,799 | 823,146 |
| has_slash | 43,962 | 93,547 | 43,286 | 716,377 |
| has_hash | 23,975 | 322,065 | 11,843 | 220,976 |
| has_parenthesis | 111,338 | 1,250 | 158,350 | 85,271 |
| has_apostrophe | 100,904 | 2,471 | 35 | 7,205 |
| uppercase_only_cased_letters | 101,566 | 1 | 54,653 | 812 |
| lowercase_only_cased_letters | 219,809 | 0 | 110,413 | 8 |
| url_like | 133,786 | 0 | 77,136 | 71 |
| embedded_tab_or_newline | 0 | 0 | 0 | 0 |

**test_source1**

| Property | US/business_name | US/business_address | France/business_name | France/business_address | India/business_name | India/business_address |
| --- | --- | --- | --- | --- | --- | --- |
| literal_placeholder | 0 | 0 | 0 | 0 | 0 | 0 |
| leading_or_trailing_whitespace | 0 | 0 | 0 | 0 | 0 | 0 |
| repeated_whitespace | 0 | 472 | 0 | 281 | 0 | 0 |
| has_digit | 17,142 | 663,105 | 2,002 | 258,363 | 1,063 | 739,238 |
| has_ampersand | 36,969 | 75 | 15,415 | 0 | 34,533 | 17,883 |
| has_comma | 84,511 | 663,106 | 18 | 259,452 | 0 | 809,986 |
| has_hyphen | 5,474 | 7,507 | 5,742 | 214,876 | 1,980 | 313,786 |
| has_slash | 2,633 | 1,217 | 8 | 8 | 32 | 276,438 |
| has_hash | 674 | 2,283 | 0 | 0 | 1 | 10,993 |
| has_parenthesis | 15 | 420 | 20,793 | 114 | 41,761 | 42,895 |
| has_apostrophe | 23,596 | 574 | 0 | 14,583 | 16 | 3,927 |
| uppercase_only_cased_letters | 0 | 2 | 3 | 0 | 0 | 0 |
| lowercase_only_cased_letters | 0 | 0 | 0 | 0 | 0 | 0 |
| url_like | 0 | 0 | 0 | 0 | 0 | 41 |
| embedded_tab_or_newline | 0 | 0 | 0 | 0 | 0 | 0 |

**test_source2**

| Property | India/business_name | India/business_address | France/business_name | France/business_address | US/business_name | US/business_address |
| --- | --- | --- | --- | --- | --- | --- |
| literal_placeholder | 0 | 0 | 45 | 0 | 4 | 0 |
| leading_or_trailing_whitespace | 0 | 0 | 0 | 0 | 0 | 0 |
| repeated_whitespace | 225,009 | 29,205 | 64,090 | 568 | 211,138 | 52,966 |
| has_digit | 66,575 | 2,151,419 | 7,818 | 655,185 | 115,616 | 1,722,639 |
| has_ampersand | 78,246 | 46,804 | 41,315 | 0 | 89,047 | 148 |
| has_comma | 0 | 2,259,799 | 43 | 681,661 | 218,465 | 1,816,223 |
| has_hyphen | 89,784 | 980,709 | 33,009 | 334,486 | 100,833 | 140,701 |
| has_slash | 29,267 | 827,035 | 25 | 20 | 7,083 | 58,052 |
| has_hash | 10,383 | 268,834 | 1,932 | 23,368 | 13,105 | 99,460 |
| has_parenthesis | 167,076 | 107,625 | 60,913 | 11,936 | 62,269 | 694 |
| has_apostrophe | 42 | 10,016 | 0 | 37,919 | 59,672 | 1,504 |
| uppercase_only_cased_letters | 326,569 | 555,038 | 146,337 | 200,304 | 382,091 | 1,693,176 |
| lowercase_only_cased_letters | 87,656 | 0 | 45,987 | 0 | 111,975 | 0 |
| url_like | 63,588 | 123 | 24,565 | 0 | 67,887 | 0 |
| embedded_tab_or_newline | 0 | 0 | 0 | 0 | 0 | 0 |

**test_source3**

| Property | India/business_name | India/business_address | France/business_name | France/business_address | US/business_name | US/business_address |
| --- | --- | --- | --- | --- | --- | --- |
| literal_placeholder | 11 | 0 | 48 | 0 | 2 | 0 |
| leading_or_trailing_whitespace | 0 | 0 | 0 | 0 | 0 | 0 |
| repeated_whitespace | 248,382 | 0 | 58,135 | 42 | 213,079 | 3,084 |
| has_digit | 76,204 | 2,209,689 | 8,094 | 683,217 | 118,172 | 1,807,917 |
| has_ampersand | 79,565 | 42,557 | 41,780 | 0 | 91,782 | 218 |
| has_comma | 0 | 2,345,759 | 37 | 709,921 | 224,993 | 1,890,384 |
| has_hyphen | 107,068 | 945,448 | 33,490 | 357,850 | 108,679 | 142,470 |
| has_slash | 46,460 | 821,945 | 4,081 | 12 | 23,586 | 58,793 |
| has_hash | 11,755 | 265,511 | 1,936 | 23,394 | 13,145 | 202,199 |
| has_parenthesis | 180,678 | 97,756 | 62,025 | 11,915 | 66,876 | 766 |
| has_apostrophe | 42 | 8,268 | 0 | 39,407 | 62,223 | 1,570 |
| uppercase_only_cased_letters | 59,415 | 914 | 41,029 | 0 | 62,173 | 0 |
| lowercase_only_cased_letters | 110,581 | 10 | 48,137 | 0 | 122,496 | 0 |
| url_like | 71,125 | 113 | 25,101 | 0 | 67,768 | 0 |
| embedded_tab_or_newline | 0 | 0 | 0 | 0 | 0 | 0 |

## Noise patterns and true-pair examples

The supplied README names legal-suffix changes, abbreviations, trade/DBA names, word-order swaps, punctuation, typos, transliterations, partial addresses, landmarks, missing postal/state components, and municipal numbering. The data also visibly contain URL-like names, inserted business descriptors, mixed scripts, zero-padded numbers, letter/number combinations, and reordered address components. No noise-generator specification or authoritative per-noise labels were supplied, so their exact semantic rates are unknown.

The following measurements cover **all 41,396 true links in the 12,000-anchor pilot**, including blocking misses. They are not full-corpus percentages.

| Country/source | True pairs | Raw names equal | Normalized names equal | No shared name tokens | Target address blank | First number conflict / both have numbers |
| --- | --- | --- | --- | --- | --- | --- |
| US/S3 | 12,766 | 764 | 3,836 | 1,019 | 589 | 1,878 / 11,279 |
| US/S2 | 11,954 | 755 | 3,644 | 910 | 620 | 1,743 / 10,366 |
| India/S3 | 8,569 | 228 | 1,702 | 1,668 | 387 | 1,962 / 7,325 |
| India/S2 | 8,107 | 225 | 1,397 | 2,229 | 291 | 1,898 / 7,084 |

A first-number conflict refers to the first regex-extracted numeric token, not a verified house-number disagreement. Component reordering and zero padding can cause a conflict; true labelled links with conflicts show why a blanket numeric veto is unsafe.

Example of `true_match_with_first_number_conflict` (provided labels say these are the same entity):

| Record | ID | Name | Address |
| --- | --- | --- | --- |
| anchor | S1-170637976 | South Associates | 6747 Charlesgate Road, Huber Heights, OH |
| target | S3-316175985 | South Associates | 1747 Charlesgate Rd, Dyaton, Ohio |

Example of `cross_script_name` (provided labels say these are the same entity):

| Record | ID | Name | Address |
| --- | --- | --- | --- |
| anchor | S1-942129107 | City Industries Pvt Ltd | Plot No. 146, Gf, Pkt 22, Sec-24, Rohini, Delhi, North West Delhi, Delhi |
| target | S2-502470575 | सिटी इंडस्ट्रीज प्रा. लि. | H.NO #146, GF, PKT 22, SEC-24, ROHINI, DELHI, NORTH WEST DELHI, Delhi |

Example of `missing_target_address` (provided labels say these are the same entity):

| Record | ID | Name | Address |
| --- | --- | --- | --- |
| anchor | S1-942129107 | City Industries Pvt Ltd | Plot No. 146, Gf, Pkt 22, Sec-24, Rohini, Delhi, North West Delhi, Delhi |
| target | S3-666933734 | City Pvt Industries Ltd | (empty) |

## Scale and evaluation

| Split | Unrestricted S1 × (S2+S3) comparisons | Even after country-only blocking |
| --- | --- | --- |
| train | 22,774,876,013,799 | 11,839,670,856,657 |
| test | 17,272,751,604,416 | 6,724,569,566,212 |

The official objective is macro F0.5 across Source 1 entities. For nonempty true match sets it is `1.25*TP / (1.25*TP + FP + 0.25*FN)`. Empty truth and empty prediction score 1; empty truth with any prediction scores 0. A nonempty truth with an empty prediction scores 0. Each anchor contributes equally, regardless of its number of matches. The formula, rather than the informal 'twice' wording, governs the false-merge/miss tradeoff.

There is no provided test ground truth or official local scoring script. `utils/validate_submission.py` checks format, not model quality. We implemented the stated score ourselves and tested the singleton and missed-candidate cases. Public/private leaderboard subsets, their sizes, and their label distributions are not provided in the supplied files.

## What has already been trained

| Pilot fact | Measured result |
| --- | --- |
| Anchors | 12,000 |
| Target fragments | 81,227 |
| Actual scored candidate pairs | 676,932 |
| Average candidates/anchor | 56.411 |
| Audit entities | 2,359 |
| Audit macro F0.5 | 0.985374 |
| Audit blocking recall | 0.996104 |
| Audit pair precision | 0.993415 |
| Audit pair recall | 0.973338 |
| Audit singleton accuracy | 0.983051 |
| US-only training → India audit | 0.913658 |
| India-only training → US audit | 0.981854 |

The pilot fits 46 lexical/number/retrieval features with LightGBM after complementary name-character, address-character and combined-token TF-IDF blocking. Whole anchors are split, with duplicate normalized anchor signatures/shared targets grouped. Thresholds and early stopping use tuning entities; audit is separate. Candidate corpus size is reduced, so the audit score is not a leaderboard estimate. Following manual inspection of these audit errors, a fresh holdout is needed for future final comparisons.

## Submission and competition constraints

`matching_results.tsv` requires `source1_entity_id` and `matched_entity_ids`. `candidate_pairs.tsv` requires `source1_entity_id` and `candidate_entity_ids`. Both are UTF-8 tab-separated files with one row for every test Source 1 entity, comma-separated S2/S3 IDs without duplicates, and empty second fields for empty sets. Candidates must be the exact set actually fed to the matcher, and final matches must be a subset. Only the matching file is leaderboard-scored. The final archive additionally needs both output files, complete runnable code, pinned dependencies, and a completed methodology document.

The validator defaults to skipping target-ID existence checks; enable `--check-ids`. It warns rather than fails when matches are absent from candidates, so the pipeline also needs a strict containment assertion. The README and validator comments differ on whether nonexistent IDs are rejected or merely hurt the score; always emit existing IDs to satisfy the stricter rule. A formatting PASS is not evidence of matching accuracy.

The supplied rules require a final model under MIT/Apache 2.0 licensing and at most 8 billion parameters. They prohibit external business databases, registration lookups, geocoding APIs, entity-resolution services, and internet-derived task-data augmentation. The model-card/documentation checks used here do not look up business identities.

## Still unknown or unverified

We do not know test labels, test singleton prevalence, test match multiplicity, the public/private split, French matching accuracy, true language identity of every record, real-world branch semantics beyond the supplied labels, generator/provenance details, official scoring implementation edge cases beyond the documented rules, full-corpus candidate recall, final full-test runtime, the optimal neural checkpoint, GPU throughput, or transformer token truncation rates. Exact ID/raw-text checks do not resolve semantic train/test leakage. We have not trained the transformer or generated full-test predictions.

## Audit provenance and reproduction

Full audit elapsed time: 650.42 seconds. Supporting compact NumPy arrays allow exact ID joins without loading all record text into RAM. The original TSV files were read only.

| Source file | UTF-8 BOM | SHA-256 |
| --- | --- | --- |
| train_source1.tsv | False | 591af0e1dfeb65cab71ea6ee8cb69df00f92d6ba6fa79e05746c938775d14973 |
| train_source2.tsv | False | 6336c1a055eec79cf8a6d99fdc8d32a2e4d9dc2662e00963cb35d66b89ed09ed |
| train_source3.tsv | False | 67da22f5151898ff3006febd836c1a159e97ae95efa7257a5aff4fda685e58e9 |
| test_source1.tsv | False | 3d4a32c54c2ca9c53fd7c2be105bf26f708f94c4d2f88eb370972a195665c2f5 |
| test_source2.tsv | False | 79d906c7497af2ace70aa277f6e334a652094909de99bd6c57b53420b6a7b2dd |
| test_source3.tsv | False | 850942b11d2a4343486ed0834e28bce9f3b385f3fd497fd60ccf4ea3b8bda035 |

Run from `student_resource/` with the prepared Python environment:

```bash
python code/business_entity_resolution/src/audit_dataset.py --data dataset --out artifacts/dataset_audit
python code/business_entity_resolution/src/audit_pilot_pairs.py --data artifacts/pilot/train --out artifacts/dataset_audit/pilot_positive_pairs.json
python code/business_entity_resolution/src/build_dataset_report.py --audit artifacts/dataset_audit/dataset_audit.json --pilot-report artifacts/baseline_v1/report.json --out DATASET_INVENTORY.md
```
