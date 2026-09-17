# Technical Debt

## TD-001 — Combined README authorities

- **Status:** Accepted
- **Severity:** Low
- **Area:** Documentation architecture
- **Introduced/Discovered:** 2026-09-16
- **Standing waiver:** No

**Related files:** `README.md`, `PROJECT-STANDARD.md`

**Description:** The existing README is the single authority for project overview, planned architecture, and operations. This preserves the established blueprint without copying its content into competing documents, but it makes routed reading less granular.

**Why it exists:** The repository was created as one implementation guide before Standard 2.2 adoption.

**Impact:** Architecture or operations work requires reading a long file. A future split must move content rather than duplicate it.

**Recommended resolution:** Split the README only when implementation makes the boundaries stable. Keep a concise overview in README and move architecture and operating procedures into separate mapped authorities in one change.

**Fix trigger:** The first substantial implementation milestone or repeated maintenance errors caused by the combined document.

**Estimated effort:** Medium
