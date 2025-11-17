**Code Maintenance Report**

### 1. Procedure & Data Analysis

Analysis of all procedures, their local data, and shared data structures.

**User-Defined Data (`TYPE`)**

* `TYPE VREC`: Defines the "Variable Array Record" structure. Contains `BYTE`, `BOOLEAN`, and `INTEGER` fields for variable lookup.
* `TYPE VFP1`: First part of the "Variable File Record". Contains `BYTE` and `INTEGER` fields.
* `TYPE VFP2`: Second part of the "Variable File Record". Contains `STRING`, `INTEGER`, and `BYTE` fields.
* `TYPE VFP3`: Third part of the "Variable File Record". Contains `BYTE` and `BOOLEAN` fields.
* `TYPE VFRC`: The complete "Variable File Record," a composite structure combining `VFP1`, `VFP2`, and `VFP3`.
* `TYPE LNREC`: Defines the "Line Reference Record". Contains `INTEGER`, `BYTE`, and `STRING` fields for line number lookup.

**Procedure Interface & Storage**

* **`PROCEDURE uinstruction`**
    * **Parameters (`PARAM`):** Variables passed by reference.
        * `filePath`: Input I-code file path number, type `BYTE`
        * `outPath`: Output source file path number, type `BYTE`
        * `er`: Error code container, type `INTEGER`
        * `verbose`: Flag for extra console output, type `BOOLEAN`
        * `execOff`: File offset for executable code, type `REAL`
        * `descOff`: File offset for description block, type `REAL`
        * `dataOff`: File offset for data block, type `REAL`
        * `dataDir`: Directory path for data files, type `STRING[16]`
    * **Local Variables (`DIM`):**
        * `varRec(400):VREC`: Array holding the variable lookup table.
        * `varFRec:VFRC`: Record for reading from the variable definitions file.
        * `linRec:LNREC`: Record for reading from the line definitions file.
        * `found,nextVar,forVar,creop,hasRecords,hasLines,isRestore:BOOLEAN`: State flags for the parsing loop.
        * `tokenCode,tokenVal,byteVal:BYTE`: Variables for reading single-byte tokens/values.
        * `modCnt,intVal,nextRef:INTEGER`: Counters and variables for reading multi-byte values.
        * `varCount,varCnt:INTEGER`: Counters for variable loop.
        * `linCount,lrCnt,lnCnt,lastOff:INTEGER`: Counters and offset for line number loop.
        * `filCnt,realVal:REAL`: File counter and variable for `REAL` literals.
        * `vPath,lnPath,dPath:BYTE`: Path numbers for data files.
        * `vOpen,lnOpen,dOpen:BOOLEAN`: File open status flags.
        * `version:STRING[8]`: Version string from init file.
        * `vFile,lnFile,dFile:STRING[29]`: File names for data files.
        * `hex:STRING[4]`: String to hold hexadecimal conversion.
        * `lastNum:STRING[5]`: String for the last read line number.
        * `tokenName:STRING[29]`: String name of a token.
        * `source:STRING[512]`: String buffer for reconstructing a line of code.

* **`PROCEDURE uhex$`**
    * **Parameters (`PARAM`):**
        * `intVal`: Input value to convert, type `INTEGER`
        * `hex`: Result container, type `STRING[4]`
    * **Local Variables (`DIM`):**
        * `index,hex_index,bit,bit_result,hex_result:INTEGER`: Counters and temporary values for bitwise conversion.
        * `hex_bit:STRING[1]`: Temporary storage for a single hex character.

* **`PROCEDURE uDRPN`**
    * **Parameters (`PARAM`):**
        * `inline`: The line to be parsed, type `STRING[512]`
    * **Local Variables (`DIM`):**
        * `templine(10):STRING[400]`: Array used as a stack for parsing.
        * `place,length,count,cntr,cnt:INTEGER`: Loop counters and string position pointers.
        * `parse:BOOLEAN`: Flag to indicate if parsing is required.
        * `char:STRING[1]`: Temporary storage for a single character.
        * `outline:STRING[512]`: String buffer for building the parsed line.

### 2. Call Graph (RUN & GOSUB)

A map of inter-procedure (`RUN`) and intra-procedure (`GOSUB`) dependencies.

* **`PROCEDURE uinstruction`**
    * `RUN uhex$(intVal,hex)`: Converts an integer literal (tokenVal=$91) to its hexadecimal string representation.
    * `RUN uDRPN(source)`: Parses the reconstructed `source` line to convert expressions from postfix (RPN) to infix notation.
    * `GOSUB 210`: (Find Variable Reference) Looks up a variable's name from `varDefs` file using its token type and offset.
    * `GOSUB 230`: (Find Line Number) Looks up a line number string from `linDefs` file using its file offset.
    * `GOSUB 270`: (Read Keyword Tokens) Reads internal `DATA` statements (at line 275) to match a `tokenVal` (e.g., `$26`) with its string name (e.g., "PRINT #2, ").

* **`PROCEDURE uhex$`**
    * No `RUN` calls.
    * `GOSUB 2`: (Read hex intVal character) Converts a 4-bit integer value (0-15) into its corresponding ASCII hex character ('0'-'F') and prepends it to the `hex` string.

* **`PROCEDURE uDRPN`**
    * No `RUN` calls.
    * `GOSUB 1`: (Strip delimiters) Removes all "" characters from the `inline` string.
    * `GOSUB 2`: (Parse line) Initiates the main RPN parsing logic, filling the `templine` stack.
    * `GOSUB 7`: (Adjust templines) Compacts the `templine` array by shifting elements up to fill empty slots.
    * `GOSUB 8`: (Cleanup) Performs final string cleanup (e.g., fixing spacing, removing `.` from `.=`).

### 3. External Interaction Points

All statements that interact with the operating system, files, or hardware.

* **File I/O:**
    * `PROCEDURE uinstruction`: `OPEN #dPath,dFile:READ` (Opens "initDB")
    * `PROCEDURE uinstruction`: `GET #dPath,version`
    * `PROCEDURE uinstruction`: `CLOSE #dPath`
    * `PROCEDURE uinstruction`: `OPEN #dPath,dFile:READ` (Opens "procData")
    * `PROCEDURE uinstruction`: `GET #dPath,hasRecords`
    * `PROCEDURE uinstruction`: `GET #dPath,varRec`
    * `PROCEDURE uinstruction`: `OPEN #vPath,vFile:UPDATE` (Opens "varDefs")
    * `PROCEDRUE uinstruction`: `GET #vPath,varCount`
    * `PROCEDURE uinstruction`: `OPEN #lnPath,lnFile:UPDATE` (Opens "linDefs")
    * `PROCEDURE uinstruction`: `GET #lnPath,linCount`
    * `PROCEDURE uinstruction`: `GET #lnPath,linRec`
    * `PROCEDURE uinstruction`: `SEEK #outPath,0`
    * `PROCEDURE uinstruction`: `READ #outPath,source` (Reads all records from `outPath` until `EOF`, `source` holds the last record)
    * `PROCEDURE uinstruction`: `PRINT #2, "(* Source Instructions *)"` (Prints to standard error path)
    * `PROCEDURE uinstruction`: `PRINT #outPath,"(* Source Instructions *)"`
    * `PROCEDURE uinstruction`: `SEEK #filePath,execOff`
    * `PROCEDURE uinstruction`: `SEEK #lnPath,lnCnt*SIZE(linRec)+2`
    * `PROCEDURE uinstruction`: `GET #filePath,tokenVal`
    * `PROCEDURE uinstruction`: `GET #filePath,intVal`
    * `PROCEDURE uinstruction`: `GET #filePath,byteVal`
    * `PROCEDURE uinstruction`: `GET #filePath,realVal`
    * `PROCEDURE uinstruction`: `PRINT #2, source` (Prints to standard error path)
    * `PROCEDURE uinstruction`: `PRINT #outPath,source`
    * `PROCEDURE uinstruction`: `CLOSE #vPath`
    * `PROCEDURE uinstruction`: `CLOSE #lnPath`
    * `PROCEDURE uinstruction`: `PRINT #2,"Done!";` (Prints to standard error path)
    * `PROCEDURE uinstruction`: `PRINT #2,er` (Prints error code to standard error path in error handler)
    * `PROCEDURE uinstruction (Line 210)`: `SEEK #vPath,varCnt*SIZE(varFRec)+2`
    * `PROCEDURE uinstruction (Line 210)`: `GET #vPath,varFRec`
    * `PROCEDURE uinstruction (Line 230)`: `SEEK #lnPath,lrCnt*SIZE(linRec)+2`
    * `PROCEDURE uinstruction (Line 230)`: `GET #lnPath,linRec`
* **System/Shell I/O:**
    * None.
* **Memory/Hardware I/O:**
    * None.

### 4. Logic Walk-Through

A step-by-step explanation of each procedure's execution flow.

* **`PROCEDURE uinstruction`**
    * **Purpose:** To decompile the executable I-code of a Basic09 procedure by reading a tokenized input file (`filePath`) and associated data files, then writing a human-readable source file (`outPath`).
    * **Parameters (`PARAM`):** Receives input/output file paths (`filePath`, `outPath`), file offsets (`execOff`, `descOff`), a data directory path (`dataDir`), an error variable (`er`), and a `verbose` flag.
    * **Main Logic:**
        1.  Sets array indexing to `BASE 0`.
        2.  Sets a procedure-local error trap: `ON ERROR GOTO 530`.
        3.  Initializes file names (e.g., `vFile:=dataDir+"varDefs"`) and opens data files ("procData", "varDefs", "linDefs") to read metadata, the variable lookup table (`varRec`), and line number information (`linRec`).
        4.  Reads the *last line* from the `outPath` file into the `source` variable.
        5.  Prints a "Source Instructions" header to the `outPath` and (if `verbose`) to standard error (path #2).
        6.  `SEEK`s to the start of the executable code (`execOff`) in the input file (`filePath`).
        7.  Begins a main `REPEAT` loop to process the entire code section (ends at `descOff`).
        8.  Inside the loop, it checks if the current file offset (`modCnt`) matches a known line number offset (`lastOff`). If so, it prepends the line number (e.g., "100 ") to the `source` buffer and fetches the next line record.
        9.  Enters an inner `REPEAT` loop to build a single line of source code.
        10. `GET`s a token (`tokenVal`) from `filePath`.
        11. Calls `GOSUB 270` to look up the token's string (e.g., "PRINT #2, ") from internal `DATA` statements.
        12. Appends the token's string (`tokenName`) to the `source` buffer.
        13. A large decision tree processes the token:
            * **Variable Reference:** `GET`s the variable offset (`intVal`) and calls `GOSUB 210` to find its name.
            * **Literal Value:** `GET`s the literal's value (`byteVal`, `realVal`, `intVal`) and appends its string representation.
            * **Hex Literal:** Calls `RUN uhex$` to get the string.
            * **String Literal:** `GET`s bytes until the `$FF` terminator.
            * **Line Reference:** `GET`s the line offset and calls `GOSUB 230` to find the line number string.
        14. The inner loop ends when an end-of-statement token (`$3E` or `$3F`) is read.
        15. The complete `source` string (which contains RPN expressions) is passed to `RUN uDRPN` to convert it to standard infix notation.
        16. The final, parsed `source` line is `PRINT`ed to `outPath` and (if `verbose`) to path #2.
        17. The main loop repeats for the next line.
        18. After completion, `CLOSE`s all files and prints "Done!".
    * **Subroutines (GOSUB):**
        * `Line 210:` (Find Variable Reference) `SEEK`s into `vPath` to `GET` the variable record (`varFRec`) corresponding to the current token/offset and retrieves its `vName`.
        * `Line 230:` (Find Line Number) `SEEK`s sequentially through `lnPath` to `GET` the line record (`linRec`) that matches the target offset (`intVal`).
        * `Line 270:` (Read Keyword Tokens) `RESTORE`s the `DATA` pointer to line 275 and `READ`s token/string pairs until the correct `tokenVal` is found.
    * **Error Handling:**
        * `Line 530:` (Error Trap) Stores the error code in `er`, `CLOSE`s `lnPath` and `dPath`, `PRINT`s the error code to path #2, and `END`s the procedure.

* **`PROCEDURE uhex$`**
    * **Purpose:** To convert a 16-bit `INTEGER` value into its 4-character hexadecimal string representation.
    * **Parameters (`PARAM`):** Receives the `intVal` (by value) and `hex` string (by reference).
    * **Main Logic:**
        1.  Sets an `ON ERROR GOTO 1` trap, which `END`s the procedure.
        2.  Isolates each 4-bit "nybble" of the `intVal` using `LAND` (Logical AND) and integer division.
        3.  For each nybble, calls `GOSUB 2` to get its character representation.
        4.  Handles the sign bit for negative integers.
        5.  Truncates leading "00" if present.
    * **Subroutines (GOSUB):**
        * `Line 2:` Converts the nybble's numeric value (0-15) to its ASCII character ('0'-'9', 'A'-'F') and prepends it to the `hex` string.
    * **Error Handling:**
        * `Line 1:` `END`s the procedure on any error.

* **`PROCEDURE uDRPN`**
    * **Purpose:** To parse a string containing RPN (postfix) expressions and convert them to standard infix notation using "" as a delimiter.
    * **Parameters (`PARAM`):** Receives the `inline` string to be parsed.
    * **Main Logic:**
        1.  Sets an `ON ERROR GOTO 9` trap.
        2.  Initializes a 10-element string array `templine` to act as a stack.
        3.  Checks for the "" delimiter. If not found, calls `GOSUB 8` (cleanup) and `RETURN`s.
        4.  If operators (e.g., `+`, `*`, `()`) are detected, sets `parse:=TRUE`.
        5.  If `parse` is `FALSE`, calls `GOSUB 1` to strip delimiters, `GOSUB 8` to clean up, and `RETURN`s.
        6.  **Parsing (GOSUB 2):**
        7.  Splits the `inline` string by the "" delimiter and pushes elements onto the `templine` stack (array).
        8.  Iterates through the stack. When an operator is found, it "reduces" the stack by combining the operator with the preceding operands (e.g., `stack[n-2] = "(" + stack[n-2] + stack[n] + stack[n-1] + ")"`).
        9.  Calls `GOSUB 7` to compact the stack (array) after each reduction.
        10. This process repeats until all operators are processed and the stack is reduced.
        11. Reconstructs the `inline` string from the final stack elements.
        12. Calls `GOSUB 8` for final cleanup and `RETURN`s.
    * **Subroutines (GOSUB):**
        * `Line 1:` (Strip delimiters) Iterates `inline` and copies non-"" characters to `outline`.
        * `Line 2:` (Parse line) Implements the main RPN stack-based parsing logic.
        * `Line 7:` (Adjust templines) Compacts the `templine` array by shifting elements to fill empty slots.
        * `Line 8:` (Cleanup) Fixes formatting, such as `.` from `.=` and extra spaces.
    * **Error Handling:**
        * `Line 9:` Stores the error in the `er` variable (which is undeclared in this procedure) and `END`s.