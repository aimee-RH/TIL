
# CS186 WAL Recovery (ARIES) Overview

This guide covers the core concepts of the ARIES recovery algorithm as taught in Berkeley CS186.

---

## 1. Core Premise: Buffer Management Policy
ARIES is designed to handle the most flexible (but complex) buffer management policy combination:

* **Steal:** The buffer manager **allows** replacing (writing back) dirty pages to the disk *before* the transaction commits.
    * *Implication:* The disk may contain dirty data from uncommitted transactions if a crash occurs (requires **Undo**).
* **No-Force:** The buffer manager does **not** force dirty pages to be written to disk when a transaction commits (it only requires the log to be flushed).
    * *Implication:* The disk may be missing data from committed transactions upon a crash (requires **Redo**).

---

## 2. The Two WAL (Write-Ahead Logging) Rules
Before any data page in memory is written to the disk, the log must be handled first:

1.  **Log Record First:** The log record describing the change must be flushed to the disk **before** the data page itself is written to the disk.
2.  **Commit Rule:** A transaction is considered committed only after the `Commit` log record is successfully flushed to the disk.

---

## 3. Key Data Structures
During recovery, the system maintains two critical tables in memory (reconstructed during the Analysis phase):

### A. Transaction Table (TT)
Tracks the state of currently active transactions.
* **Fields:** `TransID`, `Status` (Running/Committing/Aborting), `LastLSN` (The LSN of the most recent log record for this transaction).

### B. Dirty Page Table (DPT)
Tracks all pages that have been modified in memory but not yet flushed to the disk.
* **Fields:** `PageID`, `recLSN` (Recovery LSN).
* **Crucial Concept `recLSN`:** The LSN of the log record that **first** caused the page to become dirty. This determines where the Redo phase starts for this page.

---

## 4. The 3 Phases of ARIES
The recovery process always follows this order: **Analysis $\rightarrow$ Redo $\rightarrow$ Undo**.

### Phase 1: Analysis
* **Goal:** Reconstruct the memory state (TT and DPT) at the moment of the crash and identify "Loser" (uncommitted) transactions versus "Winner" transactions.
* **Process:**
    1.  Find the most recent **Checkpoint** via the Master Record.
    2.  Initialize TT and DPT from the Checkpoint data.
    3.  Scan the log forward from the Checkpoint to the end:
        * **End Record:** Remove the transaction from the TT.
        * **Update Record:** If the page is not in the DPT, add it and set `recLSN = current LSN`. Update the `LastLSN` in the TT.
* **Result:** The TT now contains the "Losers" (transactions that were active at the time of the crash).

### Phase 2: Redo ("Repeating History")
* **Goal:** Restore the database to the **exact state** it was in at the moment of the crash (including uncommitted changes).
* **Start Point:** The smallest `recLSN` found in the DPT.
* **Process:** Scan forward from the smallest `recLSN`. For every update log record:
    * Check if the action needs to be re-applied.
* **Conditions to SKIP Redo (if ANY are true):**
    1.  **Page is not in DPT:** The page was flushed to disk before the crash (it is clean).
    2.  **Log LSN < Page recLSN:** The log record occurred before the page became dirty (implies a previous flush occurred).
    3.  **PageLSN (on disk) $\ge$ Log LSN:** You must read the page from disk to check this. If the page on disk is "newer" than the log record, the change is already there.
* **Action:** If Redo is needed, apply the change and update the `pageLSN` in memory to the current Log LSN.

### Phase 3: Undo
* **Goal:** Roll back the changes of all "Loser" transactions (those still in the TT after Analysis).
* **Process:** Scan the log **backward** using the `prevLSN` pointers.
* **CLR (Compensation Log Record):**
    * For every update you undo, you write a CLR to the log.
    * CLRs are **Redo-only** (they are never undone).
    * **`UndoNextLSN`:** CLRs store this pointer, which points to the *next* log record that needs to be undone for that transaction (skipping the record that was just compensated). This prevents infinite loops during repeated crashes.
* **End:** Once a transaction is fully undone, write an `End` record.

---

## 5. Cheat Sheet: Key Concepts

| Term | Definition |
| :--- | :--- |
| **LSN (Log Sequence Number)** | A unique, monotonically increasing ID for every log record. |
| **prevLSN** | Pointer to the previous log record *for the same transaction* (creates a linked list for Undo). |
| **pageLSN** | Stored on the data page itself; indicates the LSN of the last update to modify this page. |
| **flushedLSN** | The highest LSN currently written to stable storage (disk). |
| **Fuzzy Checkpoint** | A checkpoint that saves the state of TT and DPT without halting all transactions. |
