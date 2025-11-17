**Code Maintenance Report**

### 1\. Procedure & Data Analysis

Analysis of all procedures, their local data, and shared data structures.

**User-Defined Data (`TYPE`)**

  * `TYPE VREC`: Declared in `ubuildSrc`. A record for the variable array, storing type info, offsets, and a boolean flag.
  * `TYPE VFP1`: Declared in `ubuildSrc`. Part 1 of a variable file record, storing pointers and size information (all `INTEGER`).
  * `TYPE VFP2`: Declared in `ubuildSrc`. Part 2 of a variable file record, storing names, lengths, and reference counts.
  * `TYPE VFP3`: Declared in `ubuildSrc`. Part 3 of a variable file record, storing type/link bytes and boolean flags.
  * `TYPE VFRC`: Declared in `ubuildSrc`. A composite "Variable File Record" combining `VFP1`, `VFP2`, and `VFP3`.
  * `TYPE DSMP`: Declared in `ubuildSrc` and `dsSort`. A "DSAT Map" record storing a `REAL` offset and three `INTEGER` size/dimension values. Note: This `TYPE` must be declared in any procedure that passes or receives it.
  * `TYPE RECS`: Declared in `ubuildSrc`. A record for mapping `INTEGER` IDs to `STRING[7]` names.

**Procedure Interface & Storage**

  * **`PROCEDURE ubuildSrc`**

      * **Parameters (`PARAM`):**
          * `path`, `oPath`: Input file path and output file path, type `BYTE`.
          * `er`: Error code container, type `INTEGER`.
          * `dataSiz`: Size of data, type `INTEGER`.
          * `pOpen`, `verbose`: Boolean flags, type `BOOLEAN`.
          * `descOff`, `symTabOff`: File offsets, type `REAL`.
          * `dataDir`: Data directory path, type `STRING[16]`.
          * `oFile`: Output file name, type `STRING[29]`.
      * **Local Variables (`DIM`):**
          * `varRec(400):VREC`: Array of 400 `VREC` structures.
          * `varFRec`, `varFRec2`: Working variables of type `VFRC`.
          * `DSMap(100):DSMP`: Array of 100 `DSMP` structures (DSAT Map).
          * `rec(20):RECS`: Array of 20 `RECS` structures.
          * `vPath`, `dPath`: File path numbers, type `BYTE`.
          * `vOpen`, `dOpen`, `found`, `validated`, `endOfType`, `hasRecords`: Boolean flags.
          * `version`: `STRING[8]`.
          * `vFile`, `lnFile`, `ltFile`, `dFile`: File name strings, type `STRING[29]`.
          * `tnTyp`, `byteVal`, `posVal`, `varCntr`: `BYTE` counters.
          * `varRefs`, `varCount`, `recCnt`, `modSiz`, `lastOff`, `dataStOff`, `recNum`, `intVal`, `which`, `VDCount`, `dsatCnt`, `ucCnt`, `gapSize(20)`, `gapCnt`, `dsMCnt`, `typCnt`, `cntr`, `cntr2`, `tRec`, `tSiz`, `tVar`: Various `INTEGER` counters and storage.
          * `mSize`, `dsPointer`, `nextOff`, `gapOff(20)`: `REAL` values for offsets and sizes.
          * `tStrLen`, `typName`: `STRING[7]`.
          * `tName`: `STRING[29]`.
          * `source`: `STRING[512]`.

  * **`PROCEDURE dsSort`**

      * **Parameters (`PARAM`):**
          * `er`, `which`: Error code and recursion counter, type `INTEGER`.
          * `bottom`, `top`: Array bounds for sorting, type `INTEGER`.
          * `DSMaps(100):DSMP`: The `DSMP` array passed by reference to be sorted.
      * **Local Variables (`DIM`):**
          * `DSMap:DSMP`: A temporary `DSMP` structure used for swapping elements.
          * `lower`, `upper`: Loop counters, type `INTEGER`.
          * `btemp`: Boolean flag, type `BOOLEAN`.

-----

### 2\. Call Graph (RUN & GOSUB)

A map of inter-procedure (`RUN`) and intra-procedure (`GOSUB`) dependencies.

  * **`PROCEDURE ubuildSrc`**
      * `RUN dsSort(er,which,0,dsMCnt-1,DSMap)`: Sorts the local `DSMap` array based on file offsets.
      * `GOSUB 268`: Displays the `dsatCnt` counter value.
      * `GOSUB 200`: Formats array variable names in the `source` string (e.g., converts `name()` to `name(elem1)`).
      * `GOSUB 210`: Appends the correct data type (e.g., `:BYTE`, `:STRING[n]`) to the `source` string.
      * `GOSUB 269`: Prints a progress indicator (`.` or full count).
  * **`PROCEDURE dsSort`**
      * `RUN dsSort(er,which,bottom,lower-1,DSMaps)`: Recursive call to sort the lower partition of the array.
      * `RUN dsSort(er,which,lower+1,top,DSMaps)`: Recursive call to sort the upper partition of the array.
      * No `GOSUB` calls.

-----

### 3\. External Interaction Points

All statements that interact with the operating system, files, or hardware.

  * **File I/O:**
      * `PROCEDURE ubuildSrc`: `OPEN #dPath,dFile:READ` (Opens `initDB` and `procData` files)
      * `PROCEDURE ubuildSrc`: `OPEN #vPath,vFile:UPDATE` (Opens `varDefs` file for reading and writing)
      * `PROCEDURE ubuildSrc`: `PRINT #2,...` (Prints status messages to standard error/status path)
      * `PROCEDURE ubuildSrc`: `PRINT #oPath,source` (Writes generated source code to the output file path `oPath`)
      * `PROCEDURE ubuildSrc`: `GET #dPath,version`
      * `PROCEDURE ubuildSrc`: `GET #dPath,DSMap`
      * `PROCEDURE ubuildSrc`: `GET #dPath,hasRecords`
      * `PROCEDURE ubuildSrc`: `GET #dPath,varRec`
      * `PROCEDURE ubuildSrc`: `GET #vPath,varCount`
      * `PROCEDURE ubuildSrc`: `GET #vPath,varFRec`
      * `PROCEDURE ubuildSrc`: `GET #vPath,varFRec2`
      * `PROCEDURE ubuildSrc`: `GET #path,byteVal`
      * `PROCEDURE ubuildSrc`: `GET #path,intVal`
      * `PROCEDURE ubuildSrc`: `PUT #vPath,varFRec` (Updates `varFRec` in the `varDefs` file)
      * `PROCEDURE ubuildSrc`: `SEEK #dPath,...` (Positions file pointer in `dPath`)
      * `PROCEDURE ubuildSrc`: `SEEK #vPath,...` (Positions file pointer in `vPath`)
      * `PROCEDURE ubuildSrc`: `SEEK #path,...` (Positions file pointer in input `path`)
      * `PROCEDURE ubuildSrc`: `CLOSE #dPath`
      * `PROCEDURE ubuildSrc`: `CLOSE #vPath`
      * `PROCEDURE dsSort`: `PRINT #2,".";` (Prints progress dots to standard error/status path)
  * **System/Shell I/O:**
      * `PROCEDURE ubuildSrc`: `! chainMod:=...` (This is a commented-out line, `!` = `REM`)
      * `PROCEDURE ubuildSrc`: `! CHAIN chainMod` (This is a commented-out line, `!` = `REM`)
  * **Memory/Hardware I/O:**
      * None found.

-----

### 4\. Logic Walk-Through

A step-by-step explanation of each procedure's execution flow.

  * **`PROCEDURE ubuildSrc`**

      * **Purpose:** To read multiple binary data definition files, validate their internal structures (DSAT map, data memory), and reconstruct the original Basic09 `TYPE`, `DIM`, and `PARAM` source code statements, writing them to an output file.
      * **Parameters (`PARAM`):** Receives file paths (`path`, `oPath`), error status (`er`), size/offset data (`dataSiz`, `descOff`, `symTabOff`), flags (`pOpen`, `verbose`), and directory/file names (`dataDir`, `oFile`).
      * **Main Logic:**
        1.  Initializes `BASE 0`, sets the error handler `ON ERROR GOTO 500`, and initializes file status flags.
        2.  Opens `initDB` file. `GET`s `version`, `SEEK`s past a header, and `GET`s the `DSMap` array. `CLOSE`s file.
        3.  Sets string variables for file names (`vFile`, `lnFile`, `dFile`).
        4.  Opens `procData` file. `GET`s `hasRecords` flag and the `varRec` lookup table. `CLOSE`s file.
        5.  Opens `varDefs` file (`vFile`) for `UPDATE` and `GET`s the `varCount`.
        6.  **DSAT Validation:** If the description area has content (`symTabOff-descOff>3`):
              * Loops from 0 to `varCount-1`, `GET`ting each `varFRec` from `vFile`.
              * Populates the local `DSMap` array with data from `varFRec`, skipping duplicates.
              * Calls `RUN dsSort` to sort the `DSMap` array by offset.
              * **Validation Loop:** `SEEK`s to `descOff` in the main input file (`#path`). It then walks through the file byte by byte, comparing its position (`dsPointer`) to the sorted `DSMap` offsets. It `GET`s data to advance the pointer, reporting any gaps (unidentified data) between `DSMap` entries.
              * **Record ID Fixing:** Loops through `vFile` again. For variables that are fields of a record (`pVar>0`), it finds the parent record and `PUT`s an updated `vRecNum` to ensure all fields of the same record share a common ID.
        7.  **Data Memory Validation:** Loops through `vFile` and `varRec` data, `GET`ting records and checking if the `dmOff` (data memory offset) plus `dmSiz` (size) matches the `dmOff` of the *next* variable. This validates that the data memory allocation is contiguous.
        8.  **Source Code Generation:**
              * Initializes the `rec` array and counters.
              * Enters a `REPEAT...UNTIL recNum=varCount` loop to read `vFile` one last time.
              * `SEEK`s and `GET`s the `varFRec` for the current `recNum`.
              * **If `varRec(recNum).nTyp=0`:** Builds a `TYPE` statement string. It groups variables of the same type (e.g., `a,b:INTEGER`).
              * **If `varRec(recNum).nTyp=1`:** Builds a `DIM` statement string.
              * **If `varRec(recNum).nTyp=2`:** Builds a `PARAM` statement string.
              * `GOSUB 200` is used to format array names (e.g., `C001()`) into `C001(10)`.
              * `GOSUB 210` is used to append the correct type string (e.g., `:BYTE`, `:STRING[n]`) based on the variable's name prefix.
              * Completed statements are written to the output file using `PRINT #oPath,source`.
        9.  `CLOSE`s `vFile`.
        10. `END`s the procedure.
      * **Subroutines (GOSUB):**
          * `Line 200:` Fixes array variable name suffixes in `varFRec.vp2.vName` using `LEFT$` and `varFRec.vp2.vArray`.
          * `Line 210:` Appends the type declaration (e.g., `:BYTE`, `:REAL`) to the `source` string. It determines the type by checking prefixes in `tName` (e.g., `Bb`, `Ii`, `Ss`) and appends string lengths or user-defined type names.
          * `Line 268:` Prints the `dsatCnt` counter value.
          * `Line 269:` Prints a progress indicator (`.` or the full `cntr` value) based on the `verbose` flag.
      * **Error Handling:**
          * `Line 500:` Captures the `ERR` code. `CLOSE`s both `vPath` and `dPath` if they are open (`vOpen`, `dOpen`). `END`s the procedure, returning the error code in the `er` parameter.

  * **`PROCEDURE dsSort`**

      * **Purpose:** To sort an array of `DSMP` structures (passed by reference) using the Quicksort algorithm. The sort key is the `dsOf` (offset) field.
      * **Parameters (`PARAM`):**
          * `er`: Error code container, passed by reference.
          * `which`: Recursion depth counter, passed by reference.
          * `bottom`, `top`: `INTEGER` bounds for the current sort partition.
          * `DSMaps(100):DSMP`: The array to be sorted.
      * **Main Logic:**
        1.  Sets a local error handler `ON ERROR GOTO 10`.
        2.  Increments `which` and `PRINT`s a `.` to path \#2 (standard error) as a progress indicator.
        3.  Sets `BASE 0` for array indexing.
        4.  Implements a standard Quicksort partitioning `LOOP`. It partitions the array based on the `dsOf` field of the pivot element (`DSMaps(top)`).
        5.  Elements are swapped using the local `DSMap` variable.
        6.  Calls `RUN dsSort` recursively for the lower partition (`bottom` to `lower-1`).
        7.  Calls `RUN dsSort` recursively for the upper partition (`lower+1` to `top`).
        8.  `END`s the procedure.
      * **Subroutines (GOSUB):** None.
      * **Error Handling:**
          * `Line 10:` Captures the `ERR` code in the `er` parameter and `END`s the procedure.
