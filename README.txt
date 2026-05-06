

				DEPENDENCIES
			~~~~~~~~~~~~~~~~~~~~~~~~~~~


To run the data preparation notebook, use a Python kernel/runtime with the following packages installed:

- numpy
- pandas
- regex
- rapidfuzz
- tqdm

The notebook also uses standard Python libraries such as math, random, gc, os, and collections. 
These are part of the Python standard library and do not need to be installed separately.



				Instructions
			~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. Download the raw UK Sanctions List CSV file from the official GOV.UK sanctions list publication 
page
2. Place the raw CSV file in the same working directory as the notebook
3. Open analysis.ipynb in Jupyter Notebook, JupyterLab, VS Code, or another compatible notebook
environment.
4. Run the notebook from start to finish.
5. The notebook prepares the sanctions data into several cleaned output CSV files designed for
customer/sanctions matching.

The main generated datasets are:

- name_index.csv
- individuals.csv
- entities.csv
- ships.csv
- sanctions.csv

The transformed structure uses a generated SEID field, or Sanctioned Entity ID, to link records across the output datasets. (primary & foreign key)


	              Data Quality Insights & Observations
	        ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Below is a summary of the main findings in the analysis.ipynb notebook. For more details, please refer to the notebook.


Overall approach
----------------

The raw UK Sanctions List contains individuals, entities, and ships in one source file. These target types have different useful matching fields, so the notebook restructures the raw data into separate matching-focused datasets.

The core design is:

- name_index.csv: a central name lookup table containing names and related names linked to SEID.
- individuals.csv: individual-specific fields such as surname, given names, date of birth, gender, nationality, national identifiers, address, phone, email, and passport information.
- entities.csv: organisation-specific fields such as entity name, business registration number, subsidiaries, parent company, address, website, and email.
- ships.csv: ship-specific fields such as ship name, IMO number, flag, tonnage, length, year built, and current/previous owners or operators.
- sanctions.csv: sanctions metadata such as regime name, sanctions imposed, date designated, and last updated.

This design supports two matching routes. If the customer record type is known, matching can be performed against the relevant individuals, entities, or ships dataset. If the record type is unknown but a name is available, the name_index.csv file can be used to identify candidate SEID values and designation types before checking the relevant detailed dataset.


Field selection observations
----------------------------

The Unique ID field from the source data is retained as a legacy/source reference key. It is not useful for direct customer matching, because customer records are unlikely to contain this ID, but it is useful for traceability back to the original sanctions source.

The OFSI Group ID and UN Reference Number fields were not retained as core fields. They were mostly missing or did not add useful differentiating power beyond the source unique identifier. (many missing values too, unlike the Unique ID)

The Designation Type field is useful because it identifies whether a record relates to an individual, entity, or ship. It is not itself a matching field, but it helps route matching and validation, as well as guiding the creation of the aforementioned narrower/normalised 3 datasets

The Regime Name and Sanctions Imposed fields are useful for context and post-match decision-making rather than direct customer matching.

Other Information and UK Statement of Reasons are dense descriptive text fields. They are not primary matching fields, but they provide useful context for manual review and may support future enrichment via imputation.


Name fields
------------

The raw dataset stores names across Name 1 to Name 6. For individuals, Name 6 is treated as the surname and Name 1 to Name 5 are treated as given or middle names. For entities and ships, Name 6 is usually the main name.

A full name is constructed for matching and included in the name_index.csv output. Additional names from fields such as non-Latin names, subsidiaries, parent company, and current/previous ship owners are also extracted into the name index where useful.

We observed that some individuals have their full name in Name 1 rather than using the expected surname field. Some records have only a non-Latin script name, and some have only Latin-script names.

Some names are unusually short, but these were not automatically removed because they may be valid aliases or acronyms.

There were 40 remaining missing-name records after initial processing. These records were mostly incomplete Haji records from Kabul/Afghanistan with repeated information and insufficient name data. These were tracked for removal across datasets.

Non-Latin script names
----------------------

Non-Latin script names are sparse but useful where present, especially for robust name matching.

The Non-Latin Script Type and Non-Latin Script Language fields contain many missing values. Script type can sometimes be inferred using Unicode script detection, but language is much harder to infer reliably from a name alone. For example, Cyrillic can correspond to several languages, and Han characters should not automatically be assumed to mean Chinese language (could also be japanese for example).

Name type and alias strength
----------------------------

Name Type contained inconsistent values such as casing variations and spelling inconsistencies. These are standardised through mapping.

Alias Strength is not consistently populated. It may be useful in a more advanced match-confidence model, but cannot always be imputed reliably.

For entities and ships, Alias Strength was effectively unused or fully missing in some outputs, so it was dropped where it did not add value.

Individual-specific observations
-------------------------------

D.O.B is not always a complete date. Some records contain unknown day or month values, such as dd/mm/1955, while some contain only a year. One value contained an unexpected apostrophe. We preserve partial date information rather than forcing unreliable imputation

Gender had inconsistent formatting, including casing issues. Obvious titles such as Mr, Mrs, Ms, and Miss were used as a low-confidence source for imputing missing gender where appropriate.

Title is mostly missing and often generic, but it is retained because it may sometimes help distinguish individuals or provide useful context.

Position required minor cleaning, including conversion of empty strings to missing values and correction of whitespace formatting issues.

Nationalities appeared generally well formatted, but missing nationality is not reliably imputed because nationality rules vary by country and cannot be inferred safely from other fields in many cases.

Birth Country contains naming inconsistencies such as references to former USSR/Russian Federation variants. These were not aggressively standardised because the historical wording may be meaningful.

Birth Town could potentially be used with geocoding to infer missing birth countries, but this was not used in the final pipeline due to ambiguity in town names and dependency on external services/rate limits.

Address fields
--------------

Address lines 1 to 6 are combined into a single Address Lines field, separated by semicolons. This creates a compact multi-value address field while avoiding many sparse address columns.

Because of the concatenation process, empty address components created sequences such as ; ; ; ; ;. These are cleaned so that empty separator-only values are converted back to missing values and useful address components are retained.

Postal codes are conservatively cleaned. PO Box variants such as PO BOX, P.O. Box, and similar forms are standardised. Apostrophes and unusual formatting characters are removed where unlikely to be meaningful.

Some postal-code values remain ambiguous. For example, values that appear to contain both town and postal-code information are not aggressively split because global postal-code formats vary.

Address Country appeared generally well formatted.

Phone numbers
------------

Phone number fields contain substantial formatting variation. Some cells include multiple numbers and associated dates, such as numbers followed by date ranges. These dates may be meaningful, so they are not removed aggressively. Additionally, using characters like - or whitespaces to separate different segments of the phone numbers was present, as well as the use of parentheses, which may all be required formats for specific country/phonebook rules.

The cleaning process focuses on conservative formatting fixes: Unicode whitespace is normalised, repeated spaces are collapsed, and unlikely apostrophes are removed.

Phone numbers remain sparse but are retained because they may be valuable secondary matching criteria.

Websites and emails
-------------------

Website fields contain inconsistent separators, including and, |, and special Unicode whitespace. These are standardised conservatively, usually using semicolon separators for multiple values.

Email fields contain mixed casing and multiple emails separated by and. Emails are lowercased and separators are standardised. Some entity email fields contained website-like values without @, which were identified as data quality issues.

National identifiers and passports
------------------------------------

National identifier numbers and passport numbers are highly variable because different jurisdictions use different formats. The notebook applies conservative cleaning only, such as trimming whitespace, normalising Unicode spaces, and removing unlikely apostrophes.

The notebook avoids forcing these fields into numeric-only formats because valid identifiers may contain letters, punctuation, or country-specific structures.

Identifier information fields may contain useful context and could potentially support future country inference.

Entity-specific observations
----------------------------

Entity names may be short because acronyms are valid organisation names. A small number of entity records were missing Latin-script names but had non-Latin names, so these were still usable for matching.

Entity address and contact fields had similar formatting issues to individuals.

Business registration numbers were poorly standardised, likely because different countries use different registration formats. These are cleaned conservatively to avoid destroying meaningful information.

Entity type values mostly needed only whitespace cleaning. Some labels may be semantically similar, such as Research Institute and Research, but this is not automatically merged because the equivalence is ambiguous.

Subsidiaries and parent company fields are useful for entity name screening. They are not aliases of the same entity, but they may still need to be flagged by a bank because they represent related parties.

Some subsidiaries fields contain numbered lists where order may be meaningful, so the notebook avoids overly aggressive delimiter replacement. Some alternate-spelling patterns are safely transformed into semicolon-delimited related names. (eg. when subsidiaries are provided with alternate spellings: ...)

Ship-specific observations
--------------------------

Ship names were not missing and were generally well formatted. Some ship names contain slash-separated alternatives, which may indicate multiple names but are not aggressively altered without certainty.

Ship non-Latin script fields were effectively null in the current data and were dropped for storage efficiency, with the option to reintroduce them if future source data makes them useful.

IMO numbers appear either as a 7-digit value or as IMO followed by 7 digits. The IMO prefix is removed to leave the 7-digit identifier for consistency.

Current and previous flag fields were already well formatted. Some missing values remains, but flags are retained because they are useful for ship matching and validation.

Ship type had minor label inconsistency, such as Oil Tanker versus Oil tanker, which was standardised.

Tonnage, length, and year built were already formatted appropriately.

Previous owner/operator fields contain many missing values, but they are retained where present because they can support matching or review.

Sanctions metadata observations
-------------------------------

Regime Name contained 31 distinct values and was already well formatted. Some domain specific knowledge here is required to discern if different regimes can/should be consolidated into one, eg. given similar dates but one specifying (EU Exit)

Sanctions Imposed is frequently a multi-value field. Exploding it would improve atomicity, and the large storage overhead turned out to mainly arise from the large number of duplicates.

Date Designated and Last Updated were already formatted consistently and did not have the partial-date issues seen in D.O.B.

Duplicate handling
-------------------

Duplicate handling is performed after cleaning so that values are compared in a more standardised form, though there was a memory overhead with this approach. Whilst it isn't wrong to deduplicate records after data cleaning, sometimes it can help doing it a bit first before cleaning, and then again after. Same logic applies to data cleaning in relation to data transformation

The deduplication approach first focuses on individuals, entities, and ships. These are the core target datasets, and their duplicate decisions can be propagated to dependent datasets such as name_index.csv and sanctions.csv using the new global SEID mappings.

The duplicate-identification function compares selected columns using fuzzy similarity thresholds. A pair is only considered a possible duplicate when all specified field thresholds are met.

For individuals, duplicate detection relies on high similarity of surname, given names, non-Latin names, and exact matches on fields such as date of birth and gender. National identifier and passport number similarities are also treated as strong duplicate evidence.

For entities, duplicate detection focuses on names, non-Latin names, subsidiaries, parent company, and business registration number. Business registration formats vary, so this field requires careful interpretation. Indeed, it is a source for possible further duplicate consolidation in the future, since some entries only differed in the column and could be condensed further (unless atomicity in this field is preferred for the downstream system)

For ships, IMO number is a very strong distinguishing field. Ship name and ship type are also considered as high similarity fields.

Pairwise duplicate matches are converted into duplicate groups using a connected-components approach. This means if A matches B and B matches C, then A, B, and C are treated as one candidate duplicate group. As a result, some expressive power is lost here and improper tuning of the fuzzy thresholds could cause a strong negative feedback loop removing valid data

Canonical keys are selected consistently to avoid unstable mappings in chunked processing (cycles). Duplicate records are consolidated into canonical records, and selected fields are combined only when values differ enough to preserve potentially useful information (this is tunable via column specific fuzz thresholds too).

name_index.csv and sanctions.csv are deduplicated more directly by dropping exact duplicate rows, because they are dependent/index-style datasets, and make great use of the final identified global SEID mappings from the deduplication process on the main 3 datasets before it.

Chunking is used for memory reasons because pairwise fuzzy matching has O(n^2) complexity. This may miss duplicates that appear in different chunks, so this is noted as a limitation and area for future improvement. Indeed, the chunks parameter for each dataset can be set to maximally prevent this  limitation, eg. using fewer number of chunks on ships.csv which is several times smaller than the entities.csv dataset

Final checks suggested that entities had the largest number of duplicated records, while ships had the fewest. Most sanctions imposed on ships were unspecified shipping sanctions, creating some redundancy in the sanctions metadata.

Limitations and future work
---------------------------

Some sparse fields are retained because they may be highly valuable when present, even if missing for most records. Removing this can help with storage requirements

Geocoding-based enrichment was considered but not included due to ambiguity, external service dependency, and rate limits.

Language inference (e.g. LLM-based) from non-Latin names was not treated as reliable because script does not uniquely identify language.

Fuzzy deduplication is conservative because similar names may refer to different sanctioned targets. The notebook favours preserving information over aggressive deletion.

A future version could use blocking or repeated passes to improve cross-chunk deduplication, use richer/more nuanced logic and thresholds,

As mentioned, NLP/LLM methods can also be used to extract structured country, identifier, and relationship information from descriptive text fields.

With more domain specific knowledge and context regarding the fictitious bank's use of the customer sanctions screening algorithm's components, the datasets can be further optimised across duplicate records AND field formatting requirements
