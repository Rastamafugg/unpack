**Code Maintenance Report**

### 1. Procedure & Data Analysis

Analysis of all procedures, their local data, and shared data structures.

**User-Defined Data (`TYPE`)**
* `TYPE HEAD`: Defines the complex data structure for a standard Basic09 module header.
* `TYPE OHEAD`: Defines a partial data structure for an alternate (non-I-Code) module header.
* `TYPE REG`: Defines a complex data structure to map 6809 register values for OS-9 system calls.

**Procedure Interface & Storage**
* **`PROCEDURE unpack`**
    * **Parameters (`PARAM`):**
        * `infile`: Input value, path of the file to decode, type `STRING[80]`.
        * `option`: Input value, command-line options, type `STRING[2]`.
    * **Local Variables (`DIM`):**
        * `header`: Storage for the module header, type `HEAD`.
        * `oheader`: Storage for the alternate module header, type `OHEAD`.
        * `regs`: Storage for system call registers, type `REG`.
        * `filepath`, `outpath`, `callcode`, `char`: Path numbers and temporary storage, type `BYTE`.
        * `packet(32)`: 32-element array for system call data, type `BYTE`.
        * `maxdata`, `er`, `modsiz` (INTEGER), `count`, `counter`, `tpVars`: Loop counters and status variables, type `INTEGER`.
        * `filesize`, `modsize` (REAL), `execoff`, `dataoff`: File and module offsets, type `REAL`.
        * `start`, `symtabOff`, `descOff`, `pointer`: File position pointers, type `REAL`.
        * `nono`, `fileopen`, `outexists`, `verbose`, `outfile`, `ppause`, `skipped`: Program status flags, type `BOOLEAN`.
        * `version`: Program version identifier, type `STRING[10]`.
        * `part`: Stores name of external procedure to `RUN`, type `STRING[11]`.
        * `dataDir`: Path to data directory, type `STRING[16]`.
        * `filename`, `modname`, `outname`: File and module name storage, type `STRING[29]`.
        * `format`: Format string for `PRINT USING`, type `STRING[50]`.
        * `file`: Processed infile path, type `STRING[80]`.
    * **Undeclared Variables:**
        * `data1`: Used as a parameter in `RUN part(...)`. Per documentation, this variable defaults to type `REAL`.

### 2. Call Graph (RUN & GOSUB)

A map of inter-procedure (`RUN`) and intra-procedure (`GOSUB`) dependencies.

* **`PROCEDURE unpack`**
    * `RUN SysCall(callcode, regs)`: (Called multiple times) Executes an external OS-9 machine language system call.
    * `RUN part(...)`: (Called 4 times) Calls external Basic09 procedures stored in the `part` variable ("udecode", "udefVars", "ubuildSrc", "uinstruction").
    * `RUN GFX2("CUROFF")`: (via `GOSUB 50`) Calls an external procedure to disable the cursor.
    * `RUN GFX2("CURON")`: (via `GOSUB 60`) Calls an external procedure to enable the cursor.
    * `GOSUB 30`: Jumps to internal subroutine to turn Page Pause ON.
    * `GOSUB 40`: Jumps to internal subroutine to turn Page Pause OFF.
    * `GOSUB 50`: Jumps to internal subroutine to turn Cursor OFF.
    * `GOSUB 60`: Jumps to internal subroutine to turn Cursor ON.

### 3. External Interaction Points

All statements that interact with the operating system, files, or hardware.

* **File I/O:**
    * `PROCEDURE unpack`: `OPEN #filepath,file:READ` (Opens the main input file).
    * `PROCEDURE unpack`: `CREATE #outpath,outname:WRITE` (Creates the `.B09` source file).
    * `PROCEDURE unpack`: `GET #filepath,header` (Reads the module header).
    * `PROCEDURE unpack`: `GET #filepath,oheader` (Reads the alternate header).
    * `PROCEDURE unpack`: `GET #filepath,oheader.oname` (Reads the alternate module name).
    * `PROCEDURE unpack`: `SEEK #filepath,...` (Used multiple times to position the file pointer).
    * `PROCEDURE unpack`: `PRINT #2,...` (Prints error and status messages to Standard Error).
    * `PROCEDURE unpack`: `PRINT #outpath,...` (Writes source code to the output file).
    * `PROCEDURE unpack`: `PRINT #outpath USING ...` (Writes formatted footer to the output file).
    * `PROCEDURE unpack`: `CLOSE #filepath`
    * `PROCEDURE unpack`: `CLOSE #outpath`
* **System/Shell I/O:**
    * `PROCEDURE unpack`: `KILL part` (Used 4 times to remove external procedures from memory).
    * `PROCEDURE unpack`: `SHELL "UnLink ..."` (Used multiple times to unlink external modules).
    * `PROCEDURE unpack`: `SHELL "Load uinstruction"` (Loads an external module into memory).
* **Memory/Hardware I/O:**
    * None. (No `PEEK` or `POKE` statements are used).

### 4. Logic Walk-Through

A step-by-step explanation of each procedure's execution flow.

* **`PROCEDURE unpack`**
    * **Purpose:** Serves as the main controller for an I-Code decompiler. It reads a target file, parses module headers, and orchestrates a series of external procedures to generate an unpacked Basic09 source file.
    * **Parameters (`PARAM`):**
        * `infile`: The full path to the packed Basic09 file.
        * `option`: Flags for `-V` (Verbose) or `-O` (Output to StdOut only).
    * **Main Logic:**
        1.  Initializes local variables, flags, and version info.
        2.  Sets an error handler `ON ERROR GOTO 10` and parses the `option` parameter.
        3.  Sets an error handler `ON ERROR GOTO 20` and validates the `infile` parameter. If `infile` is missing or invalid (e.g., starts with '-'), jumps to the help screen at line 20.
        4.  Sets the main error handler `ON ERROR GOTO 100`.
        5.  `OPEN`s the `infile` on path `filepath`.
        6.  Calls `RUN SysCall` to check OS-9's page pause status. If it is on, calls `GOSUB 40` to turn it off.
        7.  Calls `GOSUB 50` to turn the cursor off.
        8.  Calls `RUN SysCall` to get the total `filesize`.
        9.  Enters the main `WHILE NOT(EOF(#filepath))` loop to process one or more modules in the file.
        10. `SEEK`s to the `start` of the module and `GET`s its `header`.
        11. Validates the header sync bytes (`$87CD`). If not executable, prints an error and `END`s.
        12. Extracts the `modname`.
        13. Validates the module type. If not Basic09 I-Code (`$22`), it attempts to read an alternate header, prints a "Skipping" message, and advances the `start` pointer.
        14. If it is I-Code, it calculates offsets (`execoff`, `symtabOff`, `descOff`).
        15. If `outfile` flag is true, `CREATE`s the destination `.B09` file.
        16. Prints the `PROCEDURE [modname]` line to the screen and (if active) the output file.
        17. Sequentially `RUN`s the four external decoder parts ("udecode", "udefVars", "ubuildSrc", "uinstruction"), passing parameters.
        18. `KILL`s each part after use and checks the `er` variable. If `er > 0`, jumps to the error handler at line 100.
        19. Prints the `END [modname]` footer to screen and (if active) the output file.
        20. `CLOSE`s the `outpath` and advances the `start` pointer to the next module.
        21. After the `WHILE` loop completes, `CLOSE`s the `filepath`.
        22. Calls `GOSUB 60` to restore the cursor.
        23. If `ppause` was originally on, calls `GOSUB 30` to restore it.
        24. `END`s the program.
    * **Subroutines (GOSUB):**
        * `Line 30:` Turns OS-9 page pause ON via `RUN SysCall`.
        * `Line 40:` Turns OS-9 page pause OFF via `RUN SysCall`.
        * `Line 50:` Turns the cursor OFF via `RUN GFX2("CUROFF")`.
        * `Line 60:` Turns the cursor ON via `RUN GFX2("CURON")`.
    * **Error Handling:**
        * `Line 10:` Local trap for option parsing, jumps to `20` on error.
        * `Line 20:` (Help Screen) Prints program syntax and help text. `CLOSE`s `filepath` if open and `END`s.
        * `Line 100:` (Main Trap) Reads the error code from `ERR`. Prints specific messages for "File Not Found" (216) or "File Already Exists" (218). Prints the generic `ERROR #`, `CLOSE`s `filepath`, restores the cursor (`GOSUB 60`), and `END`s.