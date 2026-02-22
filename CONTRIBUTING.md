# Contributing to checkstyle-openrewrite-recipes

Thank you for your interest in contributing! This guide will help you understand the codebase and get started with your first contribution.

## Table of Contents

- [Project Overview](#project-overview)
- [Project Structure](#project-structure)
- [How Recipes Work](#how-recipes-work)
- [How to Add a New Recipe](#how-to-add-a-new-recipe)
- [Good First Issues](#good-first-issues)
- [Building and Testing](#building-and-testing)

---

## Project Overview

This project provides [OpenRewrite](https://docs.openrewrite.org/) recipes that automatically fix [Checkstyle](https://checkstyle.sourceforge.io/) violations in Java source code.

The workflow is:
1. Run `mvn checkstyle:check` — Checkstyle scans your code and writes a violation report (`target/checkstyle/checkstyle-report.xml` or `.sarif`).
2. Run `mvn rewrite:run` — Our recipes read that report and rewrite the violating source files to fix the issues.

---

## Project Structure

```
src/
├── main/java/org/checkstyle/autofix/
│   ├── CheckstyleAutoFix.java          # Main recipe: reads config + report, delegates to sub-recipes
│   ├── CheckstyleRecipeRegistry.java   # Maps Checkstyle check names → Recipe factory functions
│   ├── CheckFullName.java              # Enum of all supported check class names (fully-qualified)
│   ├── CheckstyleCheck.java            # Record holding a check name + id
│   ├── PositionHelper.java             # Utility: maps AST nodes to line/column positions
│   ├── parser/
│   │   ├── CheckstyleViolation.java    # Model: one Checkstyle violation (file, line, col, msg)
│   │   ├── CheckConfiguration.java     # Model: Checkstyle check configuration (properties)
│   │   ├── ConfigurationLoader.java    # Loads the Checkstyle XML config file
│   │   ├── XmlReportParser.java        # Parses Checkstyle XML violation reports
│   │   └── SarifReportParser.java      # Parses SARIF violation reports
│   └── recipe/
│       ├── FinalLocalVariable.java     # Recipe: adds 'final' to non-reassigned local variables
│       ├── Header.java                 # Recipe: fixes or adds file header comments
│       ├── HexLiteralCase.java         # Recipe: uppercases hex literal digits (0xabcd → 0xABCD)
│       ├── NewlineAtEndOfFile.java     # Recipe: ensures files end with a newline
│       ├── RedundantImport.java        # Recipe: removes duplicate/redundant imports
│       └── UpperEll.java              # Recipe: replaces lowercase 'l' suffix in long literals (10l → 10L)
└── test/
    ├── java/org/checkstyle/autofix/
    │   ├── recipe/
    │   │   ├── AbstractRecipeTestSupport.java  # Base class for all recipe tests
    │   │   ├── FinalLocalVariableTest.java
    │   │   ├── HexLiteralCaseTest.java
    │   │   └── ...
    │   ├── InputClassRenamer.java      # Test helper: renames Input→Output class names
    │   └── RemoveViolationComments.java # Test helper: strips violation marker comments
    └── resources/org/checkstyle/autofix/recipe/
        └── <recipename>/               # One folder per test case
            ├── Input<TestCase>.java    # Java file with violations (and inline Checkstyle config)
            └── Output<TestCase>.java   # Expected result after the recipe runs
```

---

## How Recipes Work

Every recipe follows this three-step pattern:

### 1. Parse violations
`CheckstyleAutoFix.getRecipeList()` calls `XmlReportParser` (or `SarifReportParser`) to load all `CheckstyleViolation` objects from the report. Each violation carries:
- the source file path
- line number and column number
- the Checkstyle check class name (e.g. `com.puppycrawl.tools.checkstyle.checks.UpperEllCheck`)
- the human-readable message

### 2. Map to a Recipe
`CheckstyleRecipeRegistry.getRecipes()` groups violations by check name, looks up the matching recipe factory in its enum map, and instantiates the recipe with the relevant violations.

### 3. Visit and fix the AST
Each recipe extends `org.openrewrite.Recipe` and returns a `TreeVisitor` that walks the Java AST. When the visitor reaches a node whose line/column matches a violation, it rewrites that node. The comparison is done through `PositionHelper.computeLinePosition()` and `computeColumnPosition()`.

---

## How to Add a New Recipe

Follow these five steps. Use `UpperEll` as a minimal reference implementation.

### Step 1 — Create the recipe class

Create `src/main/java/org/checkstyle/autofix/recipe/<CheckName>.java`:

```java
package org.checkstyle.autofix.recipe;

import java.util.List;
import org.checkstyle.autofix.parser.CheckstyleViolation;
import org.openrewrite.ExecutionContext;
import org.openrewrite.Recipe;
import org.openrewrite.TreeVisitor;
import org.openrewrite.java.JavaIsoVisitor;
import org.openrewrite.java.tree.J;

public class MyCheck extends Recipe {

    private final List<CheckstyleViolation> violations;

    public MyCheck(List<CheckstyleViolation> violations) {
        this.violations = violations;
    }

    @Override
    public String getDisplayName() { return "MyCheck recipe"; }

    @Override
    public String getDescription() { return "Fixes MyCheck violations."; }

    @Override
    public TreeVisitor<?, ExecutionContext> getVisitor() {
        return new JavaIsoVisitor<>() {
            // override the visit method for the relevant AST node type,
            // check isAtViolationLocation(), then return a rewritten node.
        };
    }
}
```

### Step 2 — Register the check's fully-qualified class name

Add an entry to the `CheckFullName` enum in `CheckFullName.java`:

```java
MY_CHECK("com.puppycrawl.tools.checkstyle.checks.<package>.MyCheckCheck"),
```

You can find the exact class name on the [Checkstyle documentation page](https://checkstyle.sourceforge.io/) for the check.

### Step 3 — Register the recipe factory

Add one line to the `static` block in `CheckstyleRecipeRegistry.java`:

```java
RECIPE_MAP.put(CheckFullName.MY_CHECK, MyCheck::new);
```

Use `RECIPE_MAP_WITH_CONFIG` instead if your recipe needs the Checkstyle configuration (e.g. a `header` file path). See `Header` for an example.

### Step 4 — Add test resources

Create a folder `src/test/resources/org/checkstyle/autofix/recipe/<checkname>/<testcasename>/` containing:

- `Input<TestCaseName>.java` — a Java file with one or more violations. Include an inline Checkstyle configuration comment at the top of the file. See any existing `Input*.java` file for the format.
- `Output<TestCaseName>.java` — the same file after the recipe has been applied.

### Step 5 — Write the test class

Create `src/test/java/org/checkstyle/autofix/recipe/<CheckName>Test.java`:

```java
package org.checkstyle.autofix.recipe;

import org.checkstyle.autofix.parser.ReportParser;

public class MyCheckTest extends AbstractRecipeTestSupport {

    @Override
    protected String getSubpackage() { return "mycheckfolder"; }

    @RecipeTest
    void myTestCase(ReportParser parser) throws Exception {
        verify(parser, "MyTestCase");
    }
}
```

Each `@RecipeTest` method runs the test against both the XML and SARIF report parsers automatically.

---

## Good First Issues

The table in `README.md` lists every Checkstyle check and its current status. Items marked **🟢 TBD** have no recipe yet but are straightforward to implement. The following are especially good starting points, roughly ordered from simplest to more involved:

| Check | What it fixes | Checkstyle docs |
|-------|---------------|-----------------|
| `FileTabCharacter` | Replace tab characters (`\t`) with spaces in source files | [link](https://checkstyle.sourceforge.io/checks/whitespace/filetabcharacter.html) |
| `EmptyStatement` | Remove accidental extra semicolons (`;` alone on a line) | [link](https://checkstyle.sourceforge.io/checks/coding/emptystatement.html) |
| `AvoidNoArgumentSuperConstructorCall` | Remove redundant `super()` calls from constructors | [link](https://checkstyle.sourceforge.io/checks/coding/avoidnoargumentsuperconstructorcall.html) |
| `ArrayTypeStyle` | Convert C-style array declarations (`String args[]`) to Java-style (`String[] args`) | [link](https://checkstyle.sourceforge.io/checks/misc/arraytypestyle.html) |
| `RedundantModifier` | Remove modifiers that are implied by context (e.g. `public` on interface methods) | [link](https://checkstyle.sourceforge.io/checks/modifier/redundantmodifier.html) |
| `ModifierOrder` | Reorder modifiers to the order mandated by the Java Language Specification | [link](https://checkstyle.sourceforge.io/checks/modifier/modifierorder.html) |
| `UnusedImports` | Remove import statements that are never used in the file | [link](https://checkstyle.sourceforge.io/checks/imports/unusedimports.html) |
| `FinalParameters` | Add `final` to method/constructor parameters that are never reassigned | [link](https://checkstyle.sourceforge.io/checks/misc/finalparameters.html) |
| `NoEnumTrailingComma` | Remove trailing comma after the last enum constant | [link](https://checkstyle.sourceforge.io/checks/coding/noenumtrailingcomma.html) |
| `EqualsAvoidNull` | Flip `variable.equals("literal")` to `"literal".equals(variable)` | [link](https://checkstyle.sourceforge.io/checks/coding/equalsavoidnull.html) |

### Tips for picking your first issue

- **Start with `FileTabCharacter` or `EmptyStatement`** — both are pure text / token-level changes with no complex AST navigation required.
- Look at `UpperEll.java` for a minimal example of a literal-level fix that uses line/column matching.
- Look at `FinalLocalVariable.java` for a more advanced example that splits multi-variable declarations and adds modifiers.
- All recipes use `PositionHelper` to translate AST cursor positions to line/column numbers — you will need this whenever you match violations by location.

---

## Building and Testing

```bash
# Compile and run all tests
./mvnw test

# Run Checkstyle on the project itself (the project checks its own code)
./mvnw checkstyle:check

# Run a specific test class
./mvnw test -Dtest=UpperEllTest

# Run mutation tests (slow, only needed before a final PR)
./mvnw test -P pitest
```

The project requires **Java 21** and **Maven 3.x** (the `mvnw` wrapper downloads Maven automatically).
