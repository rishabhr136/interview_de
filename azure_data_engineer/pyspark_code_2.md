LEFT ANTI JOIN vs NOT IN — The NULL Trap
🎯 Interview Scenario

Give the candidate:

from pyspark.sql import SparkSession


spark = SparkSession.builder.getOrCreate()


data_users = [
    ("usr_101", "US"),
    ("usr_102", "UK"),
    ("usr_103", "CA")
]


data_flagged = [
    (None,),
    ("usr_102",)
]


df_users = spark.createDataFrame(
    data_users,
    ["user_id", "country"]
)


df_flagged = spark.createDataFrame(
    data_flagged,
    ["user_id"]
)


# Strategy A: Relational API Anti-Join
res_anti_join = df_users.join(
    df_flagged,
    df_users.user_id == df_flagged.user_id,
    "left_anti"
)


# Strategy B: Spark SQL NOT IN
df_users.createOrReplaceTempView("users")
df_flagged.createOrReplaceTempView("flagged")


res_sql_subquery = spark.sql("""
    SELECT *
    FROM users
    WHERE user_id NOT IN (
        SELECT user_id
        FROM flagged
    )
""")


print("Strategy A Count:", res_anti_join.count())
print("Strategy B Count:", res_sql_subquery.count())
🔥 Ask

"What will these two counts be? Will they be the same? Explain why."

Expected Answer

No. The counts will be different because the flagged table contains a NULL.

Expected:

Strategy A Count: 2
Strategy B Count: 0
1. Strategy A — LEFT ANTI JOIN
res_anti_join = df_users.join(
    df_flagged,
    df_users.user_id == df_flagged.user_id,
    "left_anti"
)

Users:

user_id
--------
usr_101
usr_102
usr_103

Flagged:

user_id
--------
NULL
usr_102

usr_102 has a matching record, so it is removed.

usr_101 and usr_103 have no matching records, so they remain.

Result
user_id   country
--------  -------
usr_101   US
usr_103   CA

Therefore:

Strategy A Count = 2
2. Strategy B — NOT IN

The SQL is:

WHERE user_id NOT IN (
    SELECT user_id
    FROM flagged
)

The subquery returns:

NULL
usr_102

The presence of NULL changes the behavior because SQL uses three-valued logic:

TRUE
FALSE
UNKNOWN

For example:

usr_101 NOT IN (NULL, usr_102)

cannot be determined to be definitely true because one value in the comparison set is NULL.

The result becomes effectively:

UNKNOWN

A WHERE clause keeps only rows where the condition is TRUE.

Therefore, the rows are filtered out.

Strategy B Count = 0
🧠 Key Concept

The important distinction is:

LEFT ANTI JOIN
       ↓
Find rows with NO matching row
       ↓
NULL in right table does not automatically eliminate all left rows




NOT IN
       ↓
SQL three-valued logic
       ↓
NULL in subquery
       ↓
UNKNOWN
       ↓
WHERE removes UNKNOWN
🔥 Follow-Up

Ask:

"Why does NOT IN behave differently when the subquery contains NULL?"

Strong Answer

Because SQL does not treat:

NULL = NULL

as TRUE.

Instead:

NULL = NULL
    ↓
UNKNOWN

Therefore, when NULL appears inside a NOT IN subquery, comparisons can evaluate to UNKNOWN.

🔥 Killer Follow-Up

Ask:

"How would you safely use NOT IN if the business requirement requires it?"

Expected Answer

Explicitly remove NULLs from the subquery:

SELECT *
FROM users
WHERE user_id NOT IN (
    SELECT user_id
    FROM flagged
    WHERE user_id IS NOT NULL
)

Now the subquery contains only:

usr_102

and the result becomes:

usr_101
usr_103
🧠 Advanced Follow-Up

Ask:

"Are LEFT ANTI JOIN and NOT IN always interchangeable?"

Expected Answer

No.

They can produce different results when NULL values are involved.

A candidate should understand that semantically similar SQL operations can have different behavior because of NULL semantics.

🚨 Red Flags

Be cautious if the candidate says:

❌ "Both queries always return the same result."
❌ "NULL is treated as another value."
❌ "NULL = NULL is TRUE."
❌ "NOT IN ignores NULL."
❌ "The anti-join will also return zero rows."
❌ They cannot explain three-valued logic.
❌ They solve it by blindly using DISTINCT.
⭐ Excellent Candidate Answer

"LEFT ANTI JOIN and NOT IN are not always equivalent when NULLs are present. Here, the flagged table contains usr_102 and NULL. The anti-join removes only usr_102, so it returns 2 rows. The NOT IN subquery contains NULL, which causes SQL's three-valued logic to produce UNKNOWN for the comparisons, and the WHERE clause only retains TRUE. Therefore, the NOT IN query returns 0 rows. If I use NOT IN, I would explicitly filter out NULLs from the subquery or use an anti-join depending on the required semantics."

🎯 What This Question Tests
Concept	What You're Testing
LEFT ANTI JOIN	Do they understand anti-joins?
NOT IN	Do they understand subquery behavior?
NULL	Do they understand NULL semantics?
Three-Valued Logic	Do they know TRUE / FALSE / UNKNOWN?
PySpark SQL	Can they reason across APIs?
Query Correctness	Can they identify semantic differences?
Data Quality	Do they recognize NULL-related issues?

Interviewer's goal: Don't stop at "What is the output?" Ask "Why are the outputs different?" This reveals whether the candidate genuinely understands SQL NULL semantics and anti-join behavior, rather than simply memorizing PySpark syntax.



Implicit Type Coercion — Silently Wrong Numbers
🎯 Interview Scenario

Give the candidate:

from pyspark.sql import functions as F


df = spark.createDataFrame([
    (1, "100"),
    (2, "200.50"),
    (3, "abc"),
    (4, None)
], ["id", "amount_str"])


df2 = df.withColumn(
    "amount_int",
    F.col("amount_str").cast("int")
)


df2.show()


df2.agg(
    F.sum("amount_int")
).show()
🔥 Ask

"What do you expect amount_int to contain, and what will the SUM be? More importantly, is this transformation safe?"

Expected Answer

The candidate should immediately notice that:

amount_str

is a string, while the pipeline is attempting to cast it directly to:

int

The input contains three different cases:

"100"
"200.50"
"abc"
NULL

The important issue is that not every value is a valid integer.

1. What Happens During the Cast?

The transformation is:

F.col("amount_str").cast("int")

Conceptually:

"100"    → 100
"200.50" → invalid/incompatible for direct integer conversion
"abc"    → invalid
NULL     → NULL

Depending on the Spark version and ANSI SQL configuration, invalid casts may either produce NULL or raise an error.

A strong candidate should not blindly assume that every Spark environment silently converts invalid values to NULL.

🔥 Important Follow-Up

Ask:

"If invalid values become NULL, what happens to the SUM?"

SUM() ignores NULL values.

So if the cast produces:

100
NULL
NULL
NULL

then:

SUM = 100

The dangerous part is that the pipeline may appear to work:

Job succeeded
      ↓
SUM = 100

while the original data contained:

"200.50"
"abc"

which were not successfully represented in the integer column.

🚨 Why Is This Dangerous?

The biggest issue isn't necessarily a Spark error.

The problem is silent data loss or incorrect business numbers.

Suppose the business expects:

100 + 200.50 = 300.50

but the pipeline produces:

100

because "200.50" failed the integer conversion.

The pipeline can technically succeed while producing an incorrect result.

🧠 Killer Follow-Up

Ask:

"Would you cast amount_str directly to int if this were a financial pipeline?"

Strong Answer

No.

First determine what the source field represents.

If it represents monetary values, an integer may be the wrong data type.

For example:

"200.50"

should generally not be represented as:

200

because that loses the decimal component.

A more appropriate representation may be:

F.col("amount_str").cast("decimal(18,2)")
✅ Safer Transformation

For a monetary amount:

df2 = df.withColumn(
    "amount",
    F.col("amount_str").cast("decimal(18,2)")
)

Conceptually:

"100"    → 100.00
"200.50" → 200.50
"abc"    → invalid
NULL     → NULL

Then the pipeline can explicitly decide what to do with invalid values.

🔎 Advanced Follow-Up

Ask:

"How would you detect bad records instead of silently losing them?"

A strong candidate should create a validation/quarantine path.

For example:

df2 = df.withColumn(
    "amount",
    F.col("amount_str").cast("decimal(18,2)")
)

Then investigate records where:

amount_str IS NOT NULL
AND amount IS NULL

Conceptually:

Raw Data
   │
   ▼
Type Conversion
   │
   ├───────────────┐
   ▼               ▼
Valid             Invalid
   │               │
   ▼               ▼
Continue        Quarantine
                   │
                   ▼
                 Alert
🔥 Another Killer Question

Ask:

"What if the source changes tomorrow and sends "1,200.50" instead of "1200.50"?"

A strong candidate should recognize that the comma may prevent a direct numeric cast.

They may first need to normalize the value:

df2 = df.withColumn(
    "amount",
    F.regexp_replace(
        F.col("amount_str"),
        ",",
        ""
    ).cast("decimal(18,2)")
)

Now:

"1,200.50"
      ↓
"1200.50"
      ↓
1200.50

The key point is that data-quality rules should be explicit, not assumed.

🚨 Red Flags

Be cautious if the candidate says:

❌ `"200.50" will become 200, so everything is fine."
❌ `"abc" will definitely become 0."
❌ "SUM() will include NULL as zero."
❌ "Casting strings to integers is always safe."
❌ They don't consider financial precision.
❌ They don't consider invalid source values.
❌ They don't think about ANSI mode / Spark version behavior.
❌ They don't propose validation or quarantine.
⭐ Excellent Candidate Answer

"amount_str is a string, and directly casting it to integer is unsafe. "100" can convert to 100, but "200.50" isn't an appropriate integer representation and "abc" is invalid. Depending on Spark's ANSI configuration and version, invalid casts may result in NULL or raise an error. If invalid values become NULL, SUM() ignores them, which can make the pipeline succeed while producing an incorrect total. For a monetary field I'd use an appropriate decimal type such as decimal(18,2), validate conversion failures explicitly, and quarantine or alert on invalid records rather than silently losing them."

🎯 What This Question Tests
Concept	What You're Testing
Data Types	Do they understand string vs numeric types?
Casting	Do they understand conversion behavior?
ANSI Mode	Do they know invalid casts can behave differently?
NULL Semantics	Do they know how SUM() handles NULL?
Data Quality	Can they detect invalid values?
Financial Data	Do they understand decimal precision?
Silent Data Loss	Can they recognize a pipeline that succeeds incorrectly?
Production Thinking	Do they validate rather than blindly cast?

Interviewer's goal: Don't focus only on "What does cast('200.50' as int) do?" The real test is whether the candidate recognizes the much bigger Data Engineering problem: a pipeline can complete successfully while producing financially incorrect numbers because invalid or lossy type conversions were not validated.


