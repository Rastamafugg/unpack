Based on your request, I have performed a deconstruction of the provided `udecode` and `lSort` procedures. All analysis is strictly confined to the features and syntax present in the `Basic09-only-reference.md` documentation.

### 1\. Identified Procedures and Main Logic

The input code contains one main procedure, `PROCEDURE udecode`, and one external procedure, `PROCEDURE lSort`.

`PROCEDURE udecode` is organized using `GOSUB` statements to manage its workflow. The identified logic blocks are:

  * **Main Logic (udecode):** The primary execution flow responsible for initialization, high-level control, and final data saving.
  * **GOSUB 200 (Instruction Decode):** The core I-Code parser.
  * **GOSUB 210 (Add Variable Record):** A "find-or-add" routine for variable references.
  * **GOSUB 220 (Add Line Record):** A "find-or-add" routine for line number references.
  * **GOSUB 300 (Sort/Number Line References):** Post-processing for line references.
  * **GOSUB 400 (Get Data from initDB):** Initialization data loader.
  * **GOSUB 500 (Error Trap):** The main error handling routine.

`PROCEDURE lSort` is a standalone procedure invoked by `RUN` and contains its own error trap.

### 2\. Logic Walk-Through

#### Main Logic (PROCEDURE udecode)

1.  **Parameters:** Accepts parameters (`PARAM`) defining the I-Code file path, error variables, flags, file offsets, and names.
2.  **Initialization:** Sets `BASE 0`. It initializes numerous `BOOLEAN` flags and `INTEGER`/`REAL` counters to `FALSE` or `0`.
3.  **Error Trap:** Sets the main error trap using `ON ERROR GOTO 500`.
4.  **Load Initial Data:** Calls `GOSUB 400` to open and read configuration data from the "initDB" file.
5.  **Open Work Files:** Opens the variable definitions ("varDefs") and line definitions ("linDefs") files in `UPDATE` mode, assigning them to `vPath` and `lnPath`.
6.  **Read Variable Count:** `SEEK`s to the symbol table offset in the I-Code file (`path`) and `GET`s the total variable count (`tpVars`).
7.  **Core Decode:** Calls `GOSUB 200` to execute the main I-Code parsing logic.
8.  **Process Line References:** If line references were found (`linCount > 0`), it calls `GOSUB 300` to sort and number them.
9.  **Print Summary:** `PRINT`s summary statistics (variable counts, instruction count) to the standard error path (`#2`).
10. **Save Data:** `CLOSE`s the work files. It then `OPEN`s "procData" for `WRITE` and `PUT`s the collected data: `hasRecords`, the entire `varRec` array, and all reference counts.
11. **Terminate:** `CLOSE`s the final data file and `END`s the procedure.

-----

#### GOSUB 200: Instruction Decode

1.  **Setup:** `SEEK`s to the I-Code execution offset (`execOff`) in the file (`path`).
2.  **Statement Loop:** Begins a `REPEAT`/`UNTIL` loop that processes one full statement at a time, continuing until the module offset (`modCnt`) reaches the end of the execution block.
3.  **Token Loop:** Starts a nested `REPEAT`/`UNTIL` loop to process tokens within a single statement.
4.  **Token Read:** `GET`s a single `BYTE` token (`tokenVal`) from `path`.
5.  **Token Parsing:** Uses a series of `IF`/`THEN` blocks to check the hex value of `tokenVal`.
      * **Flagging:** Sets flags based on statement type (e.g., `isOn:=$1D`, `creop:=$29`).
      * **Variable References:** If a token indicates a variable (e.g., `$13` FOR, `$81` INTEGER var), it `GET`s the following 2-byte offset (`intVal`) and calls `GOSUB 210` to log it.
      * **Line References:** If a token indicates a line reference (e.g., `$20` GOTO, `$22` GOSUB), it `GET`s the 2-byte offset (`intVal`) and calls `GOSUB 220` to log it.
      * **Literals:** If a token indicates a literal (e.g., `$8F` REAL, `$90` STRING), it `GET`s the appropriate number of bytes (5 for REAL, N for STRING) to advance the file pointer past the literal's data.
6.  **Loop End:** The inner loop terminates on an end-of-statement token (`$3E` or `$3F`). The outer loop increments the instruction counter (`tInstr`) and repeats.
7.  **Return:** After the outer loop finishes, the subroutine `RETURN`s.

-----

#### GOSUB 210: Add Variable Record

1.  **Flagging:** Sets `isField` and `potRec` booleans based on `tokenVal` to identify data structure access.
2.  **Search:** If `varCount > 0`, it executes a `REPEAT`/`UNTIL` loop to scan the `varRec` array for an existing entry matching `tokenVal` and `intVal`.
3.  **Found:** If a match is found, it `SEEK`s to that record's position in the `vPath` file, `GET`s the `varFRec`, and sets `found:=TRUE`.
4.  **Not Found (Add):** If `NOT(found)`, it initializes a new `varRec` entry, increments `varCount`, sets default `varFRec` fields (like `vRefCnt:=1`), and seeks to the end of the `vPath` file.
5.  **Update/Write:** It updates `varFRec` fields (e.g., linking fields to records, or incrementing `vRefCnt` if `found`).
6.  **Commit:** `PUT`s the `varFRec` to the `vPath` file.
7.  **Update Header:** If a new record was added, it `SEEK`s to the start of `vPath` (position 0) and `PUT`s the new `varCount`.
8.  **Return:** `RETURN`s to `GOSUB 200`.

-----

#### GOSUB 220: Add Line Record

1.  **Search:** If `linCount > 0`, it executes a `REPEAT`/`UNTIL` loop to scan the `linRefs` array for an existing entry matching `tokenVal` and `intVal`.
2.  **Found:** If a match is found, it `SEEK`s to that record's position in `lnPath`, `GET`s `linRec`, increments `linRec.lnRefCnt`, `SEEK`s back, and `PUT`s the updated record.
3.  **Not Found (Add):** If `NOT(found)`, it `SEEK`s to the end of the `lnPath` file.
4.  **Update Array:** Adds `tokenVal` and `intVal` to the in-memory `linRefs` array at the `linCount` index. Increments `linCount`.
5.  **Write Record:** Initializes a new `linRec` and `PUT`s it to the file.
6.  **Update Header:** `SEEK`s to the start of `lnPath` (position 0) and `PUT`s the new `linCount`.
7.  **Return:** `RETURN`s to `GOSUB 200`.

-----

#### GOSUB 300: Sort/Number Line References

1.  **Sort:** If `linCount > 1`, it `RUN`s the external procedure `lSort`, passing it the `lnPath` and the `linRefs` array to sort the line references by their file offset.
2.  **Numbering Loop:** If sorting occurred (`renum=TRUE`), it `SEEK`s to the start of data in `lnPath`.
3.  **Update Records:** It loops (`REPEAT`/`UNTIL`) through all `linCount` records. In each loop:
      * `GET`s the `linRec`.
      * Assigns a line number string (e.g., "10", "20") using `STR$`. It reuses the previous number (`lastNum`) if the offset (`linRec.lnOff`) is identical to the previous record's offset, ensuring identical offsets map to the same line number.
      * `SEEK`s back to the record's position and `PUT`s the updated `linRec`.
4.  **Return:** `RETURN`s to the main logic.

-----

#### GOSUB 400: Get Data from initDB

1.  **Open:** `OPEN`s the `dFile` ("initDB") in `READ` mode.
2.  **Read:** `GET`s three data structures from the file: `version`, `varFRec` (an initialized record), and `DSMap` (an array).
3.  **Close:** `CLOSE`s the `dPath`.
4.  **Return:** `RETURN`s to the main logic.

-----

#### GOSUB 500: Error Trap

1.  **Get Error Code:** If the global `er` variable is 0, it retrieves the system error code using `er:=ERR`.
2.  **Specific Errors:** Handles errors `56` or `216`. If `216`, it attempts to recover by `GOTO 20`.
3.  **Cleanup:** In all error cases, it checks `vOpen`, `lnOpen`, and `dOpen` flags. If any are `TRUE`, it `CLOSE`s the corresponding file path.
4.  **Terminate:** `END`s the program execution.

-----

#### PROCEDURE lSort

1.  **Parameters:** Accepts error var (`er`), recursion counter (`which`), file path (`lnPath`), bounds (`bottom`, `top`), and the `linRefs` array.
2.  **Setup:** Sets `BASE 0` and an `ON ERROR GOTO 10` trap.
3.  **Sort Logic:** Implements a quicksort algorithm based on the `qsortl` example.
4.  **Partitioning:** Uses `LOOP`/`ENDLOOP` and nested `REPEAT`/`UNTIL` loops to find elements to swap, comparing `linRefs(lower).hLOff` (the file offset).
5.  **Dual Swap:** When a swap is required, it performs two swaps:
      * It swaps the elements in the in-memory `linRefs` array.
      * It `SEEK`s to the `lower` and `upper` positions in the `lnPath` file, `GET`s both `linRec` records, swaps them in memory, and `PUT`s them back to their new file positions.
6.  **Recursion:** `RUN`s itself recursively for the remaining partitions.
7.  **Error Trap (10):** On error, sets `er:=ERR` and `END`s.

### 3\. Process and Workflow Synthesis (PWS) Templates

#### PWS: GOSUB 200 (Instruction Decode)

  * **Purpose:** To parse a Basic09 I-Code file byte-by-byte, identify all variable and line references, and log them.
  * **Process:**
    1.  **Input:** I-Code file path (`path`), execution offset (`execOff`), descriptor offset (`descOff`).
    2.  **Initialization:** `SEEK` to `execOff` in `path`.
    3.  **Statement Loop:** Start a `REPEAT` loop (terminates when file offset = `descOff`).
    4.  **Token Loop:** Start a nested `REPEAT` loop (terminates on EOL token `$3E`/`$3F`).
    5.  **Read Token:** `GET` one `BYTE` token.
    6.  **Identify Token Type:**
          * `IF` token is a Variable Reference: `GET` 2-byte offset. `GOSUB 210` (Log Variable).
          * `IF` token is a Line Reference: `GET` 2-byte offset. `GOSUB 220` (Log Line Ref).
          * `IF` token is a Literal (String, Real, etc.): `GET` N bytes to skip data.
          * `IF` token is a flag (ON, OPEN, etc.): Set corresponding boolean flag.
    7.  **End Loops:** Close inner, then outer loops.
    8.  **Output:** Populated variable/line reference files (`vPath`, `lnPath`) and counts.
    9.  **Action:** `RETURN`.

-----

#### PWS: GOSUB 210 (Add Variable Record)

  * **Purpose:** To log a variable reference, ensuring uniqueness based on token and offset.
  * **Process:**
    1.  **Input:** `tokenVal` (type), `intVal` (offset), `varCount`, `vPath` file, `varRec` array.
    2.  **Identify Type:** Set `isField` / `potRec` flags based on `tokenVal`.
    3.  **Search:** `REPEAT` loop `0` to `varCount` on `varRec`.
          * `IF varRec(i).vTyp = tokenVal AND varRec(i).vOff = intVal THEN`
              * `found:=TRUE`.
              * `SEEK` to record `i` in `vPath`.
              * `GET` `varFRec`.
              * `EXITIF` loop.
          * `ENDIF`.
    4.  **Handle Result:**
          * `IF found THEN`: Increment `varFRec.vp2.vRefCnt`.
          * `IF NOT(found) THEN`:
              * Initialize new `varRec(varCount)`.
              * Initialize new `varFRec` (set `vRefCnt:=1`).
              * `SEEK` to end of `vPath`.
              * Increment `varCount`.
    5.  **Write Record:** `PUT` the (updated or new) `varFRec` to `vPath`.
    6.  **Update Header:** `IF NOT(found) THEN`: `SEEK` to `0` in `vPath`, `PUT varCount`.
    7.  **Output:** Updated `vPath` file, `varCount`, and `varRec` array.
    8.  **Action:** `RETURN`.

-----

#### PWS: GOSUB 220 (Add Line Record)

  * **Purpose:** To log a line reference, ensuring uniqueness based on token and offset.
  * **Process:**
    1.  **Input:** `tokenVal` (type), `intVal` (offset), `linCount`, `lnPath` file, `linRefs` array.
    2.  **Search:** `REPEAT` loop `0` to `linCount` on `linRefs`.
          * `IF linRefs(i).hLTyp = tokenVal AND linRefs(i).hLOff = intVal THEN`
              * `found:=TRUE`.
              * `SEEK` to record `i` in `lnPath`.
              * `GET` `linRec`.
              * Increment `linRec.lnRefCnt`.
              * `SEEK` back to record `i`.
              * `PUT linRec`.
              * `EXITIF` loop.
          * `ENDIF`.
    3.  **Handle Result:**
          * `IF NOT(found) THEN`:
              * `SEEK` to end of `lnPath`.
              * Update `linRefs(linCount)` with `tokenVal`, `intVal`.
              * Initialize new `linRec` (set `lnRefCnt:=1`).
              * `PUT linRec`.
              * Increment `linCount`.
    4.  **Update Header:** `IF NOT(found) THEN`: `SEEK` to `0` in `lnPath`, `PUT linCount`.
    5.  **Output:** Updated `lnPath` file, `linCount`, and `linRefs` array.
    6.  **Action:** `RETURN`.

-----

#### PWS: GOSUB 300 (Sort/Number Line References)

  * **Purpose:** To sort the logged line references by offset and assign sequential line numbers.
  * **Process:**
    1.  **Input:** `linCount`, `lnPath` file, `linRefs` array.
    2.  **Validate:** `IF linCount > 1 THEN`.
    3.  **Sort:** `RUN lSort` procedure, passing `lnPath` and `linRefs`.
    4.  **Number:** `IF renum=TRUE THEN`.
    5.  **Loop:** `SEEK` to start of `lnPath` data. `REPEAT` loop `0` to `linCount-1`.
    6.  **Read:** `GET linRec`.
    7.  **Assign Number:**
          * `IF linRec.lnOff = lastOff THEN` (duplicate offset):
              * `linRec.lnNum := lastNum`.
          * `ELSE` (new offset):
              * `linRec.lnNum := STR$((lnCnt+1)*10)`.
              * `lastOff := linRec.lnOff`.
              * `lastNum := linRec.lnNum`.
          * `ENDIF`.
    8.  **Write:** `SEEK` back to the record position. `PUT linRec`.
    9.  **End Loop.**
    10. **Output:** An updated `lnPath` file where all records have a `lnNum` string.
    11. **Action:** `RETURN`.

-----

#### PWS: GOSUB 400 (Get Data from initDB)

  * **Purpose:** To load initial configuration data from a file.
  * **Process:**
    1.  **Input:** `dFile` (filename string).
    2.  **Action:** `OPEN #dPath, dFile:READ`.
    3.  **Action:** `GET #dPath, version`.
    4.  **Action:** `GET #dPath, varFRec`.
    5.  **Action:** `GET #dPath, DSMap`.
    6.  **Action:** `CLOSE #dPath`.
    7.  **Output:** Populated `version`, `varFRec`, and `DSMap` variables.
    8.  **Action:** `RETURN`.

-----

#### PWS: GOSUB 500 (Error Trap)

  * **Purpose:** To gracefully handle runtime errors, close files, and terminate.
  * **Process:**
    1.  **Input:** `er` (global error code), `vOpen`, `lnOpen`, `dOpen` (file status flags).
    2.  **Get Error:** `IF er=0 THEN er:=ERR`.
    3.  **Check Recovery:** `IF er=216 THEN GOTO 20`.
    4.  **Cleanup:**
          * `IF vOpen THEN CLOSE #vPath`.
          * `IF lnOpen THEN CLOSE #lnPath`.
          * `IF dOpen THEN CLOSE #dPath`.
    5.  **Action:** `END`.

-----

#### PWS: PROCEDURE lSort

  * **Purpose:** To (recursively) sort records in a file (`lnPath`) based on an in-memory array (`linRefs`) using the Quicksort algorithm.
  * **Process:**
    1.  **Input:** `lnPath`, `linRefs` array, `bottom`, `top` (bounds).
    2.  **Setup:** Set `BASE 0`. Set error trap.
    3.  **Partition:** `LOOP`/`REPEAT` logic to find `lower` and `upper` indices to swap based on `linRefs(i).hLOff`.
    4.  **Swap:**
          * Swap `linRefs(lower)` and `linRefs(upper)` in memory.
          * `SEEK`/`GET` records at `lower` and `upper` from `lnPath` into a temp array.
          * Swap records in the temp array.
          * `SEEK`/`PUT` records back to `lnPath` at their new positions.
    5.  **Recurse:**
          * `IF bottom < lower-1 THEN RUN lSort(..., bottom, lower-1, ...)`
          * `IF lower+1 < top THEN RUN lSort(..., lower+1, top, ...)`
    6.  **Output:** The `lnPath` file is physically re-sorted on disk, and the `linRefs` array is sorted in memory.
    7.  **Action:** `END`.
