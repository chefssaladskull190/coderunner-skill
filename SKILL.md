---
name: moodle-coderunner
version: 5.0.0
description: |
  Generate Moodle CodeRunner programming questions with Jobe server validation.
  Supports Java, Python, C, C++, Node.js. Produces import-ready Moodle XML.
  Battle-tested: 1000+ questions validated with 0% failure rate using these rules.
  Optimized: AI generates compact JSON, Python wraps in XML boilerplate (saves 70% tokens).
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - AskUserQuestion
---

# CodeRunner Question Generator

Generate Moodle CodeRunner questions. Every question MUST be validated on Jobe before delivery.

**Preferred method:** Think through each question as JSON first (name, type, solution, test cases), then write the full XML using the template in Section 7. This JSON-first approach improves accuracy.

---

## 1. Question Type Reference

Choose the type FIRST — it determines everything else.

| Type | Student Writes | Test Mechanism | Stdin? | Reliability |
|------|---------------|----------------|--------|-------------|
| `java_class` | A class | Test code calls methods | No | HIGH |
| `java_method` | Static method(s) ONLY | Test code calls methods | No | HIGH |
| `java_program` | Full program with `main` | Empty test, stdin input | Yes | LOW — EOF issues |
| `python3` | Function(s) or class | Test code calls with `print()` | No | HIGH |
| `python3_w_input` | Full program | Empty test, stdin input | Yes | MEDIUM — EOF issues |
| `c_function` | Function(s) + headers | Test code has `main()` | No | HIGH |
| `c_program` | Full program with `main` | Empty test, stdin input | Yes | MEDIUM |
| `cpp_function` | Function(s) + headers | Test code has `main()` | No | MEDIUM — Werror |
| `cpp_program` | Full program with `main` | Empty test, stdin input | Yes | MEDIUM — Werror |
| `nodejs` | Function(s) | Test code uses `console.log()` | No | HIGH |

**Default choice**: Prefer `java_class` > `java_method` > `java_program`. Prefer function types over program types — they avoid stdin/EOF problems entirely.

---

## 2. Per-Language Rules

These rules are derived from 1000+ validated questions. Every rule exists because questions failed without it.

### Java

| Rule | Applies To | What To Do |
|------|-----------|------------|
| Scanner EOF guard | `java_program` | ALWAYS call `hasNextLine()`/`hasNextInt()` before EVERY `Scanner` read. #1 Java failure cause (30% of all Java failures). |
| Class name | `java_program` | Class MUST be named `Answer` |
| Method-only solution | `java_method` | Solution MUST be ONLY the method(s) — NO class wrapper, NO main(), NO import statements. The system wraps your solution in `public class Answer { <your methods> ... main() { testcode } }`. Including `public class` or `main()` in the solution causes "illegal start of expression" errors. |
| Reflection in tests | `java_class` | Wrap test code in `try { ... } catch (Exception e) { System.out.println("Error: " + e.getMessage()); }` |

### Python

| Rule | Applies To | What To Do |
|------|-----------|------------|
| EOF guard | `python3_w_input` | Use `import sys; data = sys.stdin.read().strip()` with `if data:` guard. Never bare `input()`. |
| Function-only solution | `python3` | Solution is function/class definitions only. Test code calls them. |

### C

| Rule | Applies To | What To Do |
|------|-----------|------------|
| `-Werror` | ALL C types | Jobe compiles with `-Werror`. ALL warnings are fatal. No unused variables, no implicit declarations. |
| `#include` in answer | `c_function` | ALL headers (`<stdio.h>`, `<ctype.h>`, `<string.h>`, `<math.h>`) MUST be in `<answer>`, not just test code. |
| `scanf` return check | `c_program` | Always: `if (scanf("%d", &n) == 1) { ... }`. Unchecked scanf on empty stdin = uninitialized variable. |
| Test code wrapper | `c_function` | Test code MUST include `#include <stdio.h>` and `int main(void) { ... return 0; }` |
| Array printing | ALL C types | Use `if (i) printf(" "); printf("%d", arr[i]);` — NOT `printf("%d ", arr[i])` (trailing space fails). |
| Function-only solution | `c_function` | Solution is function(s) with #include headers only. NO main(). Test code provides main(). |

### C++

| Rule | Applies To | What To Do |
|------|-----------|------------|
| `-Werror` | ALL C++ types | Same as C — all warnings fatal. |
| `size_t` for `.size()` | ALL C++ types | `for (size_t i = 0; i < vec.size(); i++)` — `int` vs `size_t` is `-Werror=sign-compare` fatal. |
| Member init order | `cpp_function` | Declare class members in SAME order as constructor initializer list. `-Werror=reorder` is fatal. |
| `#include` in answer | `cpp_function` | Include `<vector>`, `<string>`, `<algorithm>`, `<sstream>` etc. in `<answer>`, not just test code. |
| Test code wrapper | `cpp_function` | Test code MUST include `#include <iostream>`, `using namespace std;`, and `int main() { ... }` |
| Array printing | ALL C++ types | `if (i) cout << " "; cout << v[i];` — no trailing space. |

### Node.js

| Rule | Applies To | What To Do |
|------|-----------|------------|
| Deterministic output | `nodejs` | Use `JSON.stringify(result)` for objects/arrays. Do NOT use `JSON.stringify(obj, replacer)` — replacer drops nested keys. |
| No npm packages | `nodejs` | Only Node.js built-in features. No require() of external modules. |

---

## 3. Output Matching Rules

These apply to ALL languages. 35% of all failures are output mismatches.

1. Expected output MUST match actual output character-for-character
2. Do NOT add trailing newline in `<expected>` — CodeRunner strips trailing newlines before comparing
3. Leading/trailing spaces cause failures — be precise
4. NEVER depend on exact floating-point results — use integer math or `String.format("%.2f", val)` / `round(val, 2)` / `printf("%.2f", val)`
5. Mentally trace the model solution through EVERY test case — compute expected output, do NOT guess
6. Jobe stdin requires trailing `\n` — the validation script adds this automatically

---

## 4. Test Case Design

### Structure

| Attribute | Values | Purpose |
|-----------|--------|---------|
| `testtype` | `0`=normal, `1`=precheck-only, `2`=both | When test runs |
| `useasexample` | `1`=visible, `0`=hidden | Student visibility |
| `hiderestiffail` | `0` or `1` | Stop showing tests after failure |
| `mark` | Decimal (e.g. `0.2500000`) | Weight for partial credit |
| `display` | `SHOW`, `HIDE`, `HIDE_IF_FAIL`, `HIDE_IF_SUCCEED` | Result visibility |

### Requirements

- Minimum 3 test cases: 1 visible (`useasexample="1"`, `SHOW`), 2+ hidden (`useasexample="0"`, `HIDE`)
- Each test MUST produce DIFFERENT output (prevents hard-coding)
- Include edge case test (empty input, zero, negative, boundary values)
- For stdin types: include empty stdin test case
- For partial credit (`allornothing="0"`): marks MUST sum to 1.0

### Test Code Patterns by Type

**Function types** (java_class, java_method, python3, c_function, cpp_function, nodejs):
```
<testcode> calls student code </testcode>
<stdin> empty </stdin>
```

**Program types** (java_program, c_program, cpp_program, python3_w_input):
```
<testcode> empty </testcode>
<stdin> input data </stdin>
```

### Grading Modes

| Mode | XML | When To Use |
|------|-----|-------------|
| All-or-nothing | `<allornothing>1</allornothing>` | Methods depend on each other. Default. |
| Partial credit | `<allornothing>0</allornothing>` | Independent tests. Set `mark` per test, sum to 1.0. |

---

## 5. Question Design

Every question MUST include:

1. **`<name>`** — `Micro-Assessment - Descriptive Title` (unique, not "Question 1")
2. **`<questiontext>`** — Clear instructions: method names, parameter types, return types, example usage
3. **Penalty regime text** in questiontext:
   ```html
   <p><strong>Penalty Regime:</strong></p>
   <ul>
   <li>You have a total of <strong>3 attempts</strong> without any penalties.</li>
   <li>Starting from the <strong>4th attempt onwards</strong>, a penalty of <strong>10%</strong> will be applied.</li>
   </ul>
   ```
4. **`<answerpreload>`** — Compilable skeleton with comments (don't give away solution structure)
5. **`<answer>`** — Complete, correct model solution (NEVER empty)
6. **`<generalfeedback>`** — Explanation of correct approach (shown after quiz closes)
7. **Self-contained** — never reference lectures or slides

---

## 6. JSON-First Planning (Recommended)

Before writing XML, plan each question as a mental JSON structure. This ensures all parts are correct before committing to XML format.

### Plan Each Question As:

```json
{
  "name": "Descriptive Title",
  "qtype": "java_class",
  "question_text": "HTML instructions with examples",
  "feedback": "Explanation of correct approach",
  "solution": "complete compilable model solution",
  "preload": "skeleton code for students",
  "test_cases": [
    {"testcode": "test code", "stdin": "", "expected": "exact output", "visible": true},
    {"testcode": "test code", "stdin": "", "expected": "exact output", "visible": false}
  ]
}
```

### Pre-Writing Checklist

Before writing the XML, verify for each question:
- `solution`: For java_method — ONLY the method(s), no class wrapper, no main()
- `solution`: For function types — ONLY the function(s), no main()
- `solution`: For program types — complete program with main()
- `testcode`: For c_function/cpp_function — MUST include #include and int main() wrapper
- `testcode`: Empty for program types (use stdin instead)
- `expected`: Mentally trace solution through EVERY test case
- First test case visible, rest hidden

---

## 7. XML Template

Use this only when generating XML directly (prefer JSON method above).

```xml
<?xml version="1.0" encoding="UTF-8"?>
<quiz>
  <question type="category">
    <category><text>$course$/Category Name</text></category>
  </question>

  <question type="coderunner">
    <name><text>Micro-Assessment - Title</text></name>
    <questiontext format="html"><text><![CDATA[QUESTION_HTML]]></text></questiontext>
    <generalfeedback format="html"><text><![CDATA[<p>FEEDBACK</p>]]></text></generalfeedback>
    <defaultgrade>1</defaultgrade>
    <penalty>0</penalty>
    <hidden>0</hidden>
    <idnumber></idnumber>
    <coderunnertype>TYPE</coderunnertype>
    <prototypetype>0</prototypetype>
    <allornothing>1</allornothing>
    <penaltyregime>0, 0, 0, 10, 20, ...</penaltyregime>
    <precheck>0</precheck>
    <hidecheck>0</hidecheck>
    <showsource>0</showsource>
    <answerboxlines>18</answerboxlines>
    <answerboxcolumns>100</answerboxcolumns>
    <answerpreload><![CDATA[SKELETON]]></answerpreload>
    <globalextra></globalextra>
    <useace></useace>
    <resultcolumns></resultcolumns>
    <template></template>
    <iscombinatortemplate></iscombinatortemplate>
    <allowmultiplestdins></allowmultiplestdins>
    <answer><![CDATA[MODEL_SOLUTION]]></answer>
    <validateonsave>1</validateonsave>
    <testsplitterre></testsplitterre>
    <language></language>
    <acelang></acelang>
    <sandbox></sandbox>
    <grader></grader>
    <cputimelimitsecs></cputimelimitsecs>
    <memlimitmb></memlimitmb>
    <sandboxparams></sandboxparams>
    <templateparams></templateparams>
    <hoisttemplateparams>1</hoisttemplateparams>
    <extractcodefromjson>1</extractcodefromjson>
    <templateparamslang>None</templateparamslang>
    <templateparamsevalpertry>0</templateparamsevalpertry>
    <templateparamsevald>{}</templateparamsevald>
    <twigall>0</twigall>
    <uiplugin></uiplugin>
    <uiparameters><![CDATA[{"live_autocompletion": true}]]></uiparameters>
    <attachments>0</attachments>
    <attachmentsrequired>0</attachmentsrequired>
    <maxfilesize>10240</maxfilesize>
    <filenamesregex></filenamesregex>
    <filenamesexplain></filenamesexplain>
    <displayfeedback>1</displayfeedback>
    <giveupallowed>0</giveupallowed>
    <prototypeextra></prototypeextra>
    <testcases>
      <testcase testtype="0" useasexample="1" hiderestiffail="0" mark="1.0000000">
        <testcode><text>TEST_CODE</text></testcode>
        <stdin><text></text></stdin>
        <expected><text>EXPECTED</text></expected>
        <extra><text></text></extra>
        <display><text>SHOW</text></display>
      </testcase>
    </testcases>
  </question>
</quiz>
```

### Optional Feature Tags

| Feature | Tag | Values | Effect |
|---------|-----|--------|--------|
| Precheck | `<precheck>1</precheck>` | 0/1 | Students can test before submitting |
| Give up | `<giveupallowed>1</giveupallowed>` | 0/1 | Students can reveal model answer |
| CPU limit | `<cputimelimitsecs>5</cputimelimitsecs>` | seconds | Enforce time limit |
| Memory limit | `<memlimitmb>64</memlimitmb>` | MB | Enforce memory limit |

---

## 8. Validation

**Mandatory.** Run after generating XML:

```bash
python3 ~/.claude/skills/moodle-coderunner/validate_coderunner.py <xml_file> --jobe-url http://3.250.73.210:4000
```

### Jobe Servers

| Server | URL | Notes |
|--------|-----|-------|
| AWS Jobe 1 | `http://3.250.73.210:4000` | Primary |
| AWS Jobe 2 | `http://3.250.73.210:4001` | Backup |
| AWS Jobe 3 | `http://3.250.73.210:4002` | Backup |
| AWS Jobe 4 | `http://3.250.73.210:4003` | Backup |
| AWS Jobe 5 | `http://3.250.73.210:4004` | Backup |
| AWS Jobe 6 | `http://3.250.73.210:4005` | Backup |

Languages: C 13.3, C++ 13.3, Java 21.0.5, Python 3.12.3, Node 18.19

### Fix Loop

If any question fails:
1. Read the error (COMPILE_ERROR, WRONG_OUTPUT, RUNTIME_ERROR)
2. Fix the `<answer>` or `<expected>` in the XML
3. Re-validate
4. Repeat until all pass (max 3 cycles)

### Common Fixes

| Error | Fix |
|-------|-----|
| COMPILE_ERROR: implicit declaration | Add missing `#include` to `<answer>` |
| COMPILE_ERROR: sign-compare | Change `int i` to `size_t i` in `.size()` loops |
| COMPILE_ERROR: reorder | Reorder class members to match constructor |
| COMPILE_ERROR: illegal start of expression (java_method) | Remove class wrapper / access modifiers from solution — must be raw method(s) only |
| WRONG_OUTPUT: trailing space | Use `if(i) printf(" ")` pattern for arrays |
| WRONG_OUTPUT: expected mismatch | Re-trace solution, fix `<expected>` |
| RUNTIME_ERROR: NoSuchElementException | Add `hasNext()` guard before Scanner read |
| RUNTIME_ERROR: EOFError | Use `sys.stdin.read()` instead of `input()` |

---

## 9. Workflow

1. **Ask**: language, topic, count, difficulty, grading mode
2. **Design**: Generate as JSON (Section 6) — instructions, preload, solution, test cases, feedback
3. **Convert**: Wrap JSON in XML boilerplate (Section 7)
4. **Validate**: Run validation script against Jobe — fix any failures
5. **Deliver**: XML file + confirmation all questions passed

---

## 10. Final Checklist

Before delivery, every question must satisfy:

- [ ] `<name>` tag uses full word, descriptive title
- [ ] Correct `<coderunnertype>` for the task
- [ ] `<answer>` contains complete, compilable model solution
- [ ] `<answer>` for java_method is raw method(s) only (no class wrapper)
- [ ] `<answerpreload>` provides compilable skeleton
- [ ] `<generalfeedback>` explains correct approach
- [ ] `<validateonsave>1</validateonsave>`
- [ ] `<penaltyregime>0, 0, 0, 10, 20, ...</penaltyregime>`
- [ ] Penalty regime text in `<questiontext>`
- [ ] Self-contained (no lecture references)
- [ ] 3+ test cases: 1 visible, 2+ hidden, edge cases included
- [ ] Each test produces different output
- [ ] Expected output traced character-by-character
- [ ] No floating-point exact comparisons
- [ ] Marks sum to 1.0 (if partial credit)
- [ ] Language-specific rules from Section 2 followed
- [ ] **ALL questions passed Jobe validation**

---

## 11. File Naming

- Questions: `{module}_{topic}_coderunner.xml`
- Review: `{module}_{topic}_coderunner_review.md`

## Moodle Import

1. Course > Question bank > Import
2. Format: Moodle XML
3. Upload `.xml` file
4. Import and review
