**Code Maintenance Report**

### 1. Procedure & Data Analysis

Analysis of all procedures, their local data, and shared data structures.

**User-Defined Data (`TYPE`)**

* `TYPE VREC`: Structure (defined in `udecode`) holding variable type (`vTyp`, `nTyp`), flags (`nVar`), and various integer offsets (`vOff`, `vdOff`, `dsOff`, `dmOff`, `fOff`, `pOff`).
* `TYPE VFP1`: Structure (defined in `udecode`) holding integer-based file pointer and size information.
* `TYPE VFP2`: Structure (defined in `udecode`) holding string-based variable names/lengths and integer-based reference counters.
* `TYPE VFP3`: Structure (defined in `udecode`) holding byte/boolean flags for type, linking, and record status.
* `TYPE VFRC`: Complex structure (defined in `udecode`) combining `VFP1`, `VFP2`, and `VFP3` into a single variable file record.
* `TYPE LNREF`: Structure (defined in `udecode` and `lSort`) holding a line reference type (`hLTyp`) and its offset (`hLOff`).
* `TYPE LNREC`: Structure (defined in `udecode` and `lSort`) holding full line reference data, including counts, type, offset, and assigned line number string.
* `TYPE DSMP`: Structure (defined in `udecode`) holding DSAT map data, including a `REAL` offset and integer dimensions/sizes.

**Procedure Interface & Storage**

* **`PROCEDURE udecode`**
    * **Parameters (`PARAM`):** Variables passed by reference.
        * `path`: Input, file path number for I-Code file, type `BYTE`
        * `er`: Input/Output, error code container, type `INTEGER`
        * `tpVars`: Input/Output, total program variables, type `INTEGER`
        * `pOpen`: Input, flag (procedure open status), type `BOOLEAN`
        * `verbose`: Input, flag for verbose output, type `BOOLEAN`
        * `execOff`: Input, I-Code execution offset, type `REAL`
        * `descOff`: Input, I-Code descriptor offset, type `REAL`
        * `symTabOff`: Input, I-Code symbol table offset, type `REAL`
        * `dataDir`: Input, data directory path, type `STRING[16]`
        * `modName`: Input, module name, type `STRING[29]`
    * **Local Variables (`DIM`):**
        * `varRec(400)`: Array of `VREC` (Variable record cache)
        * `varFRec`, `defFRec`: Variables of `VFRC` (Working/default variable file records)
        * `linRefs(300)`: Array of `LNREF` (Line reference cache)
        * `linRec`: Variable of `LNREC` (Working line record)
        * `DSMap(100)`: Array of `DSMP` (DSAT map)
        * `tokenVal`, `byteVal`, `linkVal`: I-Code parser values, type `BYTE`
        * `loopCnt`, `nextRef`, `recCount`, `tInstr`: Counters, type `INTEGER`
        * `quote`, `modCnt`, `intVal`, `tmpTyp`, `intOrReal`: I-Code parser values/state, type `INTEGER`
        * `realVal`: I-Code parser value, type `REAL`
        * `found`, `nextVar`, `forVar`, `namVar`, `creop`, `isOn`, `isRestore`, `renum`, `potRec`, `isField`, `extRef`, `hasRecords`: State flags, type `BOOLEAN`
        * `varRefs`, `varCount`: Variable reference counters, type `INTEGER`
        * `lineRefs`, `linCount`, `lnCnt`: Line reference counters, type `INTEGER`
        * `lastOff`: Line numbering state, type `INTEGER`
        * `lastNum`: Line numbering state, type `STRING[5]`
        * `vPath`, `lnPath`, `dPath`: File path numbers, type `BYTE`
        * `vOpen`, `lnOpen`, `dOpen`: File open flags, type `BOOLEAN`
        * `which`, `dataStAdd`: State/offset variables, type `INTEGER`
        * `vFile`, `lnFile`, `dFile`: File names, type `STRING[29]`
* **`PROCEDURE lSort`**
    * **Parameters (`PARAM`):**
        * `er`: Input/Output, error code container, type `INTEGER`
        * `which`: Input/Output, recursion depth counter, type `INTEGER`
        * `lnPath`: Input, file path number for line definitions, type `BYTE`
        * `bottom`: Input, sort partition boundary, type `INTEGER`
        * `top`: Input, sort partition boundary, type `INTEGER`
        * `linRefs(300)`: Input/Output, array of line references to be sorted, type `LNREF`
    * **Local Variables (`DIM`):**
        * `linRef`: Temporary variable for `LNREF` swap
        * `linRec`: Working variable of `LNREC`
        * `linRecs(2)`: Array of `LNREC` for disk record swapping
        * `lower`, `upper`: Sort partition indices, type `INTEGER`
        * `btemp`: Comparison result flag, type `BOOLEAN`

---

### 2. Call Graph (RUN & GOSUB)

A map of inter-procedure (`RUN`) and intra-procedure (`GOSUB`) dependencies.

* **`PROCEDURE udecode`**
    * `RUN lSort(er,which,lnPath,0,linCount-1,linRefs)`: (from `GOSUB 300`) Calls the quicksort procedure to sort the line reference file.
    * `GOSUB 200`: (Decode Instructions) The main I-Code parsing loop.
    * `GOSUB 210`: (from `GOSUB 200`) (Identify Duplicate and Add Variable Record) Logic to add/update a variable reference.
    * `GOSUB 220`: (from `GOSUB 200`) (Identify Duplicate and Add Line Record) Logic to add/update a line reference.
    * `GOSUB 300`: (Sort Line References) Sorts and numbers the collected line references.
    * `GOSUB 400`: (Get Data from initDB) Loads initial data from the "initDB" file.
* **`PROCEDURE lSort`**
    * `RUN lSort(er,which,lnPath,bottom,lower-1,linRefs)`: Recursive call to sort the lower partition.
    * `RUN lSort(er,which,lnPath,lower+1,top,linRefs)`: Recursive call to sort the upper partition.
    * No `GOSUB` calls.

---

### 3. External Interaction Points

All statements that interact with the operating system, files, or hardware.

* **File I/O:**
    * `PROCEDURE udecode`: `OPEN #vPath,vFile:UPDATE` (Opens varDefs file)
    * `PROCEDURE udecode`: `OPEN #lnPath,lnFile:UPDATE` (Opens linDefs file)
    * `PROCEDURE udecode`: `SEEK #path,symTabOff-3` (Positions I-Code file)
    * `PROCEDURE udecode`: `GET #path,tpVars` (Reads from I-Code file)
    * `PROCEDURE udecode`: `PRINT #2, ...` (Prints to Standard Error/Status path)
    * `PROCEDURE udecode`: `CLOSE #vPath`
    * `PROCEDURE udecode`: `CLOSE #lnPath`
    * `PROCEDURE udecode`: `OPEN #dPath,dFile:WRITE` (Opens procData file)
    * `PROCEDURE udecode`: `PUT #dPath,hasRecords` (Writes to procData)
    * `PROCEDURE udecode`: `PUT #dPath,varRec` (Writes to procData)
    * `PROCEDURE udecode`: `PUT #dPath,varRefs` (Writes to procData)
    * `PROCEDURE udecode`: `PUT #dPath,varCount` (Writes to procData)
    * `PROCEDURE udecode`: `PUT #dPath,lineRefs` (Writes to procData)
    * `PROCEDURE udecode`: `PUT #dPath,linCount` (Writes to procData)
    * `PROCEDURE udecode`: `CLOSE #dPath`
    * `PROCEDURE udecode` (Line 200): `SEEK #path,execOff` (Positions I-Code file)
    * `PROCEDURE udecode` (Line 200): `GET #path,tokenVal` (Reads I-Code token)
    * `PROCEDURE udecode` (Line 200): `GET #path,intVal` (Reads I-Code integer)
    * `PROCEDURE udecode` (Line 200): `GET #path,realVal` (Reads I-Code real)
    * `PROCEDURE udecode` (Line 200): `GET #path,byteVal` (Reads I-Code byte/string)
    * `PROCEDURE udecode` (Line 210): `SEEK #vPath,...` (Positions varDefs file)
    * `PROCEDURE udecode` (Line 210): `GET #vPath,varFRec` (Reads from varDefs)
    * `PROCEDURE udecode` (Line 210): `PUT #vPath,varFRec` (Writes to varDefs)
    * `PROCEDURE udecode` (Line 210): `SEEK #vPath,0` (Rewinds varDefs)
    * `PROCEDURE udecode` (Line 210): `PUT #vPath,varCount` (Writes header to varDefs)
    * `PROCEDURE udecode` (Line 220): `SEEK #lnPath,...` (Positions linDefs file)
    * `PROCEDURE udecode` (Line 220): `GET #lnPath,linRec` (Reads from linDefs)
    * `PROCEDURE udecode` (Line 220): `PUT #lnPath,linRec` (Writes to linDefs)
    * `PROCEDURE udecode` (Line 220): `SEEK #lnPath,0` (Rewinds linDefs)
    * `PROCEDURE udecode` (Line 220): `PUT #lnPath,linCount` (Writes header to linDefs)
    * `PROCEDURE udecode` (Line 300): `SEEK #lnPath,0` (Rewinds linDefs)
    * `PROCEDURE udecode` (Line 300): `SEEK #lnPath,2` (Positions linDefs)
    * `PROCEDURE udecode` (Line 300): `GET #lnPath,linRec` (Reads from linDefs)
    * `PROCEDURE udecode` (Line 300): `PUT #lnPath,linRec` (Writes to linDefs)
    * `PROCEDURE udecode` (Line 400): `OPEN #dPath,dFile:READ` (Opens initDB file)
    * `PROCEDURE udecode` (Line 400): `GET #dPath,version` (Reads from initDB)
    * `PROCEDURE udecode` (Line 400): `GET #dPath,varFRec` (Reads from initDB)
    * `PROCEDURE udecode` (Line 400): `GET #dPath,DSMap` (Reads from initDB)
    * `PROCEDURE udecode` (Line 400): `CLOSE #dPath`
    * `PROCEDURE udecode` (Line 500): `CLOSE` statements for `#vPath`, `#lnPath`, `#dPath` in error handler.
    * `PROCEDURE lSort`: `PRINT #2,"."` (Prints to Standard Error/Status path)
    * `PROCEDURE lSort`: `SEEK #lnPath,...` (Positions linDefs file for swapping)
    * `PROCEDURE lSort`: `GET #lnPath,linRec` (Reads from linDefs for swapping)
    * `PROCEDURE lSort`: `PUT #lnPath,linRec` (Writes to linDefs for swapping)
* **System/Shell I/O:**
    * None. (Note: The line `! CHAIN chainMod` in `udecode` is a comment, as `!` is a `REM` equivalent per the documentation).
* **Memory/Hardware I/O:**
    * None.

---

### 4. Logic Walk-Through

A step-by-step explanation of each procedure's execution flow.

* **`PROCEDURE udecode`**
    * **Purpose:** The main procedure to parse a Basic09 I-Code file, extract all variable and line number references, and save this cross-reference data to a set of output files.
    * **Parameters (`PARAM`):** Receives the I-Code file path (`path`), offsets (`execOff`, `descOff`, `symTabOff`), flags (`verbose`, `pOpen`), and name/path strings (`dataDir`, `modName`) from a calling procedure.
    * **Main Logic:**
        1.  Sets `BASE 0` for array indexing.
        2.  Initializes all file open flags (`vOpen`, `lnOpen`, `dOpen`) to `FALSE`.
        3.  Sets the procedure-local error handler `ON ERROR GOTO 500`.
        4.  Sets the `dFile` path and calls `GOSUB 400` to load initialization data from "initDB".
        5.  Initializes numerous local flags (e.g., `creop`, `isOn`) and counters (e.g., `varRefs`, `linCount`) to zero/`FALSE`.
        6.  `OPEN`s the "varDefs" and "linDefs" files in `UPDATE` mode, setting their flags to `TRUE`.
        7.  `SEEK`s into the I-Code file (`#path`) using `symTabOff` and `GET`s the `tpVars` (Total Program Variables).
        8.  Calls `GOSUB 200`, which performs the primary I-Code parsing.
        9.  After parsing, calls `GOSUB 300` to sort and number the line references collected by the previous step.
        10. `PRINT`s final summary counts to Standard Error (`#2`).
        11. `CLOSE`s the `#vPath` and `#lnPath` files.
        12. `OPEN`s the "procData" file (`#dPath`) in `WRITE` mode.
        13. `PUT`s the `hasRecords` flag, the entire `varRec` array, and all counters (`varRefs`, `varCount`, etc.) to the `#dPath` file.
        14. `CLOSE`s the `#dPath` file and `END`s the procedure.
    * **Subroutines (GOSUB):**
        * `Line 200:` (Instruction Decode) `SEEK`s to the I-Code start (`execOff`). `REPEAT`s a loop, `GET`ting one `tokenVal` at a time. It uses a state machine to interpret tokens, identify variable/line references, and skip literals. Calls `GOSUB 210` for variables and `GOSUB 220` for lines. The loop terminates `UNTIL` the entire I-Code block is read (`modCnt=descOff-execOff`).
        * `Line 210:` (Add Variable Record) Manages the "varDefs" file. It searches the `varRec` cache for the variable. If not `found`, it creates a new `varFRec`, adds it to `varRec`, increments `varCount`, and `PUT`s the new record to `#vPath`, updating the file header. If `found`, it `GET`s the record, increments `vRefCnt`, and `PUT`s the updated record.
        * `Line 220:` (Add Line Record) Manages the "linDefs" file. It searches the `linRefs` cache. If not `found`, it adds to `linRefs`, increments `linCount`, creates a `linRec`, and `PUT`s the new record to `#lnPath`, updating the file header. If `found`, it `GET`s the record, increments `lnRefCnt`, and `PUT`s the update.
        * `Line 300:` (Sort/Number Lines) If line references were found, it calls `RUN lSort` to sort the `linRefs` array *and* the on-disk `#lnPath` file simultaneously. It then re-reads the sorted file, assigning sequential line numbers (e.g., "10", "20") to unique offsets, and `PUT`s the updated records back.
        * `Line 400:` (Get Data) `OPEN`s, `GET`s data from ("initDB"), and `CLOSE`s the file, populating `version`, `defFRec`, and `DSMap`.
    * **Error Handling:**
        * `Line 500:` Traps all runtime errors. It checks the `ERR` code. In all cases, it ensures all opened files (`#vPath`, `#lnPath`, `#dPath`) are `CLOSE`d before `END`ing the procedure. It specifically checks for error 216 (GOTO 20, a retry) and 56.

* **`PROCEDURE lSort`**
    * **Purpose:** An in-place (on-disk) quicksort algorithm. It sorts the `linRefs` array in memory while simultaneously applying the same swaps to the records in the file specified by `lnPath`.
    * **Parameters (`PARAM`):**
        * `er`, `which`: Error variable and recursion counter.
        * `lnPath`: The file path number for the "linDefs" file.
        * `bottom`, `top`: The partition boundaries for this recursive call.
        * `linRefs(300)`: The in-memory array used for comparison.
    * **Main Logic:**
        1.  Sets `BASE 0` and the local error handler `ON ERROR GOTO 10`.
        2.  Selects a pivot (the `top` element) and partitions the `linRefs` array using `lower` and `upper` indices.
        3.  When a swap is required, it swaps the elements in the `linRefs` array.
        4.  Simultaneously, it `SEEK`s to the disk locations for `lower` and `upper`, `GET`s both records into `linRecs(0)` and `linRecs(1)`, swaps them, and `PUT`s them back in their new positions.
        5.  `RUN`s itself recursively for the sub-partitions (`bottom` to `lower-1` and `lower+1` to `top`).
    * **Subroutines (GOSUB):**
        * None.
    * **Error Handling:**
        * `Line 10:` Catches a runtime error, assigns the `ERR` code to the `er` parameter (passing it back to `udecode`), and `END`s the procedure.