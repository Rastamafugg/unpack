Analysis complete.

### 1\. Procedure Identification

The input code contains two primary `PROCEDURE` blocks and five local `GOSUB/RETURN` subroutines.

**Main Procedures:**

  * `PROCEDURE ubuildSrc`: The main program logic.
  * `PROCEDURE dsSort`: A standalone Quicksort algorithm used by `ubuildSrc`.

**GOSUB Subroutines (Local to `ubuildSrc`):**

  * **Label 200:** Array Naming Formatter.
  * **Label 210:** Data Type Appender.
  * **Label 268:** DSAT Counter Display.
  * **Label 269:** Validation Progress Display.
  * **Label 500:** Main Error Trap.

**GOSUB Subroutines (Local to `dsSort`):**

  * **Label 10:** `dsSort` Error Trap.

### 2\. Logic Walk-Through

#### `PROCEDURE ubuildSrc` (Main Logic)

This procedure reads several data files (`initDB`, `procData`, `varDefs`) to reconstruct the `TYPE`, `DIM`, and `PARAM` declarative statements of a Basic09 program.

1.  **Declarations:** Declares complex `TYPE` structures (VREC, VFRC, DSMP, RECS) and `DIM`s arrays and variables for file paths, flags, counters, and file records. It accepts `PARAM`s from a calling procedure, including file paths, memory offsets, and flags.
2.  **Initialization:** Sets `BASE 0` for array indexing. It initializes error flags and file status booleans. It sets a main error trap using `ON ERROR GOTO 500`.
3.  **Load Initial Data:**
      * Opens `initDB` file, reads `version` and `DSMap` (DSAT Map) array, skipping over a file record structure using `SEEK` and `SIZE`. Closes the file.
      * Opens `procData` file, reads `hasRecords` flag and the `varRec` (Variable Array Record) lookup table. Closes the file.
4.  **DSAT Processing:**
      * Opens the `varDefs` file for `UPDATE`.
      * Reads `varCount` from the file.
      * It checks if a Description Area exists (`symTabOff-descOff>3`).
      * **Build DSAT Map:** It loops (`FOR...NEXT`) through the `varDefs` file, reads each `varFRec`, and populates the `DSMap` array with offset and size information for variables that have a Description Area (`dsSiz>0`). It filters out duplicate entries.
      * **Sort DSAT Map:** Calls `RUN dsSort` to recursively sort the `DSMap` array based on the `dsOf` (offset) field.
      * **Validate DSAT:** It `SEEK`s to the `descOff` in the main program file (`#path`). It iterates through the sorted `DSMap` and compares the expected offsets (`dsOf`) with the current file pointer (`dsPointer`).
          * If `dsOf > dsPointer`, it registers this as a "gap" of unidentified data, reads the bytes, and reports it.
          * It reads the expected data block (`dsSz`) and continues.
          * This process validates that the `DSMap` correctly describes the data layout.
      * **Fix Type Records:** It performs a pass over the `varDefs` file to link complex `TYPE` fields. It finds a parent record (`pVar`) and then scans forwards and backwards, using `SEEK`, `GET`, and `PUT`, to update related fields (`vRecNum`). This ensures all fields of a record type share the same ID.
5.  **Validate Data Memory:**
      * Performs a `REPEAT/UNTIL` loop reading `varDefs` to ensure there are no gaps between variables in the Data Memory block. It compares `varRec(cntr).dmOff+varFRec.vp1.dmSiz` with `varRec(cntr+1).dmOff`.
      * Calls `GOSUB 269` to print progress dots.
6.  **Source Code Generation:**
      * Initializes a `rec` array to track `TYPE` definitions and resets loop counters.
      * Enters a main `REPEAT/UNTIL` loop to read `varDefs` one record at a time.
      * **`nTyp` Switch:** It checks `varRec(recNum).nTyp` to determine what kind of statement to build.
      * **`nTyp=0` (TYPE):**
          * Builds `TYPE` statements.
          * Calls `GOSUB 200` to format array dimensions.
          * Groups variables of the same type (e.g., `a,b:INTEGER`).
          * When the type changes (e.g., to `REAL`), it calls `GOSUB 210` to append the type string (e.g., `:INTEGER`).
          * When a `TYPE` block is complete (`pVar` changes), it prints the `source` string to the output file (`#oPath`) and resets `source`.
      * **`nTyp=1` (DIM):**
          * Same logic, but builds `DIM` statements.
      * **`nTyp=2` (PARAM):**
          * Same logic, but builds `PARAM` statements.
      * The loop continues until `recNum=varCount`.
7.  **Shutdown:**
      * Closes the `varDefs` file.
      * `! CHAIN chainMod`: This line is a comment, as `!` is a synonym for `REM`. The `CHAIN` command is not executed.
      * The procedure finishes with `END`.

-----

#### `GOSUB 200` (Array Naming Formatter)

  * **Purpose:** To replace array placeholders with formatted dimension strings.
  * **Logic:**
    1.  Checks if the `vName` string ends in `(,,)`, `(,)`, or `()`.
    2.  If a match is found, it strips the placeholder and appends the `vArray` string (which contains the actual dimensions, e.g., `(20,10)`).
    3.  `RETURN`.

-----

#### `GOSUB 210` (Type Naming Appender)

  * **Purpose:** To append the correct data type suffix to a `DIM`, `PARAM`, or `TYPE` statement.
  * **Logic:**
    1.  Inspects the prefix character of the `tName` variable (e.g., 'B' for `BYTE`, 'S' for `STRING`).
    2.  Appends the corresponding type string (e.g., `:BYTE`, `:INTEGER`) to the global `source` string.
    3.  If `STRING`, it also appends the length (e.g., `[80]`) if it is not the default `[32]`.
    4.  If a complex type (`'C'`):
          * It searches the `rec` array for a matching `recID`.
          * If found, it appends the known `recName` (e.g., `:TYP1`).
          * If *not* found (an unclassified type), it generates a *new* `TYPE` definition on the fly (e.g., `TYPE TYP2=uC1(100):BYTE`), prints this new `TYPE` statement to the output file immediately, and appends the new `typName` to the `source` string.
    5.  `RETURN`.

-----

#### `GOSUB 268` & `GOSUB 269` (Progress Display)

  * **Purpose:** To print status information.
  * **Logic (268):** Prints the `dsatCnt` counter using `PRINT USING`.
  * **Logic (269):** Prints the `cntr` if `verbose` is true, otherwise prints a `.` as a progress indicator.
  * `RETURN`.

-----

#### `GOSUB 500` (Main Error Trap)

  * **Purpose:** To handle runtime errors and shut down cleanly.
  * **Logic:**
    1.  Activated by `ON ERROR GOTO 500`.
    2.  Captures the error code using the `ERR` function.
    3.  Closes any open files (`#vPath`, `#dPath`) to prevent data corruption.
    4.  `END` statement terminates all execution.

-----

#### `PROCEDURE dsSort` (Quicksort)

  * **Purpose:** A recursive Quicksort algorithm to sort the `DSMap` array.
  * **Logic:**
    1.  Receives parameters: `er`, `which`, `bottom`, `top`, and the array `DSMaps`.
    2.  Sets a local error trap `ON ERROR GOTO 10` and `BASE 0`.
    3.  Implements a standard Quicksort algorithm using the `top` element as the pivot.
    4.  Uses a main `LOOP/ENDLOOP`.
    5.  Uses two `REPEAT/UNTIL` loops to find the partition points (`lower` and `upper`) based on the `dsOf` field.
    6.  Swaps elements using a temporary `DSMap` variable. This is a full structure assignment.
    7.  Uses `EXITIF` to break the loop when partitions meet.
    8.  Recursively calls `RUN dsSort` for the lower and upper partitions.
    9.  The error trap at label `10` passes the `ERR` code back to the `er` parameter, which was passed by reference.

### 3\. Process and Workflow Synthesis (PWS) Templates

#### PWS Template: `ubuildSrc`

  * **Identifier:** `PROCEDURE ubuildSrc`
  * **Summary:** Reconstructs Basic09 declarative source code (`TYPE`, `DIM`, `PARAM`) by reading, parsing, and validating binary definition files (`initDB`, `procData`, `varDefs`).
  * **Inputs (Parameters):**
      * `path`, `oPath`: `BYTE` (File paths)
      * `er`, `dataSiz`: `INTEGER` (Error flag, Data size)
      * `pOpen`, `verbose`: `BOOLEAN` (Flags)
      * `descOff`, `symTabOff`: `REAL` (File offsets)
      * `dataDir`, `oFile`: `STRING` (File names/paths)
  * **Outputs (State):**
      * Generates a source code file at path `oFile` (via `#oPath`).
      * Modifies the `varDefs` file (via `#vPath`).
      * Prints status messages to Standard Error (path `#2`).
      * Sets `er` parameter if an error occurs.
  * **Process:**
    1.  **Initialize:** Set `BASE 0`, set `ON ERROR GOTO 500`.
    2.  **Load Data:** `OPEN`/`GET`/`CLOSE` `initDB` and `procData` to load `DSMap` and `varRec` arrays.
    3.  **Process DSAT:**
          * `OPEN` `varDefs`.
          * Loop (`FOR`) through `varDefs`, read `varFRec`, and build `DSMap`.
          * `RUN dsSort` to sort `DSMap`.
          * `SEEK` to `descOff` in `#path`.
          * Loop (`REPEAT/UNTIL`) to validate `DSMap` against file data, identifying gaps.
          * Loop (`FOR`) through `varDefs` to link complex type records (`pVar`), updating `varDefs` file with `PUT`.
    4.  **Validate Memory:** Loop (`REPEAT/UNTIL`) through `varRec` to check for gaps in `dmOff`.
    5.  **Generate Source:**
          * Initialize `rec` array and state variables.
          * Loop (`REPEAT/UNTIL`) from `recNum=0` to `varCount-1`.
          * `GET` `varFRec`.
          * `IF varRec(recNum).nTyp = 0`: Build `TYPE` string.
          * `IF varRec(recNum).nTyp = 1`: Build `DIM` string.
          * `IF varRec(recNum).nTyp = 2`: Build `PARAM` string.
          * Use `GOSUB 200` (Format Array) and `GOSUB 210` (Append Type).
          * `PRINT` completed `source` string to `#oPath`.
    6.  **Cleanup:** `CLOSE #vPath`.
    7.  **Terminate:** `END`.
  * **Dependencies:**
      * `GOSUB 200` (Array Naming)
      * `GOSUB 210` (Type Naming)
      * `GOSUB 268`/`269` (Progress Display)
      * `GOSUB 500` (Error Trap)
      * `PROCEDURE dsSort`

#### PWS Template: `dsSort`

  * **Identifier:** `PROCEDURE dsSort`
  * **Summary:** Implements a recursive Quicksort algorithm to sort an array of `DSMP` (DSAT Map) structures.
  * **Inputs (Parameters):**
      * `er`, `which`: `INTEGER` (Error flag, recursion depth)
      * `bottom`, `top`: `INTEGER` (Array indices for partition)
      * `DSMaps(100):DSMP` (Array of structures, passed by reference)
  * **Outputs (State):**
      * Sorts the `DSMaps` array (passed by reference) in place.
      * Sets `er` parameter if an error occurs.
      * Prints `.` to Standard Error (path `#2`) for progress.
  * **Process:**
    1.  **Initialize:** Set `ON ERROR GOTO 10`, `BASE 0`.
    2.  **Partition:**
          * Set `lower:=bottom`, `upper:=top`.
          * Enter `LOOP/ENDLOOP`.
          * `REPEAT/UNTIL`: Find `lower` partition point (where `dsOf >=` pivot).
          * `REPEAT/UNTIL`: Find `upper` partition point (where `dsOf <=` pivot).
          * `EXITIF lower=upper`.
          * **Swap:** Swap `DSMaps(lower)` and `DSMaps(upper)` using a temporary `DSMap` variable.
    3.  **Swap Pivot:** Swap final `lower` element with `top` (pivot).
    4.  **Recurse:**
          * `IF bottom < lower-1 THEN RUN dsSort(er,which,bottom,lower-1,DSMaps)`.
          * `IF lower+1 < top THEN RUN dsSort(er,which,lower+1,top,DSMaps)`.
    5.  **Terminate:** `END`.
  * **Dependencies:**
      * `GOSUB 10` (Error Trap)
