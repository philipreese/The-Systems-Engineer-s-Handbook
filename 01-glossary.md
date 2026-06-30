# Glossary

Authoritative definitions for terms used throughout this handbook. A term gets an entry here when it is used across more than one chapter or when its precise meaning matters to the argument being made.

Terms are added as chapters are completed. If a term is used in a chapter but not yet defined here, that is a gap to fill.

---

**accidental complexity**: Complexity introduced by the engineering solution rather than the problem domain itself. Can be reduced by changing the solution. Contrasted with essential complexity. First introduced in: [Part I, Ch 02](part1-systems-thinking/ch02-complexity-is-the-enemy.md).

**cyclomatic complexity**: A quantitative measure of the number of independent execution paths through a program, derived from the control flow graph. Higher values indicate harder-to-test and harder-to-reason-about code. First introduced in: [Part I, Ch 02](part1-systems-thinking/ch02-complexity-is-the-enemy.md).

**cost of change**: How expensive it is to modify a system over time, measured in engineering time, risk, and coordination overhead. Distinct from cost of execution. First introduced in: [Part I, Ch 01](part1-systems-thinking/ch01-what-engineering-optimizes.md).

**essential complexity**: Complexity inherent to the problem domain that cannot be eliminated without changing what the system does. Contrasted with accidental complexity. First introduced in: [Part I, Ch 02](part1-systems-thinking/ch02-complexity-is-the-enemy.md).

**state space explosion**: The condition where mutable state variables combine to produce a number of possible system configurations that exceeds what engineers can anticipate or test. A primary failure mode of unconstrained mutable state. First introduced in: [Part I, Ch 02](part1-systems-thinking/ch02-complexity-is-the-enemy.md).

**MTBF (Mean Time Between Failures)**: A reliability paradigm that optimizes for preventing failures from occurring. Contrasted with MTTR. First introduced in: [Part I, Ch 01](part1-systems-thinking/ch01-what-engineering-optimizes.md).

**MTTR (Mean Time To Recovery)**: A reliability paradigm that accepts failures as inevitable and optimizes for recovering from them quickly. Contrasted with MTBF. First introduced in: [Part I, Ch 01](part1-systems-thinking/ch01-what-engineering-optimizes.md).

**optimization target**: An objective a system is designed to optimize — latency, throughput, reliability, cost of change, etc. Targets may be explicit (documented) or implicit (inferred). First introduced in: [Part I, Ch 01](part1-systems-thinking/ch01-what-engineering-optimizes.md).

**optimization target drift**: The phenomenon where a system's actual optimization targets diverge from its intended ones over time due to accumulated changes, hotfixes, and operational adjustments. First introduced in: [Part I, Ch 01](part1-systems-thinking/ch01-what-engineering-optimizes.md).

---

## Format

Each entry follows:

```
**Term**: Definition in one to three sentences.
First introduced in: [Part X, Chapter Y](link).
```
