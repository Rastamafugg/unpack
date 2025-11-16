Analysis complete.

### 1\. Identified Procedures and Logic Blocks

Based on the `INPUT_CODE`, the following logic units have been identified:

  * **Main Procedure:** `PROCEDURE udefVars`
  * **Internal Subroutines (GOSUB):**
      * `Line 200`: Display header/labels for variable references.
      * `Line 250`: Display formatted data for a single variable record.
      * `Line 267`: Display validation counts.
      * `Line 410`: Variable name lookup from `DATA` statements.
      * `Line 500`: Error trap handler.
  * **External Procedures:**
      * `PROCEDURE vSort`: Quicksort algorithm for variable records.
      * `PROCEDURE fSort`: Bubble sort algorithm for field records.

-----

### 2\. Logic Walk-Through

#### A. `PROCEDURE udefVars` (Main Logic)

This procedure is the primary controller. It deconstructs a procedure's variable definitions and symbol table.

1.  **Initialization:**
      * Sets `BASE 0` (array indices start at 0), initializes error flags, and sets the error trap `ON ERROR GOTO 500`.
      * Constructs file paths (`vFile`, `dFile`) from the `dataDir` parameter.
      * The `chainMod` string is commented out (using `!`, equivalent to `REM` per documentation).
2.  **Data Loading:**
      * Opens `dFile` (`procData`) for `READ`.
      * Reads `hasRecords` (BOOLEAN) and the entire `varRec` array using `GET`.
      * Closes `dFile`.
      * Opens `vFile` (`varDefs`) for `UPDATE` (Read/Write).
      * Reads `varCount` (INTEGER) from the start of `vFile`.
      * Seeks to `descOff` (descriptor offset) in the main procedure file (`path`).
3.  **Variable Identification Loop (`REPEAT` UNTIL `recNum=varCount`):**
      * This is the core processing loop. It iterates `varCount` times.
      * It seeks past the `varCount` integer at the start of `vFile` (`recNum*SIZE(varFRec)+2`).
      * It reads one `varFRec` structure from `vFile` using `GET`.
      * It retrieves the variable's type (`iToken`) and offset (`iPointer`) from the pre-loaded `varRec` array.
      * **Type 1 (mTyp=1 - Simple Atomic):**
          * `RESTORE 430` (points `READ` to atomic name `DATA`).
          * `GOSUB 410` (finds and sets `tokenString`).
          * Determines `dmSize` (1, 2, or 5 bytes) based on `tokenCode`.
          * Stores data memory offset (`dmAddr`) and size (`dmSize`).
      * **Type 2 (mTyp=2 - STRING):**
          * Sets `isString:=TRUE`.
          * `RESTORE 430`, `GOSUB 410` (get name).
          * Follows `iPointer` (`dsAddr`) into the main file (`path`) to the DSAT (Data Structure Allocation Table).
          * `GET`s the `dmAddr` (Data Memory address) and `dmSize` (string length) from the DSAT entry.
          * Creates the `strLen` string (e.g., `"[32]"`).
      * **Type 3 (mTyp=3 - VDT Reference):**
          * Follows `iPointer` (`vAddr`) to the VDT (Variable Descriptor Table) in the main file (`symTabOff+vAddr`).
          * `GET`s the `vToken` and `vPointer`.
          * `RESTORE 420` (points `READ` to VDT name `DATA`).
          * `GOSUB 410` (get name).
          * **If `vToken=$A0` (Subroutine):** Reads the full subroutine name.
          * **Else (Variable):** Enters a large `IF`/`THEN`/`ELSE` block to identify the variable's precise type (Simple, Field, Param, String, Record, Array) based on the `vToken` range.
          * It follows the `vPointer` to the DSAT (`dsAddr`), reads the `dsPointer` and `dSize` from the DSAT entry, and, if an array, reads dimension sizes (`element1`, `element2`, `element3`).
      * **Loop End:**
          * `GOSUB 250` (prints verbose output if enabled).
          * `SEEK`s back to the current record position in `vFile`.
          * `PUT`s the (potentially updated) `varFRec` back into `vFile`.
          * Increments `recNum`.
4.  **Sorting:**
      * `RUN vSort`: Calls the external quicksort procedure to sort `vFile` records and the `varRecs` array based on data memory offset (`dmOff`).
      * Counts `fRecs` (field records) by reading `vFile` until the `vdTkn` is out of the field range.
      * `RUN fSort`: Calls the external bubble sort procedure to sort the `fRecs` block at the beginning of the file based on variable offset (`vOff`).
      * `OPEN`s `dFile` for `UPDATE` and `PUT`s the modified `hasRecords` flag and the sorted `varRec` array back to disk.
5.  **Variable Naming:**
      * Loops `varCount` times (`REPEAT`/`UNTIL`).
      * `GET`s each `varFRec`.
      * If `varRec(recNum).nVar` is `TRUE`, it generates a sequential name (e.g., `f001`, `V0001`, `P0001`) based on its type (`nTyp`) and writes it into `varFRec.vp2.vName`.
      * It detects shared memory offsets (`dmOff`, `fOff`) to assign the same name to duplicate references.
      * `PUT`s the updated `varFRec` back to `vFile`.
6.  **Final Display and Validation:**
      * `GOSUB 200` (prints headers).
      * Loops `varCount` times, `GET`s each record, and `GOSUB 250` (prints formatted, sorted data).
      * Validates the Symbol Table by reading the main file's VDT (`SEEK #path,symTabOff`) entry by entry (`GET #path,vToken,vPointer`).
      * For each VDT entry, it searches `vFile` to ensure a "Validated" match exists.
      * Prints final validation status.
7.  **Termination:**
      * `END`s the procedure.

#### B. `PROCEDURE vSort`

1.  **Purpose:** A quicksort algorithm that sorts both an in-memory array (`varRecs`) and a parallel disk file (`vPath`) simultaneously.
2.  **Logic:**
      * Receives `vPath`, `bottom`, `top`, and the `varRecs` array as parameters (`PARAM`).
      * Sets `BASE 0` and error trap `ON ERROR GOTO 10`.
      * Implements a standard quicksort partitioning `LOOP` comparing `varRecs(n).dmOff`.
      * **Swap Logic:** When a swap is required (e.g., for `lower` and `upper`), it:
        1.  `SEEK`s and `GET`s the `lower` record from `vFile` into `varFRecs(0)`.
        2.  `SEEK`s and `GET`s the `upper` record from `vFile` into `varFRecs(1)`.
        3.  Swaps the `varRecs` array elements (`varRecs(lower)` and `varRecs(upper)`).
        4.  Swaps the temporary buffer (`varFRecs(0)` and `varFRecs(1)`).
        5.  `SEEK`s and `PUT`s the new `lower` record (from `varFRecs(0)`) to `vFile`.
        6.  `SEEK`s and `PUT`s the new `upper` record (from `varFRecs(1)`) to `vFile`.
      * **Recursion:** Calls `RUN vSort` for the remaining partitions.

#### C. `PROCEDURE fSort`

1.  **Purpose:** An optimized bubble sort algorithm, also sorting `varRecs` and `vFile` simultaneously.
2.  **Logic:**
      * Receives `vPath` and `varRecs` as parameters.
      * Sets `BASE 0`.
      * Uses a `REPEAT`/`UNTIL` loop.
      * Compares adjacent elements: `varRecs(recCount).vOff > varRecs(recCount+1).vOff`.
      * **Swap Logic:** If a swap is needed:
        1.  Swaps the `varRecs` array elements (`varRecs(recCount)` and `varRecs(recCount+1)`).
        2.  `SEEK`s to `recCount` in `vFile`.
        3.  `GET`s 3 records into `varFRecs` (indices 0, 1, 2).
        4.  Performs an in-memory swap: `varFRec := varFRecs(0)`, `varFRecs(0) := varFRecs(1)`, `varFRecs(1) := varFRec`.
        5.  `SEEK`s back to `recCount`.
        6.  `PUT`s the 3-record `varFRecs` buffer, effectively swapping the first two records on disk.
        7.  Moves `recCount` backward (`ABS(recCount-1)`) to continue bubbling.
      * If no swap, advances `recCount` to the next unsorted pair (`recCount:=sortedInt`).

-----

### 3\. Process and Workflow Synthesis (PWS) Templates

#### A. PWS Template: `udefVars` (Procedure Deconstruction)

```
(* ==================================== *)
(* TEMPLATE: Procedure Deconstruction
(* ==================================== *)

PROCEDURE DeconstructProc
  (* 1. Define all data structures *)
  TYPE VREC=...
  TYPE VFRC=...
  DIM varRec(...):VREC
  DIM varFRec:VFRC

  (* 2. Define parameters and local variables *)
  DIM ...
  PARAM path:BYTE; ...

  (* 3. Initialize Environment *)
  BASE 0
  er:=0
  ON ERROR GOTO 500 (* Trap *)

  (* 4. Load lookup data *)
  OPEN #dPath,dFile:READ
  GET #dPath,lookupFlag
  GET #dPath,lookupArray
  CLOSE #dPath

  (* 5. Open main data file and main subject file *)
  OPEN #vPath,vFile:UPDATE
  GET #vPath,dataCount
  SEEK #path,dataOffset (* 'path' is subject file *)

  (* 6. Main processing loop *)
  PRINT #2,"Starting Process..."
  GOSUB 200 (* Display Headers *)
  recNum:=0
  SEEK #vPath, ... (* Skip header *)
  REPEAT
    (* 6a. Initialize loop state *)
    isTypeA:=FALSE \isTypeB:=FALSE \...
    
    (* 6b. Read data record *)
    GET #vPath,dataRec
    
    (* 6c. Get lookup info *)
    token:=lookupArray(recNum).token
    pointer:=lookupArray(recNum).pointer
    
    (* 6d. Categorize and process *)
    IF dataRec.type = 1 THEN (* Category 1 *)
      RESTORE 430 (* Data for Cat 1 *)
      GOSUB 410 (* Get Name *)
      (* ... process simple data ... *)
      dataRec.dmSize := ...
    ENDIF

    IF dataRec.type = 2 THEN (* Category 2 *)
      RESTORE 430 (* Data for Cat 2 *)
      GOSUB 410 (* Get Name *)
      (* ... follow pointer to secondary table ... *)
      SEEK #path, pointer
      GET #path, dmAddr, dmSize
      (* ... store results ... *)
      dataRec.dmSiz := dmSize
    ENDIF
    
    IF dataRec.type = 3 THEN (* Category 3 *)
      (* ... follow pointer to primary table ... *)
      SEEK #path, symTabOff + pointer
      GET #path, vToken, vPointer
      RESTORE 420 (* Data for Cat 3 *)
      GOSUB 410 (* Get Name *)

      (* 6e. Deep processing of category 3 *)
      IF vToken = $A0 THEN (* Sub-type A *)
        ...
      ELSE (* Sub-type B, C, D *)
        IF vToken >= $40 AND vToken <= $43 THEN
           ...
        ENDIF
        IF vToken >= $48 AND vToken <= $4D THEN (* Array *)
           SEEK #path, ...
           GET #path, element1
           ...
        ENDIF
      ENDIF
    ENDIF
    
    (* 6f. Display status and write back *)
    GOSUB 250 (* Display Record *)
    SEEK #vPath, recNum*SIZE(dataRec)+...
    PUT #vPath,dataRec
    recNum:=recNum+1
  UNTIL recNum=dataCount
  PRINT #2,"Done"

  (* 7. Sort data *)
  PRINT #2,"Sorting..."
  RUN vSort(er,which,vPath,0,dataCount-1,lookupArray)
  IF er>0 THEN 500
  (* ... optional second sort ... *)
  RUN fSort(...)
  PRINT #2,"Done"
  
  (* 8. Update lookup file *)
  OPEN #dPath,dFile:UPDATE
  PUT #dPath,lookupFlag
  PUT #dPath,lookupArray
  CLOSE #dPath
  
  (* 9. Post-process (e.g., Renaming) *)
  PRINT #2,"Post-processing..."
  recNum:=0
  REPEAT
    SEEK #vPath, ...
    GET #vPath,dataRec
    IF lookupArray(recNum).needsName THEN
      (* ... generate name logic ... *)
      dataRec.name := ...
      PUT #vPath,dataRec
    ENDIF
    recNum:=recNum+1
  UNTIL recNum=dataCount
  PRINT #2,"Done"
  
  (* 10. Final display and validation *)
  GOSUB 200 (* Headers *)
  FOR recNum:=0 TO dataCount-1
    GET #vPath,dataRec
    GOSUB 250 (* Display Record *)
  NEXT recNum
  
  (* ... Validation logic ... *)
  
  (* 11. End *)
  END

  (* === SUBROUTINES === *)
200 (* Display Headers *)
  IF verbose THEN
    PRINT #2, "Header..."
  ENDIF
  RETURN

250 (* Display Record *)
  IF verbose THEN
    PRINT #2 USING "...", dataRec.field
  ELSE
    PRINT #2, ".";
  ENDIF
  RETURN

410 (* Name Lookup *)
  REPEAT
    READ byteVal,nameStr
  UNTIL byteVal=token OR byteVal=$A0
  RETURN

420 (* VDT Name Data *)
  DATA $40,"name1",$41,"name2",...
  
430 (* Atomic Name Data *)
  DATA $00,"nameA",$80,"nameB",...

500 (* Error Trap *)
  IF er=0 THEN er:=ERR
  IF vOpen THEN CLOSE #vPath
  IF dOpen THEN CLOSE #dPath
  END
END
```

#### B. PWS Template: `vSort` (Quicksort with Disk Sync)

```
(* ==================================== *)
(* TEMPLATE: Quicksort (In-Memory Array + Disk File)
(* ==================================== *)

PROCEDURE vSort
  (* 1. Define types and parameters *)
  TYPE VREC=...
  TYPE VFRC=...
  DIM varRec:VREC
  DIM varFRec,varFRecs(2):VFRC
  PARAM er:INTEGER; vPath:BYTE; bottom,top:INTEGER; varRecs(...):VREC
  DIM lower,upper:INTEGER

  (* 2. Set environment *)
  ON ERROR GOTO 10 (* Error Trap *)
  BASE 0

  (* 3. Partitioning loop *)
  lower:=bottom
  upper:=top
  LOOP
    (* 3a. Find lower element to swap *)
    REPEAT
      btemp := varRecs(lower).sortKey < varRecs(top).sortKey
      lower:=lower+1
    UNTIL NOT(btemp)
    lower:=lower-1
    EXITIF lower=upper THEN ENDEXIT

    (* 3b. Find upper element to swap *)
    REPEAT
      upper:=upper-1
    UNTIL varRecs(upper).sortKey <= varRecs(top).sortKey OR upper=lower
    EXITIF lower=upper THEN ENDEXIT

    (* 3c. Perform synchronized swap *)
    (* Read records from disk *)
    SEEK #vPath,lower*SIZE(varFRec)+...
    GET #vPath,varFRec
    varFRecs(0):=varFRec
    SEEK #vPath,upper*SIZE(varFRec)+...
    GET #vPath,varFRec
    varFRecs(1):=varFRec
    (* Swap in-memory array *)
    varRec:=varRecs(lower)
    varRecs(lower):=varRecs(upper)
    varRecs(upper):=varRec
    (* Swap buffer *)
    varFRec:=varFRecs(0)
    varFRecs(0):=varFRecs(1)
    varFRecs(1):=varFRec
    (* Write records to disk *)
    SEEK #vPath,lower*SIZE(varFRec)+...
    varFRec:=varFRecs(0)
    PUT #vPath,varFRec
    SEEK #vPath,upper*SIZE(varFRec)+...
    varFRec:=varFRecs(1)
    PUT #vPath,varFRec

    lower:=lower+1
    EXITIF lower=upper THEN ENDEXIT
  ENDLOOP

  (* 4. Swap pivot element *)
  IF lower<>top THEN
    IF varRecs(lower).sortKey<>varRecs(top).sortKey THEN
      (* ... perform synchronized swap (lower, top) ... *)
    ENDIF
  ENDIF

  (* 5. Recursive calls *)
  IF bottom<lower-1 THEN
    RUN vSort(er,vPath,bottom,lower-1,varRecs)
  ENDIF
  IF lower+1<top THEN
    RUN vSort(er,vPath,lower+1,top,varRecs)
  ENDIF

  END

10 (* Error Trap *)
  er:=ERR
  END
END
```

#### C. PWS Template: `fSort` (Bubble Sort with Disk Sync)

```
(* ==================================== *)
(* TEMPLATE: Bubble Sort (In-Memory Array + Disk File)
(* ==================================== *)

PROCEDURE fSort
  (* 1. Define types and parameters *)
  TYPE VREC=...
  TYPE VFRC=...
  DIM varRec:VREC
  DIM varFRec,varFRecs(2):VFRC (* 3-element buffer for swap *)
  PARAM vPath:BYTE; recCount,varCount:INTEGER; varRecs(...):VREC
  DIM sortedInt:INTEGER

  (* 2. Set environment *)
  BASE 0

  (* 3. Sorting loop *)
  sortedInt:=0
  REPEAT
    (* 3a. Check element type (optional filter) *)
    SEEK #vPath,recCount*SIZE(varFRec)+...
    GET #vPath,varFRec (* Read 1 record just to check token *)
    IF varFRec.vp1.vdTkn > $3F AND varFRec.vp1.vdTkn < $60 THEN
      
      (* 3b. Compare adjacent elements *)
      IF varRecs(recCount).sortKey > varRecs(recCount+1).sortKey THEN
        (* 3c. Perform synchronized swap *)
        (* Swap in-memory array *)
        varRec:=varRecs(recCount)
        varRecs(recCount):=varRecs(recCount+1)
        varRecs(recCount+1):=varRec
        (* Read 3 records from disk (n, n+1, n+2) *)
        SEEK #vPath,recCount*SIZE(varFRec)+...
        GET #vPath,varFRecs (* Reads into varFRecs(0,1,2) *)
        (* Swap buffer (n, n+1) *)
        varFRec:=varFRecs(0)
        varFRecs(0):=varFRecs(1)
        varFRecs(1):=varFRec
        (* Write 3 records back to disk *)
        SEEK #vPath,recCount*SIZE(varFRec)+...
        PUT #vPath,varFRecs
        
        (* 3d. Bubble: move index back *)
        recCount:=ABS(recCount-1)
      ELSE
        (* 3e. Advance: move to next unsorted pair *)
        recCount:=sortedInt
        sortedInt:=sortedInt+1
      ENDIF
    ENDIF
  UNTIL recCount=varCount-1
  END
END
```