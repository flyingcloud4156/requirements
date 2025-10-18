# Checkstyle Auto‑Fix Setup (Final Working Config)

This guide explains how to **auto‑format Java code** (indentation, imports, line‑wraps) before running **Checkstyle**, so your build passes consistently.

---

## 1) Add Spotless (auto‑formatter) **before** Checkstyle

Insert this plugin **inside** `<build><plugins>` and **above** your Checkstyle plugin.

```xml
<plugin>
  <groupId>com.diffplug.spotless</groupId>
  <artifactId>spotless-maven-plugin</artifactId>
  <version>2.43.0</version>
  <configuration>
    <java>
      <googleJavaFormat version="1.17.0"/>
      <removeUnusedImports/>
      <includes>
        <include>src/main/java/**/*.java</include>
        <include>src/test/java/**/*.java</include>
      </includes>
    </java>
  </configuration>
  <executions>
    <execution>
      <id>spotless-apply</id>
      <phase>process-sources</phase>
      <goals>
        <goal>apply</goal>
      </goals>
    </execution>
    <execution>
      <id>spotless-check</id>
      <phase>verify</phase>
      <goals>
        <goal>check</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

**Why:** `google-java-format` enforces consistent indentation/spacing and import order across the codebase. Running it in `process-sources` guarantees formatting happens **before** style checks.

---

## 2) Run Checkstyle in the `verify` phase

Change your existing Checkstyle execution to run during `verify` (not `validate`) so it executes **after** Spotless.

**Replace** the execution block like this:

```xml
<execution>
  <id>checkstyle-verify</id>
  <phase>verify</phase>
  <goals>
    <goal>check</goal>
  </goals>
</execution>
```

---

## 3) Commands

Local auto‑fix and verify:
```bash
mvn spotless:apply -DskipTests
mvn verify -DskipTests
```

- `spotless:apply` will automatically fix indentation and other formatting issues that previously triggered errors like:
  - `Indentation level 4, expected 2`
- `verify` will then run Checkstyle and other quality gates.

---

## 4) Notes on Javadoc rules

`MissingJavadocType` cannot be auto‑fixed by formatters. You have two choices:

- **Preferred:** Add a minimal Javadoc to the class.
  ```java
  /**
   * Service for category operations.
   */
  public class CategoryService {
  }
  ```

- **Temporary:** Relax or suppress the rule in your Checkstyle config if your project policy allows it.

---

## 5) Expected outcome

- Formatting violations (indentation, spacing, imports) are **auto‑corrected** by `spotless:apply`.
- Checkstyle runs later and should **pass** for formatting, leaving only non‑format concerns (e.g., missing Javadoc) to address.
- CI pipelines become stable and reproducible.

---

## 6) Troubleshooting

- If the build fails with a Spotless configuration error, ensure you **do not** use `<target>` under `<java>`. Use `<includes>`/`<excludes>` as shown.
- If Checkstyle still runs in `validate`, confirm you changed the execution phase to `verify`.
- If you need to re‑format only, you can run:
  ```bash
  mvn spotless:apply -DskipTests
  ```

---

## 7) Optional: CI tip

Add a CI step that runs:
```bash
mvn -B spotless:apply verify -DskipTests
```
so formatting is applied automatically in the pipeline before style checks.

---

Happy building!
