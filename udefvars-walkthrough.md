**Code Maintenance Report**

### 1\. Procedure & Data Analysis

Analysis of all procedures, their local data, and shared data structures.

**User-Defined Data (`TYPE`)**

*Note: The following `TYPE` definitions are declared locally within `PROCEDURE udefVars` and are used to structure its `DIM` variables. `vSort` and `fSort` redeclare these types to correctly interpret their `PARAM` and `DIM` variables.*

* `TYPE VREC`: A record for the in-memory variable lookup table.
    * `vTyp,nTyp:BYTE`
    * `nVar:BOOLEAN`
    * `vOff,vdOff,dsOff,dmOff,fOff,pOff:INTEGER`
* `TYPE VFP1`: First part of a complex file record, holds pointers and sizes.
    * `vdTkn:BYTE`
    * `vdPtr,vdSiz,dsPtr,dsvSiz,dsSiz,dmSiz,fSiz,pSiz,elem1,elem2,elem3,vSRLen:INTEGER`
* `TYPE VFP2`: Second part of a complex file record, holds strings and metadata.
    * `vArray:STRING[19]`
    * `vName:STRING[29]`
    * `vStrLen:STRING[7]`
    * `vRecNum,vRefCnt,intReal,pVar:INTEGER`
* `TYPE VFP3`: Third part of a complex file record, holds type flags.
    * `mTyp,vLnk,fNum:BYTE`
    * `pRec,pFld:BOOLEAN`
* `TYPE VFRC`: The complete variable file record, combining the three parts.
    * `vp1:VFP1`
    * `vp2:VFP2`
    * `vp3:VFP3`

**Procedure Interface & Storage**

* **`PROCEDURE udefVars`**
    * **Parameters (`PARAM`):**
        * `path:BYTE`: Input file path number for the main module.
        * `er,tpVars:INTEGER`: Error code (by ref) and variable count.
        * `pOpen,verbose:BOOLEAN`: Flags for file status and logging level.
        * `modSize,execOff,descOff,symTabOff:REAL`: Offsets into the file.
        * `dataDir:STRING[16]`: Path for data files.
    * **Local Variables (`DIM`):**
        * `varRec(400):VREC`: In-memory array for variable lookup.
        * `varFRec:VFRC`: Working record for file I/O.
        * `tokenCode,tokenByte,byteVal:BYTE`: Byte-level I/O variables.
        * `iToken,vToken:BYTE`: Token counters.
        * `recNum,mCount...recArraySize:INTEGER`: Numerous counters and pointers for offsets and sizes.
        * `mSize:REAL`: Real-type size variable.
        * `found,isSimple...first:BOOLEAN`: Numerous flags for parsing logic.
        * `varRefs,varCount,varNum,fldNum:INTEGER`: Reference counters.
        * `vPath,dPath,oPath:BYTE`: File path numbers for varDefs, procData, and output.
        * `vOpen,dOpen,oOpen:BOOLEAN`: File open status flags.
        * `which:INTEGER`: Error data.
        * `vFile,dFile,oFile:STRING[29]`: File names.
        * `numStr:STRING[4]`, `strLen:STRING[7]`, `tokenString:STRING[8]`, `arrayStr:STRING[19]`, `subName,tName:STRING[29]`: String manipulation variables.
        * *Note:* `chainMod:STRING[177]` is commented out via `!`, which is an editor shortcut for `REM`.

* **`PROCEDURE vSort`**
    * **Parameters (`PARAM`):**
        * `er,which:INTEGER`: Error code (by ref) and recursion counter.
        * `vPath:BYTE`: File path number for the variable definitions file.
        * `bottom,top:INTEGER`: Array bounds for the quicksort.
        * `varRecs(400):VREC`: The in-memory variable lookup array (passed by reference).
    * **Local Variables (`DIM`):**
        * `varRec:VREC`: Temporary record for swapping `varRecs` array elements.
        * `varFRec:VFRC`: Temporary record for file I/O.
        * `varFRecs(2):VFRC`: Array to hold two file records for swapping.
        * `lower,upper:INTEGER`: Loop counters for quicksort partitioning.
        * `btemp:BOOLEAN`: Flag for loop condition.

* **`PROCEDURE fSort`**
    * **Parameters (`PARAM`):**
        * `vPath:BYTE`: File path number for the variable definitions file.
        * `recCount,varCount:INTEGER`: Loop counters.
        * `varRecs(400):VREC`: The in-memory variable lookup array (passed by reference).
    * **Local Variables (`DIM`):**
        * `varRec:VREC`: Temporary record for swapping `varRecs` array elements.
        * `varFRec:VFRC`: Temporary record for file I/O.
        * `varFRecs(2):VFRC`: Array to hold two file records for swapping.
        * `sortedInt:INTEGER`: Optimization counter for the bubble sort.

---

### 2\. Call Graph (RUN & GOSUB)

A map of inter-procedure (`RUN`) and intra-procedure (`GOSUB`) dependencies.

* **`PROCEDURE udefVars`**
    * `RUN vSort(er,which,vPath,0,varCount-1,varRec)`: Calls the quicksort procedure to sort the `varRec` array and the corresponding records in `vFile`.
    * `RUN fSort(vPath,0,fRecs,varRec)`: Calls the bubble sort procedure to sort the field records.
    * `GOSUB 200`: Prints headers for variable reference display.
    * `GOSUB 250`: Prints a single formatted variable reference line.
    * `GOSUB 267`: Prints a counter during VDT validation.
    * `GOSUB 410`: Reads `DATA` statements to find a variable's base name.

* **`PROCEDURE vSort`**
    * `RUN vSort(er,which,vPath,bottom,lower-1,varRecs)`: Recursive call for the lower partition.
    * `RUN vSort(er,which,vPath,lower+1,top,varRecs)`: Recursive call for the upper partition.
    * No `GOSUB` calls.

* **`PROCEDURE fSort`**
    * No `RUN` calls.
    * No `GOSUB` calls.

---

### 3\. External Interaction Points

All statements that interact with the operating system, files, or hardware.

* **File I/O:**
    * `PROCEDURE udefVars`: `OPEN #dPath,dFile:READ` (Opens "procData")
    * `PROCEDURE udefVars`: `GET #dPath,hasRecords`
    * `PROCEDURE udefVars`: `GET #dPath,varRec`
    * `PROCEDURE udefVars`: `CLOSE #dPath`
    * `PROCEDURE udefVars`: `OPEN #vPath,vFile:UPDATE` (Opens "varDefs")
    * `PROCEDURE udefVars`: `GET #vPath,varCount`
    * `PROCEDURE udefVars`: `SEEK #path,[offset]` (Used with `descOff`, `symTabOff`, etc.)
    * `PROCEDURE udefVars`: `SEEK #vPath,[offset]` (Used for record-based access)
    * `PROCEDURE udefVars`: `GET #vPath,varFRec` (Reads records from "varDefs")
    * `PROCEDURE udefVars`: `GET #path,[variable]` (Reads data from main module file)
    * `PROCEDURE udefVars`: `PUT #vPath,varFRec` (Writes updated records to "varDefs")
    * `PROCEDURE udefVars`: `OPEN #dPath,dFile:UPDATE` (Re-opens "procData")
    * `PROCEDURE udefVars`: `PUT #dPath,hasRecords`
    * `PROCEDURE udefVars`: `PUT #dPath,varRec` (Saves sorted array)
    * `PROCEDURE udefVars`: `CLOSE #vPath` (In error handler)
    * `PROCEDURE udefVars`: `PRINT #2,[message]` (Prints status to Standard Error/Status)
    * `PROCEDURE vSort`: `SEEK #vPath,[offset]` (Used to find records for swapping)
    * `PROCEDURE vSort`: `GET #vPath,varFRec` (Reads records for swapping)
    * `PROCEDURE vSort`: `PUT #vPath,varFRec` (Writes records for swapping)
    * `PROCEDURE fSort`: `SEEK #vPath,[offset]` (Used to find records for swapping)
    * `PROCEDURE fSort`: `GET #vPath,varFRec` (Reads records for swapping)
    * `PROCEDURE fSort`: `PUT #vPath,varFRecs` (Writes records for swapping)
* **System/Shell I/O:**
    * None.
* **Memory/Hardware I/O:**
    * None.

---

### 4\. Logic Walk-Through

A step-by-step explanation of each procedure's execution flow.

* **`PROCEDURE udefVars`**
    * **Purpose:** The main procedure to identify, parse, sort, and rename all variable definitions from a compiled module's data sections.
    * **Parameters (`PARAM`):** Receives file paths, offsets (`descOff`, `symTabOff`), and flags (`verbose`) from a calling procedure.
    * **Main Logic:**
        1.  **Initialization:** Sets `BASE 0` and an error trap (`ON ERROR GOTO 500`).
        2.  **Load Data:** `OPEN`s `dFile` ("procData") and `GET`s the `hasRecords` flag and the `varRec` array (a variable lookup table).
        3.  **Open Var File:** `OPEN`s `vFile` ("varDefs") for `UPDATE` and `GET`s the `varCount`.
        4.  **Parse Variables (Main Loop):**
            * It enters a `REPEAT...UNTIL` loop that runs `varCount` times.
            * Inside the loop, it `GET`s a `varFRec` (Variable File Record) from `vFile`.
            * It analyzes `varFRec.vp3.mTyp` (memory type) to determine if the variable is a simple atomic (`mTyp=1`), a string (`mTyp=2`), or a VDT-referenced complex type (`mTyp=3`).
            * Based on the type, it `SEEK`s into the main module file (using `path` and `descOff`) to `GET` additional details (like offsets, sizes, and array dimensions).
            * It uses `GOSUB 410` to look up a base name (e.g., "I", "S", "CA()") from `DATA` statements.
            * It populates the `varFRec` with all discovered metadata.
            * It `PUT`s the completed `varFRec` back into `vFile`.
        5.  **Sort Data:**
            * Calls `RUN vSort` to quicksort the `varRec` array in memory and simultaneously reorganize the `vFile` on disk, ordering them by `dmOff` (Data Memory Offset).
            * It then counts all field records (`fRecs`).
            * If fields exist, it calls `RUN fSort` to bubble sort the field records (both in `varRec` and `vFile`) based on `vOff`.
        6.  **Save Sorted Array:** `OPEN`s `dFile` for `UPDATE` and `PUT`s the now-sorted `varRec` array back into it.
        7.  **Rename Variables:**
            * Loops through `vFile` again.
            * If `varRec(recNum).nVar` is `TRUE`, it generates a new sequential name (e.g., "I001", "c001") for the variable.
            * It `PUT`s the record, now with its new name, back into `vFile`.
        8.  **Display & Validate:**
            * If `verbose=TRUE`, displays the final sorted records using `GOSUB 200` and `GOSUB 250`.
            * `SEEK`s to `symTabOff` in the main module file.
            * It reads the VDT (Variable Definition Table) entry by entry and compares it against the `vFile` records to ensure every VDT entry is accounted for ("Validated").
        9.  **Cleanup:** `END`s the procedure.
    * **Subroutines (GOSUB):**
        * `Line 200:` Prints headers for the verbose variable display.
        * `Line 250:` Prints a single, heavily formatted line of variable metadata.
        * `Line 267:` Prints a counter (used during verbose validation).
        * `Line 410:` `READ`s from `DATA` blocks `420` or `430` to match a token code to a string.
    * **Error Handling:**
        * `Line 500:` On error, captures the `ERR` code, `CLOSE`s any open files (`vPath`, `dPath`), and `END`s.

* **`PROCEDURE vSort`**
    * **Purpose:** To perform an in-place quicksort on both the in-memory `varRecs` array and the on-disk `vFile`.
    * **Parameters (`PARAM`):** The `varRecs` array (by reference), file path `vPath`, and sort bounds `bottom`/`top`.
    * **Main Logic:**
        1.  Implements a standard quicksort algorithm using `lower` and `upper` pointers.
        2.  The sort key is `varRecs(index).dmOff`.
        3.  When a swap is required (e.g., at `lower` and `upper`), it swaps the elements in the `varRecs` array *and* swaps the corresponding full records on disk in `vFile` using `SEEK`/`GET`/`PUT`.
        4.  Recursively `RUN`s itself for the remaining partitions.
    * **Error Handling:**
        * `Line 10:` `ON ERROR GOTO 10` sets a local trap. On error, it stores `ERR` in the `er` parameter and `END`s.

* **`PROCEDURE fSort`**
    * **Purpose:** To perform an in-place bubble sort on the *field* records.
    * **Parameters (`PARAM`):** The `varRecs` array (by reference), file path `vPath`, and counters.
    * **Main Logic:**
        1.  Implements a bubble sort.
        * The sort key is `varRecs(recCount).vOff`.
        * When a swap is required, it swaps the elements in the `varRecs` array *and* swaps the corresponding records on disk in `vFile` using `SEEK`/`GET`/`PUT`.
    * **Error Handling:**
        * None. Relies on the `ON ERROR GOTO 500` trap set in the calling procedure (`udefVars`).