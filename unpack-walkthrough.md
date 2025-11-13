### 1\. Procedure Identification

**Main Logic:**

  * `PROCEDURE unpack`

**Internal Procedures (GOSUB/RETURN):**

  * **Label 30:** Turn Pause On
  * **Label 40:** Turn Pause Off
  * **Label 50:** Turn Cursor Off (via external `GFX2`)
  * **Label 60:** Turn Cursor On (via external `GFX2`)

**Error Handlers (GOTO):**

  * **Label 10:** Skip option parsing
  * **Label 20:** Display Help Screen
  * **Label 100:** Primary Error Trap

-----

### 2\. Logic Walk-Through

A deconstruction of the main logic flow and internal procedures, validating syntax against the provided documentation.

#### Main Logic (`PROCEDURE unpack`)

1.  **Declarations (Lines 2-90):**

      * The code uses `TYPE` to define three complex data structures: `HEAD`, `OHEAD`, and `REG`. (Valid: "TYPE Statement")
      * It uses `DIM` to declare numerous variables, specifying atomic types (`BYTE`, `INTEGER`, `REAL`, `BOOLEAN`) and `STRING` with explicit lengths. (Valid: "DIM Statement")
      * It uses `PARAM` to define the procedure's input parameters, `infile` and `option`. (Valid: "PARAM Statement")
      * All comments use the `(* ... *)` syntax. (Valid: "Comment Statements")

2.  **Initialization (Lines 94-106):**

      * Variables are assigned default values using `LET` (implied). (Valid: "Assignment Statements")
      * Boolean flags (`nono`, `fileopen`, `verbose`, etc.) are set.

3.  **Parameter Parsing (Lines 109-120):**

      * An error trap is set: `ON ERROR GOTO 10`. (Valid: "ON ERROR GOTO Statement")
      * The `option` string is checked using `SUBSTR` for "-V" or "-O" (and lowercase variants). (Valid: "SUBSTR" function, "IF Statement: Type 2")
      * If found, `verbose` and `outfile` flags are updated.

4.  **File Input Parsing (Label 10, Lines 123-134):**

      * A new error trap is set: `ON ERROR GOTO 20`.
      * The `infile` parameter is assigned to the `file` variable.
      * The code attempts to parse a `filename` from the `file` pathlist using a `REPEAT...UNTIL` loop that searches backward for a "/" character, utilizing `LEN`, `MID$`, and `RIGHT$`. (Valid: "REPEAT/UNTIL Statement", "LEN", "MID$", "RIGHT$" functions)
      * It checks if the `file` string starts with "-" or "?" using `SUBSTR` and, if so, branches to Label 20 using `IF...THEN <line #>`. (Valid: "IF Statement: Type 1")

5.  **Setup & Pre-flight (Lines 137-160):**

      * The main error trap is set: `ON ERROR GOTO 100`.
      * The input `file` is opened using `OPEN #filepath,file:READ`. (Valid: "OPEN Statement")
      * The code calls an external procedure `RUN SysCall` to get the OS-9 pause status. It passes parameters by reference (`regs`) and uses `ADDR` to get the variable address. (Valid: "RUN Statement", "ADDR" function)
      * If pause is found to be on, it calls the internal procedure `GOSUB 40` (Turn Pause Off). (Valid: "GOSUB Statement")
      * It calls `GOSUB 50` (Turn Cursor Off).
      * It calls `RUN SysCall` again to get the `filesize`.

6.  **Main Processing Loop (Lines 163-305):**

      * A `WHILE NOT(EOF(#filepath)) DO ... ENDWHILE` loop iterates as long as the end of the input file is not reached. (Valid: "WHILE/DO Statement", "EOF" function)
      * An `EXITIF start=filesize THEN ... ENDEXIT` check provides a clean exit. (Valid: "EXITIF and ENDEXIT Statements")
      * Inside the loop:
          * `SEEK #filepath,start` positions the file pointer. (Valid: "SEEK Statement")
          * `GET #filepath,header` reads binary data directly into the `header` structure. (Valid: "GET Statement")
          * Header fields (`ms`, `eo`, `pss`, `dso`) are extracted.
          * **Module Validation:** It checks `header.sb` (sync bytes). If not `$87CD`, it prints an error to path \#2 (`PRINT #2,...`), calls `GOSUB 30` (Turn Pause On), and `END`s. (Valid: "PRINT Statement", "END Statement")
          * **Name Extraction:** The module name is parsed using `LAND`, `ASC`, `MID$`, `LEFT$`, and `CHR$`. (Valid: "LAND", "ASC", "MID$", "LEFT$", "CHR$" functions)
          * **Type Check:** The code checks `header.tl` and `header.hp` to identify I-Code.
          * If it is not I-Code (`header.tl<>$22`), it reads the `oheader` structure, parses the name using a `REPEAT...UNTIL` loop, prints a "NOT I-CODE\!" message, sets `skipped:=TRUE`, and updates the `start` pointer to skip the module.
          * **I-Code Processing:** If `NOT(skipped)`:
              * It creates an output file: `CREATE #outpath,outname:WRITE`. (Valid: "CREATE statement")
              * It `PRINT`s the `PROCEDURE` line to standard output (default path \#1) and the output file (path `#outpath`).
              * **External Modules:** It calls `RUN part(...)` three times (for "udecode", "udefVars", and "ubuildSrc"), executing `KILL part` and `SHELL "UnLink ..."` after each. (Valid: "RUN", "KILL Statement", "SHELL Statement")
              * **Commented Code:** Lines prefixed with `!` are comments, per the documentation ("The '\!' character can be typed in place of the keyword REM"). The entire "Load part 4" section (lines 262-269) is commented out.
              * **Footer:** It dynamically builds a format string for `PRINT USING` using `STR$` and `LEN`. (Valid: "PRINT USING Statement", "STR$", "LEN")
              * It closes the output file: `CLOSE #outpath`. (Valid: "CLOSE Statement")
              * It updates the `start` pointer for the next loop iteration.

7.  **Program Termination (Lines 307-329):**

      * The `WHILE` loop ends.
      * `CLOSE #filepath` is called.
      * `GOSUB 60` (Turn Cursor On) is called.
      * **Commented Code:** The `!IF er>0...` block (lines 314-329) is commented out.
      * If `ppause` was `TRUE` at the start, `GOSUB 30` (Turn Pause On) is called to restore the state.
      * `END` terminates the procedure.

#### Help Screen (Label 20)

  * This block `PRINT`s help text to path \#2 (Standard Error). (Valid: "PRINT Statement", "I/O PATHS")
  * It ensures the file is closed (`CLOSE #filepath`) and terminates with `END`.

#### Internal Procedure: Turn Pause On (Label 30)

  * This is a `GOSUB` subroutine ending in `RETURN`. (Valid: "GOSUB/RETURN Statements")
  * It modifies the `packet` array, sets `regs` values, and calls `RUN SysCall` to set the OS-9 pause state.

#### Internal Procedure: Turn Pause Off (Label 40)

  * Identical to Label 30, but sets `packet(8):=0` before calling `RUN SysCall`.

#### Internal Procedure: Cursor Off (Label 50)

  * This subroutine calls an external procedure `RUN GFX2("CUROFF")` and `RETURN`s.

#### Internal Procedure: Cursor On (Label 60)

  * This subroutine calls an external procedure `RUN GFX2("CURON")` and `RETURN`s.

#### Primary Error Trap (Label 100)

  * This block is reached via `ON ERROR GOTO 100`.
  * It captures the error code using `er:=ERR` if `er` is not already set. (Valid: "ERR" function)
  * It handles specific error codes (216, 218) with custom `PRINT #2` messages.
  * It cleans up by closing `filepath`, printing the `er` number, calling `GOSUB 60` (Turn Cursor On), and `END`ing.

-----

### 3\. Process and Workflow Synthesis (PWS) Templates

#### PWS Template: Turn Pause On (Label 30)

```basic
(* (PWS Template: Turn Pause On) *)
(*
(* 1. PURPOSE: Calls an external procedure (SysCall) to enable OS-9 page pause. *)
(* 2. USES: packet, callcode, regs *)
(* 3. VALIDATION (from Basic09-only-reference.md): *)
(* - Assignment: LET (implied) *)
(* - Constants: $8E (Hexadecimal) *)
(* - Functions: ADDR() *)
(* - Statements: RUN, RETURN *)
(*
DIM packet(32):BYTE
DIM callcode:BYTE
TYPE REG=cc,a,b,dp:BYTE; x,y,u:INTEGER
DIM regs:REG
*)

30 packet(8):=1
callcode:=$8E
regs.a:=2
regs.b:=$00
regs.x:=ADDR(packet)
RUN SysCall(callcode,regs)
RETURN
```

#### PWS Template: Turn Pause Off (Label 40)

```basic
(* (PWS Template: Turn Pause Off) *)
(*
(* 1. PURPOSE: Calls an external procedure (SysCall) to disable OS-9 page pause. *)
(* 2. USES: packet, callcode, regs *)
(* 3. VALIDATION (from Basic09-only-reference.md): *)
(* - Assignment: LET (implied) *)
(* - Constants: $8E (Hexadecimal) *)
(* - Functions: ADDR() *)
(* - Statements: RUN, RETURN *)
(*
DIM packet(32):BYTE
DIM callcode:BYTE
TYPE REG=cc,a,b,dp:BYTE; x,y,u:INTEGER
DIM regs:REG
*)

40 packet(8):=0
callcode:=$8E
regs.a:=2
regs.b:=$00
regs.x:=ADDR(packet)
RUN SysCall(callcode,regs)
RETURN
```

#### PWS Template: Cursor Off (Label 50)

```basic
(* (PWS Template: Cursor Off) *)
(*
(* 1. PURPOSE: Calls an external procedure (GFX2) to turn the terminal cursor off. *)
(* 2. USES: (None) *)
(* 3. VALIDATION (from Basic09-only-reference.md): *)
(* - Statements: RUN, RETURN *)
(* - Constants: "CUROFF" (String) *)
*)

50 RUN GFX2("CUROFF")
RETURN
```

#### PWS Template: Cursor On (Label 60)

```basic
(* (PWS Template: Cursor On) *)
(*
(* 1. PURPOSE: Calls an external procedure (GFX2) to turn the terminal cursor on. *)
(* 2. USES: (None) *)
(* 3. VALIDATION (from Basic09-only-reference.md): *)
(* - Statements: RUN, RETURN *)
(* - Constants: "CURON" (String) *)
*)

60 RUN GFX2("CURON")
RETURN
```