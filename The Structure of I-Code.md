# The Structure of I-Code

The 'I' in I-Code stands for Intermediate. Intermediate code is code that is a step between interpreted source statements and fully compiled machine code. Typically, intermediate code uses tokens to represent the instructions to be executed. Basic09 I-Code goes a step further by re-arranging the source code instructions in post-fix notation order (also known as Reverse Polish Notation).

The tokens used by Basic09 are a single byte ranging from $00 to $FF. [See I-Code Token List](#i-code-token-list)

## Contents

1. [Terms Used](#terms-used)
2. [Module Format](#module-format)
   1. [Module Header](#module-header)
      1. [NOTES](#notes)
   2. [I-Code Area](#i-code-area)
   3. [Description Area (DSAT)](#description-area-dsat)
   4. [Total Program Variables](#total-program-variables)
   5. [Symbol Table (VDT)](#symbol-table-vdt)
   6. [CRC](#crc)
3. [Instruction I-Code](#instruction-i-code)
   1. [Internal Structures](#internal-structures)
   2. [Files](#files)
   3. [Data Memory Map](#data-memory-map)
   4. [Structures](#structures)
      1. [FOR/NEXT](#fornext)
      2. [WHILE/ENDWHILE](#whileendwhile)
      3. [REPEAT/UNTIL](#repeatuntil)
      4. [LOOP/ENDLOOP](#loopendloop)
   5. [Conditional](#conditional)
      1. [IF-THEN <lref>](#if-then-lref)
      2. [IF-THEN/ENDIF](#if-thenendif)
      3. [IF-THEN/ELSE/ENDIF](#if-thenelseendif)
      4. [EXITIF-THEN/ENDEXIT](#exitif-thenendexit)
   6. [Literals](#literals)
4. [About GOTO, GOSUB and <lref> tokens](#about-goto-gosub-and-lref-tokens)
5. [DATA Statement Structure](#data-statement-structure)
6. [File Access Mode Tokens](#file-access-mode-tokens)
7. [I-Code Variable Map](#i-code-variable-map)
8. [I-Code Instruction Variable Tokens](#i-code-instruction-variable-tokens)
9. [Symbol Table (VDT) Tokens](#symbol-table-vdt-tokens)
10. [Keyword Tokens](#keyword-tokens)
11. [Secondary Keyword Tokens](#secondary-keyword-tokens)
12. [Punctuation Tokens](#punctuation-tokens)
13. [String Length Value Tokens](#string-length-value-tokens)
14. [Assignment Operator Tokens](#assignment-operator-tokens)
15. [File Mode and Path Operator Tokens](#file-mode-and-path-operator-tokens)
16. [Internal Tokens](#internal-tokens)
17. [Numeric Function Tokens](#numeric-function-tokens)
18. [Screen/Print Function Tokens](#screenprint-function-tokens)
19. [File and Other Function Tokens](#file-and-other-function-tokens)
20. [String Function Tokens](#string-function-tokens)
21. [String-to-Number Function Tokens](#string-to-number-function-tokens)
22. [Number-to-String Function Tokens](#number-to-string-function-tokens)
23. [Logical Operator Tokens](#logical-operator-tokens)
24. [Boolean Function Tokens](#boolean-function-tokens)
25. [Comparison Operator Tokens](#comparison-operator-tokens)
26. [Math Operator Tokens](#math-operator-tokens)
27. [I-Code Token List](#i-code-token-list)

## Terms Used

| | |
|---|---|
| Vector | 1 Dimensional Array |
| Table | 2 Dimensional Array |
| Matrix | 3 Dimensional Array |
| Variable Declaration Table (VDT) | Symbol Table |
| Data Storage Allocation Table (DSAT) | Description Area |

The terms Vector, Table and Matrix, as used to describe the arrays, is defined in the Microware OS-9 Basic User Manual (for OS-9/68000, (c)1991), Chapter 3: Program Construction: Complex Data Types and Subroutines on page 3-1.

The terms VDT and DSAT are terms I invented before I knew what Microware named those tables. They are based on what I perceived the purpose of the tables to be.

## Module Format

Basic09 I-Code modules use the same module format as any other OS-9 executable module. However, I-Code modules include optional header extensions not included in other types of memory modules. They also include two very important tables specific to I-Code modules.

Modules are divided into the following sections:

  * [Module Header](#module-header)
  * [I-Code Area](#i-code-area)
  * [Description Area](#description-area-dsat)
  * [Total Program Variables](#total-program-variables)
  * [Symbol Table](#symbol-table-vdt)
  * [CRC](#crc)

### Module Header

| Description | Size | Note |
|---|---|---|
| Module Header Space (Sync Bytes) | INTEGER | Value always $87CD |
| Total Procedure Block Size | INTEGER | Total size of the module, including CRC and header\*\* |
| Module Name Offset | INTEGER | Offset address in the header of the Procedure Name Beginning |
| Module Type/Language | BYTE | Value always $22; Sub-Routine/Basic09 I-Code\* |
| Module Attributes/Revision | BYTE | Value always $81; Re-Entrant,Re-Locatable Object/Revision=1 |
| Module Header Parity | BYTE | Last required module header entry; optional header extensions follow\* |
| I-Code Area Address | INTEGER | Offset address of the I-Code instructions |
| Runtime Variable Storage Size | INTEGER | Total data memory size for the module |
| Symbol Table Address | INTEGER | Offset address of the VDT\*\* |
| Description Area Address | INTEGER | Offset address of the DSAT\*\* |
| Procedure Link Storage Offset | INTEGER | Beginning of Named Subroutine Storage in Data Memory |
| First Data Statement Address | INTEGER | Offset address of the first element of the first data statement\*\* |
| First Executable Statement Address | INTEGER | Value always $0000\*\*; See **2** |
| Procedure Status | BYTE | Value always $80\*; See **3** |
| Procedure Name Size | BYTE | |
| Procedure Name Beginning | Procedure Name Size BYTEs | Last character hi-bit set |
| Edition Number | BYTE | Always the first byte after the Module Name. This BYTE is actually the first byte of I-Code, at the I-Code Area Address location, and not part of the header. See **1** |

#### NOTES
-----

**1**: The OS-9 Level II Manual describes the Edition byte in Chapter 6. System Command Descriptions, under the Ident command on pages 6-52,53.<br>The OS-9 Level I Manual Set includes this information in the OS9 Commands Manual, Chapter 6. System Command Descriptions, Ident command, page 87.<br>
**2**: P.EXEC (First Executable Statement Address) is also used when in BASIC09 - it's an offset into the module to start executing (used when switching between which procedure is currently being run) that helps with debugging, etc. *Thanks to L. Curtis Boyle for this information.*<br>
**3**: Procedure Status, P.STAT in the Microware defs sourcefile, should always be $80 when a procedure is packed. It can have different values when running BASIC09 in source code form. (If I remember, the least significant bit, if set, means "Line with compiler error" (i.e. it couldn't compile due to an error)) *Thanks to L. Curtis Boyle for this information.*

  \* These values are not set in unpacked I-Code.  
  \*\* These values are different in unpacked I-Code.

### I-Code Area

This is where all of your instruction statements are stored in tokenized post-fix notation form. While there is no requirement to do so, it is typical to see this area terminated with a END ($39) token followed by a line termination token ($3F).

Basic09 allows you to place TYPE, DIM and PARAM statements anywhere in the source code. When Basic09 loads your source, it immediately converts the source to tokenized form. The tokenized code is in the same form as a packed module, except the Symbol Table includes a complete list of your program variables. All line references (the numbers at the beginning of a line) are denoted with the $3A token and an integer value that is the line number itself. Branch statements (GOTO ($20) and GOSUB ($22) (including ON ERROR GOTO, and ON <expr> GOTO/GOSUB), RESTORE <lref> ($31) and IF-THEN \<lref\> ($10-$45) statements) reflect different locations. Not all of them point to the beginning of a statement, or to the first byte after the line reference. Packing the procedure corrects this and all branch statements point to the correct location in the module. When in edit mode branch statements (GOTO $20 and GOSUB $22) are replaced with their unbound counterparts ($1F and $21 respectively), and RESTORE ($31), IF-THEN \<lref\> ($10-$45) and ON \<expr\> GOTO/GOSUB (which all use the $3B token) are changed to their unbound counterparts ($3A). In addition, there are internal offset branch tokens, including invisible goto (\<ivgt\> $55), ELSE ($11), ENDWHILE ($16) and ENDLOOP ($1A).

  * [See I-Code Instruction Variable Tokens](#i-code-instruction-variable-tokens).

### Description Area (DSAT)

This is the first of the two variable-related tables in I-Code modules. In reality, this section contains the shape data for all variables more complex than a simple atomic type. The simple atomic types are BYTE, BOOLEAN, INTEGER and REAL. String variables in the I-Code area point directly to this table. Record and record field variables point to the Symbol Table (VDT) which in turn points to the position of the variable in the record (except string and record fields which also point to this table). The form the shape data takes is specific to the type of variable being described. The most simple form is:

**offset - size** (both INT-size)

Offset is the data memory location of the variable, and size is the length of the variable. (In the case of field strings or records, the offset is the position in the record where that field begins.) This form is used, by itself, to describe non-array STRING and record variables. All other entries (except TYPE shape data) are the array variables and begin with these two fields, then add extra fields to describe the variable. These variables include the following fields (all INT-size):

  * first element size (elem1 - all arrays, number of vector elements)
  * second element size (elem2 - all table and matrix arrays, number of table elements)
  * third element size (elem3 - all matrix arrays, number of matrix elements)

The above are actually added in reverse order:

```
  offset - size - elem3 - elem2 - elem1[ - elemSize]
```

  * element length (elemSize - all STRING and record arrays, the size of one string or record element)

TYPE statement data is different. First, all complex data forms included in the TYPE are listed immediately before the shape data for the TYPE. In these cases, the offset is not a data memory address, but the offset within the record for that variable. The form the TYPE shape data takes is as follows:

```
  size (INT); unknown use (1 BYTE); first field (2 BYTEs, first BYTE is high-bit set (I think)); second field (INT); ... <nth> field (INT)
```

TYPEs within TYPEs adds a new level of complexity. Before the shape data that includes a field DIM'd as a record, the shape data for that record TYPE is defined. as well as any other complex variables defined in that TYPE occurring before its shape data.

TYPE field variable references are in ascending value order from the first field of the first TYPE to the last field of the last TYPE.

### Total Program Variables

At the end of this table is another "3-byte" structure. However, this one is reversed, so it is \<INT\>-\<BYTE\>, and each can be viewed as separate from the other.

The INT is the Total Program Variables count. This number includes total TYPE field variables + total DIM variables + total PARAM variables + total by-use variables (these include the MIRROR variables) + all named subroutines (not including subroutine names assigned only to string variables). Any variables that were declared in a TYPE or DIM statement that were never used are still counted. Also counted are the TYPE variables themselves (e.g. in TYPE REG=, REG counts as a variable.)

The BYTE, in every module I have seen except one, holds the value $03. In the one case, it held the value $04. I no longer have the module that contained $04, and I have never understood what this byte is used for. It is the final byte before the beginning of the Symbol Table.

### Symbol Table (VDT)

This is the second of the two variable-related tables in I-Code modules. The first thing you will notice is that this table is almost entirely 3-byte structures, similar to the ones found in the I-Code area. The difference here is the number of different tokens for defining variables according to their atomic type, as well as STRINGs, records and arrays. Parameter variables are listed here, following the program variables, and after that comes the named subroutine (external procedure/object-code subroutine) list.

Sub-routine entries begin with the $A0 token, followed by the data memory offset of the pointer to that subroutine, and the text of the name of the subroutine (last character hi-bit set, same as the module name in the header of the module). The Procedure Link Storage Offset in the header points to the first memory location that holds the pointer to the first subroutine named in this section of the list. For all other VDT tokens, see the Symbol Table Tokens.

Unpacked I-Code includes all TYPE variable names (e.g. in TYPE REG, REG is the TYPE variable name) using the token $25. Also, all entries include the variable name associated with that entry. Packing strips the TYPE entries and variable names from the table, except the named subroutine names.

The offset field in the program variables and parameter variables portion of the list points to one of three places.

  * In all atomic record field entries ($40-$43) the offset is the position within the record of that field. The atomic type determines the size.
  * In all atomic variables ($60-$63), the offset is the data memory location of that variable, and the size is determined by the type.
  * In all atomic parameter variables ($80-$83), the offset is the data memory location of that variable, and the size is four BYTEs.
  * In all other variable types, the offset is relative to the beginning of the DSAT.

The data size for each parameter variable, whether atomic or not, is four BYTEs. That is the difference between one parameter's data memory address and the next one. The DSAT contains the shape data for the more complex variables, but the data memory pointer in a called procedure is a 4-byte pointer.

Note: In a perfect world these three sections of the VDT would be completely segregated. However, this is not the case in the real world. You will see times when there are program variables occurring within the parameter and subroutine sections. I think this is due to repeated edit/save/pack sessions. I believe that a minor change will not result in rebuilding the entire list, especially if you do not reload the source after saving changes.

### CRC

All OS-9 memory modules end with a 3-byte CRC. While I have not seen a utility for re-calculating a module's CRC, there is a system call that can be used for that purpose.

## Instruction I-Code

-----

In this section I will describe the various structures used in Basic09 coding, including the looping constructs, conditional statements, and the use of literals (constants).

### Internal Structures

As you have seen, I-Code makes use of 3-byte structures for variable references. It uses them as well for branch references (including line references). They use the form:

**token - offset**

where token is any valid Basic09 branch or variable token, and offset is a INTEGER-size value. All addressing in I-Code is relative. Branch references are relative to the beginning of the I-Code area, and variable reference offsets are relative to one of the following:

  * BYTE, INTEGER, REAL and BOOLEAN references (token values $80-$83): data memory offset relative to the beginning of data memory.
  * STRING references (token value $84): DSAT offset relative to the beginning of the Description area.
  * all other token values in the I-Code area: VDT offset relative to the beginning of the Symbol Table.

### Files

File access in Basic09 is accomplished through a specific set of keywords and functions. OPEN and CREATE statements deal with opening or creating a file on your disk. Both statements have the same syntax, and except for the OPEN or CREATE keyword are syntactically identical:

**CREATE \#path,filename**
| 29 | 54 | 80 0016 | 4B | 84 0000 | 3F |
|---|---|---|---|---|---|
| CREATE | \# | path | , | filename | \<eol\> |

**OPEN \#path,filename:READ**
| 2A | 54 | 80 0016 | 4B | 84 0000 | 4A | 01 | 3F |
|---|---|---|---|---|---|---|---|
| OPEN | \# | path | , | filename | : | READ | \<eol\> |

**SEEK \#path,SIZE(myVar)**
| 2B | 54 | 80 0016 | 4B | 95 | F2 0027 | 94 | 3F |
|---|---|---|---|---|---|---|---|
| SEEK | \# | path | , | \<size\> | myVar | SIZE() | \<eol\> |

**CLOSE \#path**
| 30 | 54 | 80 0016 | 3F |
|---|---|---|---|
| CLOSE | \# | path | \<eol\> |

**DELETE filename**
| 32 | 84 0000 | 3F |
|---|---|---|
| DELETE | filename | \<eol\> |

  * See [File Access Mode Tokens](#file-access-mode-tokens).

### Data Memory Map

```
  0                  $15|$16
 +----------------------+-----------------+--------------------+--------------+
 |reserved by RunB      |program variables|named procedure list|parameter list|
 +----------------------+-----------------+--------------------+--------------+
```

The first $16 (22) bytes of the data space is reserved for use by RunB. Following this are the program variables, the named procedure list, and the parameter list. The reserved bytes are defined in the Basic09 header file as follows:

```
* Procedure Invocation Temporaries

org 0

U.RUNM rmb 1 Run mode flag
U.DEG  rmb 1 Deg/rad flag
U.BASE rmb 1 Array/vector base flag (0/1)
U.PROC rmb 2 Procedure ptr
U.S    rmb 2 S register on entry
U.U    rmb 2 U register on entry
U.DATA rmb 2 Data ptr
U.ICPT rmb 2 I-code program counter
U.STBG rmb 2 String stack ptr
U.PRLM rmb 2 Parameter limit
U.ERRA rmb 2 Error trap address
U.ERRS rmb 1 Error trap armed flag
U.SBSP rmb 2 Subroutine stack pointer

U.SIZE equ . Size of invocation temporaries
```

### Structures

#### FOR/NEXT

The FOR/NEXT loop structure is the most complex in terms of I-Code. It contains the most fields of any other loop structure in Basic09.

**FOR R0122= 1 TO 8 \\**
| 13 | 82 07D3 | 53 | 8D 01 | CB | 46 07D8 | 8D 08 | CB | 55 006A | 3E |
|---|---|---|---|---|---|---|---|---|---|
| FOR | R0122 | = | \<bc\> 1 | \<flt1\> | TO | \<bc\> 8 | \<flt1\> | \<ivgt\> | \\ |

**13** is the **FOR** token<br>
**82** is the **REAL** type token<br>
**07D3** is the data memory offset for a REAL variable named **R0122**, which occupies 5 bytes beginning at that location<br>
**53** is the **=** assignment operator token<br>
**8D** is the **\<bc\>** (byte constant) token, and is associated to the following numeric value<br>
**01** is the value **1**<br>
**CB** is the **\<flt1\>** (float top of stack) token (changes the previous non-REAL value to a REAL value)<br>
**46** is the **TO** token<br>
**07D8** is the data memory offset for a temporary variable used to store the following value<br>
**8D** is the **\<bc\>** (byte constant) token, and is associated to the following numeric value<br>
**08** is the value **8**<br>
**CB** is the **\<flt1\>** (float top of stack) token<br>
**55** is the **\<ivgt\>** (invisible goto) token<br>
**006A** is the module offset to branch to when the condition fails<br>
**3E** is the \**\** instruction terminator. This character allows stacking of mutiple statements on a single line in the source code. Many times, these occur because the author included line comments, such as \\REM populate array. Usually, this will be the \<eol\> (3F) terminator

**NEXT R0122**
| 14 | 02 | 07D3 | 07D8 | 0000 | 004C | 3F |
|---|---|---|---|---|---|---|
| NEXT | | R0122 | TO | STEP | \<ivgt\> | \<eol\> |

**14** is the **NEXT** token<br>
**02** determines REAL or INTEGER, with or without a STEP value.

1.  **00** INTEGER
2.  **01** INTEGER w/STEP
3.  **02** REAL
4.  **03** REAL w/STEP
    **07D3** is the data memory address for **R0122**<br>
    **07D8** is the **TO** variable<br>
    **0000** is the **STEP** variable, if included in the FOR statement. In every FOR loop that does not include a STEP value, this integer is 0<br>
    **004C** **\<ivgt\>** pointing to the beginning of the first instruction within the loop. Note the lack of a $55 token. This is the only condition where a \<ivgt\> does not have the token<br>
    **3F** is the **\<eol\>** (end of instruction/line) token

#### WHILE/ENDWHILE

WHILE loops test an expression as TRUE or FALSE at the beginning of the loop. As a result, these loops may be skipped if the condition is already FALSE.

**WHILE NOT(EOF(\#B0028)) DO**
| 15 | 80 0329 | BE | CD | 55 0DE7 | 48 | 3F |
|---|---|---|---|---|---|---|
| WHILE | B0028 | EOF() | NOT() | \<ivgt\> | DO | \<eol\> |

**15** is the **WHILE** token<br>
**80** is the **BYTE** type token<br>
**0329** is the data memory address of a variable named **B0028**<br>
**BE** is the **EOF()** function token<br>
**CD** is the **NOT()** function token<br>
**55** is the **\<ivgt\>** token<br>
**0DE7** is the offset to branch to when the condition fails<br>
**48** is the **DO** token<br>
**3F** is the **\<eol\>** token

**ENDWHILE**
| 16 | 0CBF | 3F |
|---|---|---|
| ENDWHILE | \<offset\> | \<eol\> |

**16** is the **ENDWHILE** token<br>
**0CBF** is the offset address of the WHILE statement<br>
**3F** is the **\<eol\>** token

#### REPEAT/UNTIL

REPEAT loops will always be executed at least once, since the test for a TRUE/FALSE condition occurs at the bottom of the loop.

**REPEAT**
| 17 | 3F |
|---|---|
| REPEAT | \<eol\> |

**17** is the **REPEAT** token<br>
**3F** is the **\<eol\>** token

**UNTIL MID$(S0107,I0051,1)="/" OR I0051=0**
| 18 | 84 01FF | 81 0378 | 8D 01 | C0 | 90 2F FF | DF | 81 0378 | 8D 00 | DD | D1 | 55 102C | 3F |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| UNTIL | S0107 | I0051 | &lt;bc&gt; 1 | MID$(,,) | "/" | = | I0051 | \<bc\> 0 | = | OR | \<ivgt\> | \<eol\> |

**18** is the **UNTIL** token<br>
**84** is the **STRING** type token<br>
**01FF** is the data memory address of a variable named **S0107**<br>
**81** is the **INTEGER** type token<br>
**0378** is the data memory address of a variable named **I0051**<br>
**8D** is the **\<bc\>** token<br>
**01** is the value **1**<br>
**C0** is the **MID$** function token<br>
**90** is the **"** beginning-of-string token<br>
**2F** is the **/** character<br>
**FF** is the **"** end-of-string token<br>
**DF** is the **=** string comparison operator token<br>
**81** is the **INTEGER** type token<br>
**0378** is the data memory address of a variable named **I0051**<br>
**8D** is the **\<bc\>** token<br>
**00** is the value **0**<br>
**DD** is the **=** byte/integer comparison operator token<br>
**D1** is the **OR** boolean operator token<br>
**55** is the **\<ivgt\>** token<br>
**102C** is the offset address of the first instruction inside the loop<br>
**3F** is the **\<eol\>** token

#### LOOP/ENDLOOP

This is the most simple loop structure in Basic09. The EXITIF-THEN/ENDEXIT construct is the only way to exit this loop.

**LOOP**
| 19 | 3F |
|---|---|
| LOOP | \<eol\> |

**19** is the **LOOP** token<br>
**3F** is the **\<eol\>** token

**ENDLOOP**
| 1A | 0BC6 | 3F |
|---|---|---|
| ENDLOOP | \<offset\> | \<eol\> |

**1A** is the **ENDLOOP** token<br>
**0BC6** is the offset address of the first instruction inside the loop<br>
**3F** is the **\<eol\>** token

### Conditional

#### IF-THEN \<lref\>

This form of IF/THEN does not use a ENDIF token.

**IF I0040=0 THEN 20**
| 10 | 81 035E | 8D 00 | DD | 55 0395 | 45 | 3B 05E1 | 3F |
|---|---|---|---|---|---|---|---|
| IF | I0040 | \<bc\> 0 | = | \<ivgt\> | THEN | \<blref\> 20 | \<eol\> |

**10** is the **IF** token<br>
**81** is the **INTEGER** type token<br>
**035E** is the data memory address of the variable **I0040**<br>
**8D** is the **\<bc\>** token<br>
**00** is the value **0**<br>
**DD** is the **=** byte/integer comparison operator token<br>
**55** is the **\<ivgt\>** token<br>
**0395** is the offset address to branch to if the condition fails, the next instruction<br>
**45** is the **THEN** token<br>
**3B** is the **\<blref\>** token<br>
**05E1** is the offset address inferred by the reference **20** and is where execution will continue if the condition is TRUE<br>
**3F** is the **\<eol\>** token

#### IF-THEN/ENDIF

**IF LEFT$(S0098,2)="-D" THEN \\**
| 10 | 84 01CF | 8D 02 | C1 | 90 2D44 FF | DF | 55 03DD | 45 | 3E |
|---|---|---|---|---|---|---|---|---|
| IF | S0098 | &lt;bc&gt; 2 | LEFT$(,) | "-D" | = | \<ivgt\> | THEN | \\ |

**10** is the **IF** token<br>
**84** is the **STRING** type token<br>
**01CF** is the DSAT offset of the variable **S0098**<br>
**8D** is the **\<bc\>** token<br>
**02** is the value **2**<br>
**C1** is the **LEFT$** token<br>
**90** is the beginning **"** token<br>
**2D44** is the characters **-D**<br>
**FF** is the ending **"** token<br>
**DF** is the **=** string comparison operator<br>
**55** is the **\<ivgt\>** token<br>
**03DD** is the offset address to branch to if the condition fails<br>
**45** is the **THEN** token<br>
**3E** is the \**\** token

**ENDIF**
| 12 | 3F |
|---|---|
| ENDIF | \<eol\> |

**12** is the **ENDIF** token<br>
**3F** is the **\<eol\>** token

#### IF-THEN/ELSE/ENDIF

**IF LEFT$(S0098,3)="-RR" THEN**
| 10 | 84 01CF | 8D 03 | C1 | 90 2D5252 FF | DF | 55 0463 | 45 | 3F |
|---|---|---|---|---|---|---|---|---|
| IF | S0098 | &lt;bc&gt; 3 | LEFT$(,) | "-RR" | = | \<ivgt\> | THEN | \<eol\> |

**10** is the **IF** token<br>
**84** is the **STRING** type token<br>
**01CF** is the DSAT offset of the variable **S0098**<br>
**8D** is the **\<bc\>** token<br>
**03** is the value **3**<br>
**C1** is the **LEFT$** token<br>
**90** is the beginning **"** token<br>
**2D5252** is the characters **-RR**<br>
**FF** is the ending **"** token<br>
**DF** is the **=** string comparison operator<br>
**55** is the **\<ivgt\>** token<br>
**0463** is the offset address to branch to if the condition fails, the first instruction inside the ELSE clause<br>
**45** is the **THEN** token<br>
**3F** is the **\<eol\>** token

**ELSE**
| 11 | 046B | 3F |
|---|---|---|
| ELSE | \<offset\> | \<eol\> |

**11** is the **ELSE** token<br>
**046B** is the offset address of the first instruction after the ENDIF and \<eol\> tokens)<br>
**3F** is the **\<eol\>** token

**ENDIF**
| 12 | 3F |
|---|---|
| ENDIF | \<eol\> |

**12** is the **ENDIF** token<br>
**3F** is the **\<eol\>** token

#### EXITIF-THEN/ENDEXIT

**EXITIF IA0064(I0045,R0125)=0 THEN**
| 1B | 81 0368 | 82 2952 | C8 | 87 00E4 | 8D 00 | DD | 55 1CA2 | 45 | 3F |
|---|---|---|---|---|---|---|---|---|---|
| EXITIF | I0045 | R0125 | \<flt1\> | IA0064 | \<bc\> 0 | = | \<ivgt\> | THEN | \<eol\> |

**10** is the **IF** token<br>
**81** is the **INTEGER** type token<br>
**0368** is the data memory address of the variable **I0045**<br>
**82** is the **REAL** type token<br>
**2952** is the data memory address of the variable **R0125**<br>
**C8** is the **\<flt1\>** token<br>
**87** is the **INTEGER** table array type token<br>
**00E4** is the DSAT offset of the variable **IA0064**<br>
**8D** is the **\<bc\>** token<br>
**00** is the value **0**<br>
**DD** is the **=** integer comparison operator<br>
**55** is the **\<ivgt\>** token<br>
**1CA2** is the offset address to branch to if the condition fails, the next instruction after the ENDEXIT statement, or the first instruction inside the loop structure if ENDEXIT is the last statement inside the loop structure (can be used in any loop structure)<br>
**45** is the **THEN** token<br>
**3F** is the **\<eol\>** token

**ENDEXIT**
| 1C | 1F09 | 3F |
|---|---|---|
| ENDEXIT | \<offset\> | \<eol\> |

**1C** is the **ENDEXIT** token<br>
**1F09** offset of the next instruction after the end of loop statement<br>
**3F** is the **\<eol\>** token

### Literals

Basic09 does not have named constants.\<b\>\*\</b\> Because of this, all "constants" in Basic09 are of the literal variety. If you want to use variables as constants, you may do so, but it is up to you to ensure that the variables being used as constants are not being modified. To help with this, the recommended form of variable naming is to use a lowercase 'c' or a underscore '\_' as the first character of the variable name (e.g. cMyConstant or cMYCONSTANT or \_myConstant or \_MYCONSTANT).

| Type | Token | Value |
|---|---|---|
| BYTE | $8D | numeric literal from 0 to 255/-128 to 127 |
| INTEGER | $8E | numeric literal from -32768 to 32767 |
| REAL | $8F | numeric literal in the range of a Basic09 REAL |
| Hexadecimal Integer | $91 | hexadecimal value from $0000 to $FFFF |
| BOOLEAN | $BC = TRUE<br>$BD = FALSE | |
| STRING | $90 = beginning bound of string<br>$FF = ending bound of string | string of text |

  \* Basic09 has one named constant: PI. It is listed with the functions, but the descriptions in the manual say "[Represents ]the constant 3.14159265" (Level 1:Chapter 8. Expressions, Operators, and Functions:p. 54) (Level 2:Chapter 7. Expressions, Operators, and Functions:p. 7-8)

## About GOTO, GOSUB and \<lref\> tokens

There are two types of these identified in the Basic09 header. They are identified as bound and unbound.

| | GOTO | GOSUB | \<lref\> |
|---|---|---|---|
| Unbound | 1F | 21 | 3A |
| Bound | 20 | 22 | 3B |

The bound versions are the ones that occur in packed and unpacked I-Code. The unbound versions are used when Basic09 is in edit mode.

As Basic09 loads the source, it is replacing all keywords, function names and other items with tokens. As it comes across GOTO, GOSUB and \<lref\> identifiers (THEN \<lnum\>, RESTORE \<lnum\>, ON ERROR GOTO \<lnum\>, ON \<expr\> GOTO/GOSUB \<lnums\>), it uses the bound tokens to identify them as branch statements. Lines that begin with a line number are given a $3A token followed by the number used by the programmer. When the procedure is put in edit mode, the bound tokens on the branch statements are replaced with their unbound counterparts. The offset addresses in the branch statements are altered to point to the line referenced. While it would make sense that they would be pointing to the $3A at the beginning of the line, the truth is the pointer can be anywhere near the beginning of that line. In my experiments, the unbound pointer was never at the beginning of the statement, and was as far away as the third byte after the line number integer. When the procedure is packed, all of those references are resolved correctly and point to the beginning of the instruction.

This is why the line number at the beginning of a line can disappear from your I-Code. Without an identifier to point to it, that reference does not exist in packed I-Code.

## DATA Statement Structure

DATA statements in Basic09 are essentially linked-lists.

The 1st data statement address field in the module header points to the first byte of the first item in the first DATA statement in the procedure. It does not point to the DATA token.

At the end of each DATA statement is a invisible goto that points to the first byte of the first item in the next DATA statement, or of the first DATA statement if the current DATA statement is the last DATA statement in the procedure.

It is the address in the header that allows RESTORE with no line number to reset to the beginning of the DATA statements.

RESTORE with a line number allows you to insert points within the structure to allow the pointer to be re-positioned.

The structure is not doubly-linked, and you can only move through it sequentially in one direction, top-down.

**DATA 9600,4800,2400,1200,300**
| 1st DATA statement address:      0066 | | | | | | | | | | | | | |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Module<br>Offset | Execution<br>Offset | DATA | 9600 | , | 4800 | , | 2400 | , | 1200 | , | 300 | \<ivgt\> | \<eol\> |
| 00000084 | 0064 | 04 | 8E 2580 | 4B | 8E 12C0 | 4B | 8E 0960 | 4B | 8E 04B0 | 4B | 8E 012C | 55 0066 | 3F |

## File Access Mode Tokens

| | | | |
|---|---|---|---|
| READ+WRITE = UPDATE | | | |
| | | 80 | DIR |
| 01 | READ | 81 | READ+DIR |
| 02 | WRITE | 82 | WRITE+DIR |
| 03 | UPDATE | 83 | UPDATE+DIR |
| 04 | EXEC | 84 | EXEC+DIR |
| 05 | READ+EXEC | 85 | READ+EXEC+DIR |
| 06 | WRITE+EXEC | 86 | WRITE+EXEC+DIR |
| 07 | UPDATE+EXEC | 87 | UPDATE+EXEC+DIR |

## I-Code Variable Map

This map shows how variable references are routed from the I-Code Area to Data Memory. It also shows the different token ID's in each area.

```
              I-Code Area             Description Area
           +----------------+      +-------------------------+
      +----| $80-$83        |----->| $84                     |
      |    | $84            | +--->| $44-$5D,$64-$7D,$84-$9D |
      | +--| $85-8C, $F2-F9 | |    +-------------------------+
      | |  +----------------+ |                       |
      | |                     |                       |
      | |     Symbol Table    |                       |
      | |  +----------------+ |                       |
      | +->| $40-5D         |-+                       |
      |    | $60-7D         |-----------------------+ |
      |    | $80-9D         |                       | |
      |    +----------------+                       | |
      |                                             | |
      |       Data Memory                           | |
      |    +-------------------------------------+  | |
      +--->| $80-$83|$84|$40-$5D|$60-$7D|$80-$9D |<-+-+
           +-------------------------------------+
```

## I-Code Instruction Variable Tokens

Atomic (ATM) types here include Records, even though they are not implicit atomic types, to distinguish them from arrays. I have never liked the usage of MIRROR in this instance. I have no better way to describe these tokens at this time. I referred to them as mirror tokens, because in some instances they act as a duplicate of the variable, such as in the case of a:=a+1. Both 'a' variables refer to the same variable, but in the I-Code, each has a separate token, and with the exception of 80-84 (in the instruction area) point to the VDT. 80-83 in the instruction area point to data memory addresses, and 84 points to the DSAT.

| TYPE | ATOMIC - TOKEN | ATOMIC - MIRROR | FIELD - TOKEN | FIELD - MIRROR | PARAMETER - TOKEN | PARAMETER - MIRROR |
|:---|:---:|:---:|:---:|:---:|:---:|:---:|
| BYTE | 80 | F2 | F6 | 89 | 85 | F2 |
| INTEGER | 81 | F2 | F6 | 89 | 85 | F2 |
| REAL | 82 | F2 | F6 | 89 | 85 | F2 |
| BOOLEAN | 83 | F2 | F6 | 89 | 85 | F2 |
| STRING | 84 | F2 | F6 | 89 | 85 | F2 |
| RECORD | 85 | F2 | F6 | 89 | 85 | F2 |
| VECTOR | 85/86 | F3 | F7 | 8A | 85/86 | F3 |
| TABLE | 85/87 | F4 | F8 | 8B | 85/87 | F4 |
| MATRIX | 85/88 | F5 | F9 | 8C | 85/88 | F5 |

## Symbol Table (VDT) Tokens

SIM = Simple   VEC = Vector   TAB = Table   MAT = Matrix
| TYPE | FIELD - SIM | FIELD - VEC | FIELD - TAB | FIELD - MAT | ATOMIC - SIM | ATOMIC - VEC | ATOMIC - TAB | ATOMIC - MAT | PARAMETER - SIM | PARAMETER - VEC | PARAMETER - TAB | PARAMETER - MAT |
|:---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| BYTE | 40 | 48 | 50 | 58 | 60 | 68 | 70 | 78 | 80 | 88 | 90 | 98 |
| INTEGER | 41 | 49 | 51 | 59 | 61 | 69 | 71 | 79 | 81 | 89 | 91 | 99 |
| REAL | 42 | 4A | 52 | 5A | 62 | 6A | 72 | 7A | 82 | 8A | 92 | 9A |
| BOOLEAN | 43 | 4B | 53 | 5B | 63 | 6B | 73 | 7B | 83 | 8B | 93 | 9B |
| STRING | 44 | 4C | 54 | 5C | 64 | 6C | 74 | 7C | 84 | 8C | 94 | 9C |
| RECORD | 45 | 4D | 55 | 5D | 65 | 6D | 75 | 7D | 85 | 8D | 95 | 9D |

## Keyword Tokens

| | | | | | | | | | |
|---|---|---|---|---|---|---|---|---|---|
| 04 | DATA | 31 | RESTORE | | | | | | |
| 05 | STOP | 09 | PAUSE | 06 | BYE | 39 | END | | |
| 07 | TRON | 08 | TROFF | 0A | DEG | 0B | RAD | | |
| 0C | RETURN | 0D | LET | 0F | POKE | | | | |
| 10 | IF | 11 | ELSE | 12 | ENDIF | | | | |
| 13 | FOR | 14 | NEXT | 15 | WHILE | 16 | ENDWHILE | | |
| 17 | REPEAT | 18 | UNTIL | 19 | LOOP | 1A | ENDLOOP | | |
| 1B | EXITIF | 1C | ENDEXIT | | | | | | |
| 1D | ON | 1E | ERROR | 20 | GOTO | 22 | GOSUB | | |
| 23 | RUN | 24 | KILL | 25 | INPUT | 26 | PRINT | | |
| 27 | CHD | 28 | CHX | | | | | | |
| 29 | CREATE | 2A | OPEN | 2B | SEEK | 2C | READ | 2D | WRITE |
| 2E | GET | 2F | PUT | 30 | CLOSE | 32 | DELETE | | |
| 33 | CHAIN | 34 | SHELL | 35 | BASE 0 | 36 | BASE 1 | | |

## Secondary Keyword Tokens

| | |
|---|---|
| 45 | THEN |
| 46 | TO |
| 47 | STEP |
| 48 | DO |
| 49 | USING |

## Punctuation Tokens

| | | | |
|---|---|---|---|
| 4B | , | comma | used in DIM and PARAM statements and ON \<expr\> GOTO/GOSUB statements |
| 4C | : | colon | used in DIM, TYPE and PARAM statements to define type |
| 4D | ( | open parenthesis | used in RUN statements |
| 4E | ) | close parenthesis | used in RUN statements |
| 51 | ; | semi-colon | used in TYPE and PARAM statements to concatenate fields of different types |

## String Length Value Tokens

These are used to determine string length in DIM, TYPE and PARAM statements.

| | | |
|---|---|---|
| 4F | [ | open bracket |
| 50 | ] | close bracket |

## Assignment Operator Tokens

| | | | |
|---|---|---|---|
| 52 | := | colon-equals | PASCAL-style assignment operator |
| 53 | = | equals | normal assignment operator |

## File Mode and Path Operator Tokens

| | | | |
|---|---|---|---|
| 4A | : | colon | file access mode |
| 54 | \# | hash | file path |

## Internal Tokens

| | | |
|---|---|---|
| 0E | \<cva\> | complex variable assignment |
| 3A | \<ulref\> | unbound line reference |
| 3B | \<blref\> | bound line reference |
| 3E | \\ | backslash end-of-instruction, continue line |
| 3F | \<eol\> | end-of-line |
| 55 | \<ivgt\> | invisible goto |
| C8 | \<fix1\> | fix top of stack |
| C9 | \<fix2\> | fix second on stack |
| CA | \<fix3\> | fix third on stack |
| CB | \<flt1\> | float top of stack |
| CC | \<flt2\> | float second on stack |

## Numeric Function Tokens

PI is actually a constant. INT() has two forms (AC and AD). Note the difference in the way the level 1 and level 2 manuals define the function:

Level 1: Chapter 8. Expressions, Operators, and Functions: p 54:

```
  INT(<num>) 	truncates all digits to the right of the decimal point of a REAL <num>.
```

Level 2: Chapter 7. Expressions, Operators, and Functions: p 7-8:

```
  INT 	Calculates the largest whole number less than or equal to the specified number.
```

Because of this, I am not sure why there are two versions. Further investigation must be done.

In the following list, the 98,9D and A6 tokens are BYTE/INTEGER versions, and 99,9E and A7 tokens are REAL versions. There are other tokens that reflect this division, but more investigation needs to be done to determine the specifics.

| | | | | | | | | | | | |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 97 | ERR | 98 | MOD() | 99 | MOD() | 9A | RND() | | | | |
| 9B | PI | 9D | SGN() | 9E | SGN() | | | | | | |
| 9F | SIN() | A0 | COS() | A1 | TAN() | A2 | ASN() | A3 | ACS() | A4 | ATN() |
| A5 | EXP() | A6 | ABS() | A7 | ABS() | A8 | LOG() | A9 | LOG10() | | |
| AA | SQRT() | AB | SQR() | B2 | SQ() | B3 | SQ() | AC | INT() | AD | INT() |
| AE | FIX() | AF | FIX() | B0 | FLOAT() | B1 | FLOAT() | | | | |

## Screen/Print Function Tokens

| | |
|---|---|
| 96 | POS() |
| C7 | TAB() |

## File and Other Function Tokens

| | | | |
|---|---|---|---|
| 92 | ADDR() | 93 | \<addr\> |
| 94 | SIZE() | 95 | \<size\> |
| B4 | PEEK() | BE | EOF() |

ADDR() and SIZE() are the only two functions that have two tokens each. In I-Code, the second token appears in the code before the address of the item to be used as the argument to the function, and the function token appears after it.

**reg.x:=ADDR(optionPacket)**
| 0E | 85 000C | F6 0006 | 52 | 93 | F2 0015 | 92 | 3F |
|---|---|---|---|---|---|---|---|
| \<cva\> | reg | .x | := | \<addr\> | optionPacket | ADDR() | \<eol\> |

**0E** = **\<cva\>** (complex variable assignment)<br>
**85 000C** = **reg**<br>
**F6 0006** = **.x**<br>
**52** = **:=** (assignment operator)<br>
**93** = **\<addr\>**<br>
**F2 0015** = **optionPacket**<br>
**92** = **ADDR()**<br>
**3F** = **\<eol\>** (end-of-instruction, newline)

**SEEK \#filePath,numVars\*SIZE(vars)+2**
| 2B | 54 | 80 0252 | 4B | 81 0259 | 95 | F2 0027 | 94 | EC | 8D 02 | E7 | 3F |
|---|---|---|---|---|---|---|---|---|---|---|---|
| SEEK | \# | filePath | , | numVars | \<size\> | vars | SIZE() | \* | \<bc\> 2 | + | \<eol\> |

**2B** = **SEEK**<br>
**54** = **\#**<br>
**80** 0252 = **filePath**<br>
**4B** = **,** (comma)<br>
**81 0259** = **numVars**<br>
**95** = **\<size\>**<br>
**F2 0027** = **vars**<br>
**94** = **SIZE()**<br>
**EC** = **\*** (mult)<br>
**8D 02** = **\<bc\>** 2 (byte constant)<br>
**E7** = **+** (add)<br>
**3F** = **\<eol\>** (end-of-instruction, newline)

## String Function Tokens

| | |
|---|---|
| 9C | SUBSTR() |
| BF | TRIM$() |
| C0 | MID$() |
| C1 | LEFT$() |
| C2 | RIGHT$() |

## String-to-Number Function Tokens

| | |
|---|---|
| B6 | VAL() |
| B7 | LEN() |
| B8 | ASC() |

## Number-to-String Function Tokens

| | | |
|---|---|---|
| C3 | CHR$() | |
| C4 | STR$() | (BYTE/INTEGER) |
| C5 | STR$() | (REAL) |
| C6 | DATE$ | |

## Logical Operator Tokens

| | |
|---|---|
| B5 | LNOT() |
| B9 | LAND() |
| BA | LOR() |
| BB | LXOR() |

## Boolean Function Tokens

| | | |
|---|---|---|
| CD | NOT() | |
| CE | - | (BYTE/INTEGER) |
| CF | - | (REAL) |
| D0 | AND | |
| D1 | OR | |
| D2 | XOR | |

## Comparison Operator Tokens

| BYTE/INTEGER | | REAL | | STRING | | BOOLEAN | |
|:---:|:---|:---:|:---|:---:|:---|:---:|:---|
| D3 | \> | D4 | \> | D5 | \> | | |
| D6 | \< | D7 | \< | D8 | \< | | |
| D9 | \<\> | DA | \<\> | DB | \<\> | DC | \<\> |
| DD | = | DE | = | DF | = | E0 | = |
| E1 | \>= | E2 | \>= | E3 | \>= | | |
| E4 | \<= | E5 | \<= | E6 | \<= | | |

## Math Operator Tokens

| BYTE/INTEGER | | REAL | | STRING | |
|:---:|:---|:---:|:---|:---:|:---|
| E7 | + | E8 | + | E9 | + |
| EA | - | EB | - | | |
| EC | \* | ED | \* | | |
| EE | / | EF | / | | |
| F0 | ^ | F1 | \*\* | | |

## I-Code Token List

| Token | Keyword/Name | Used In | Description |
|---|---|---|---|
| 00 | GLOBAL | Reserved | Global Variable |
| 01 | PARAM | Editor | Parameter Variable statement |
| 01 | READ | I-Code | File Access Mode |
| 02 | TYPE | Editor | User-defined Type statement |
| 02 | WRITE | I-Code | File Access Mode |
| 03 | DIM | Editor | Variable Dimension statement |
| 03 | UPDATE | I-Code | File Access Mode |
| 04 | DATA | I-Code/Editor | Data statement |
| 04 | EXEC | I-Code | File Access Mode |
| 05 | STOP | I-Code/Editor | Stop Execution |
| 05 | READ+EXEC | I-Code | File Access Mode |
| 06 | BYE | I-Code/Editor | End Procedure and Close Basic09 (if in Basic09 Workspace), Display Optional Message |
| 06 | WRITE+EXEC | I-Code | File Access Mode |
| 07 | TRON | Editor/Debug | Trace On |
| 07 | UPDATE+EXEC | I-Code | File Access Mode |
| 08 | TROFF | Editor/Debug | Trace Off |
| 09 | PAUSE | Editor/Debug | Pause Execution |
| 0A | DEG | I-Code/Editor | Set Degrees |
| 0B | RAD | I-Code/Editor | Set Radians |
| 0C | RETURN | I-Code/Editor | Return from GOSUB subroutine |
| 0D | LET | I-Code/Editor | Assignment statement (optional) |
| 0E | cva | I-Code/Editor | Complex Variable Assignment |
| 0F | POKE | I-Code/Editor | Poke value to memory address |
| 10 | IF | I-Code/Editor | Boolean condition test |
| 11 | ELSE | I-Code/Editor | Boolean condition test alternate |
| 12 | ENDIF | I-Code/Editor | Boolean condition test end |
| 13 | FOR | I-Code/Editor | FOR loop start |
| 14 | NEXT | I-Code/Editor | FOR loop end |
| 15 | WHILE | I-Code/Editor | WHILE loop start |
| 16 | ENDWHILE | I-Code/Editor | WHILE loop end |
| 17 | REPEAT | I-Code/Editor | REPEAT loop start |
| 18 | UNTIL | I-Code/Editor | REPEAT loop end |
| 19 | LOOP | I-Code/Editor | Endless LOOP start |
| 1A | ENDLOOP | I-Code/Editor | Endless LOOP end |
| 1B | EXITIF | I-Code/Editor | Exit condition test start |
| 1C | ENDEXIT | I-Code/Editor | Exit condition test end |
| 1D | ON | I-Code/Editor | ON condition |
| 1E | ERROR | I-Code/Editor | ERROR condition or ERROR generation |
| 1F | GOTO | Editor | Unbound Branch to line reference |
| 20 | GOTO | I-Code/Editor | Bound Branch to line reference |
| 21 | GOSUB | Editor | Unbound Branch to subroutine reference |
| 22 | GOSUB | I-Code/Editor | Bound Branch to subroutine reference |
| 23 | RUN | I-Code/Editor | REN external procedure |
| 24 | KILL | I-Code/Editor | KILL external procedure in memory |
| 25 | TYPE var | I-Code/Editor | VDT (Symbol Table) Entry, TYPE variable |
| 25 | INPUT | I-Code/Editor | Line INPUT from user or device |
| 26 | PRINT | I-Code/Editor | PRINT line, ? Becomes PRINT in the Editor |
| 27 | CHD | I-Code/Editor | Change Current Working (data) Directory |
| 28 | CHX | I-Code/Editor | Change Current Execution Directory |
| 29 | CREATE | I-Code/Editor | Create a Disk File |
| 2A | OPEN | I-Code/Editor | Open a Disk File or a path to a Device |
| 2B | SEEK | I-Code/Editor | Move the File Pointer to a Specific Location in a Disk File |
| 2C | READ | I-Code/Editor | Read a Line of Text in a Sequential Disk File or Device, Reads until CR is found or EOF |
| 2D | WRITE | I-Code/Editor | Write a Line of Text to a Sequential Disk File or Device, Adds a CR to the End of the Text Written |
| 2E | GET | I-Code/Editor | Reads a Specified Number of Bytes from a Disk File or Device |
| 2F | PUT | I-Code/Editor | Writes a Specified Number of Bytes to a Disk File or Device |
| 30 | CLOSE | I-Code/Editor | Closes a Path to a Disk File or Device |
| 31 | RESTORE | I-Code/Editor | Set DATA Pointer to the Beginning of the First DATA statement, or the DATA Statement at the Specified Line Number |
| 32 | DELETE | I-Code/Editor | Delete the Specified Disk File |
| 33 | CHAIN | I-Code/Editor | End the Current Procedure and Load/Execute the Named Procedure, Passes Any Parameters Included in the Statement |
| 34 | SHELL | I-Code/Editor | Execute a NitrOS-9 Command from a Shell, Returns Control to the Procedure when the Shell is Exited |
| 35 | BASE0 | I-Code/Editor | Sets the Numerical Base of the Procedure to 0, all Arrays and Structures will begin with Index 0 |
| 36 | BASE1 | I-Code/Editor | This is the Default, Sets the Numerical Base of the Procedure to 1, all Arrays and Structures will begin with Index 1 |
| 37 | REM | Editor | Comment token, \! Becomes REM in the Editor |
| 38 | (\* | Editor | Comment token (there is no token for \*) ) |
| 39 | END | I-Code/Editor | End a Procedure Immediately |
| 3A | ulref | Editor | Unbound Line Reference |
| 3B | blref | I-Code/Editor | Bound Line Reference |
| 3C | dex | Unimplemented | Direct Execution |
| 3D | PROCEDURE | Editor | Procedure start |
| 3D | erl | Editor/Debug | Error Line |
| 3E | \\ | I-Code/Editor | End-of-Instruction, Continue Line |
| 3F | eol | I-Code/Editor | End-of-Instruction and Line |
| 40 | BYTE | Editor | Atomic Type Identifier, used in TYPE, DIM and Param Statements |
| 40 | f\_byte | I-Code/Editor | VDT (Symbol Table) Entry, Field Atomic Byte Variable |
| 41 | INTEGER | Editor | Atomic Type Identifier, used in TYPE, DIM and Param Statements |
| 41 | f\_integer | I-Code/Editor | VDT (Symbol Table) Entry, Field Atomic Integer Variable |
| 42 | REAL | Editor | Atomic Type Identifier, used in TYPE, DIM and Param Statements |
| 42 | f\_real | I-Code/Editor | VDT (Symbol Table) Entry, Field Atomic Real Variable |
| 43 | BOOLEAN | Editor | Atomic Type Identifier, used in TYPE, DIM and Param Statements |
| 43 | f\_boolean | I-Code/Editor | VDT (Symbol Table) Entry, Field Atomic Boolean Variable |
| 44 | STRING | Editor | Atomic Type Identifier, used in TYPE, DIM and Param Statements |
| 44 | f\_string | I-Code/Editor | VDT (Symbol Table) Entry, Field Atomic String Variable |
| 45 | THEN | I-Code/Editor | |
| 45 | f\_record | I-Code/Editor | VDT (Symbol Table) Entry, Field Record Variable |
| 46 | TO | I-Code/Editor | Minimum or Maximum Value of the Iteration Count in a FOR/NEXT Loop |
| 47 | STEP | I-Code/Editor | Value to be Incremented/Decremented for Each Iteration of a FOR/NEXT Loop |
| 48 | DO | I-Code/Editor | |
| 48 | f\_vector\_b | I-Code/Editor | VDT (Symbol Table) Entry, Field 1 Dimensional Byte Array |
| 49 | USING | I-Code/Editor | Format the Output in a PRINT Statement |
| 49 | f\_vector\_i | I-Code/Editor | VDT (Symbol Table) Entry, Field 1 Dimensional Integer Array |
| 4A | : | I-Code/Editor | File Access Mode Operator |
| 4A | f\_vector\_r | I-Code/Editor | VDT (Symbol Table) Entry, Field 1 Dimensional Real Array |
| 4B | , | I-Code/Editor | Comma Separator |
| 4B | f\_vector\_l | I-Code/Editor | VDT (Symbol Table) Entry, Field 1 Dimensional Boolean Array |
| 4C | : | I-Code/Editor | Colon |
| 4C | f\_vector\_s | I-Code/Editor | VDT (Symbol Table) Entry, Field 1 Dimensional String Array |
| 4D | ( | I-Code/Editor | Left Parenthesis |
| 4D | f\_vector\_c | I-Code/Editor | VDT (Symbol Table) Entry, Field 1 Dimensional Record Array |
| 4E | ) | I-Code/Editor | Right Parenthesis |
| 4F | [ | I-Code/Editor | Left Bracket |
| 50 | ] | I-Code/Editor | Right Bracket |
| 50 | f\_table\_b | I-Code/Editor | VDT (Symbol Table) Entry, Field 2 Dimensional Byte Array |
| 51 | ; | I-Code/Editor | Semi-colon |
| 51 | f\_table\_i | I-Code/Editor | VDT (Symbol Table) Entry, Field 2 Dimensional Integer Array |
| 52 | := | I-Code/Editor | Assignment Operator |
| 52 | f\_table\_r | I-Code/Editor | VDT (Symbol Table) Entry, Field 2 Dimensional Real Array |
| 53 | = | I-Code/Editor | Assignment Operator |
| 53 | f\_table\_l | I-Code/Editor | VDT (Symbol Table) Entry, Field 2 Dimensional Boolean Array |
| 54 | \# | I-Code/Editor | Channel (Path) Number Operator |
| 54 | f\_table\_s | I-Code/Editor | VDT (Symbol Table) Entry, Field 2 Dimensional String Array |
| 55 | ivgt | I-Code/Editor | Invisible GOTO (used in many statements) |
| 55 | ivgt | I-Code/Editor | Invisible GOTO (special case in ON var GOTO/GOSUB statements) |
| 55 | f\_table\_c | I-Code/Editor | VDT (Symbol Table) Entry, Field 2 Dimensional Record Array |
| 56 | Unused | | |
| 57 | Unused | | |
| 58 | f\_matrix\_b | I-Code/Editor | VDT (Symbol Table) Entry, Field 3 Dimensional Byte Array |
| 59 | f\_matrix\_i | I-Code/Editor | VDT (Symbol Table) Entry, Field 3 Dimensional Integer Array |
| 5A | f\_matrix\_r | I-Code/Editor | VDT (Symbol Table) Entry, Field 3 Dimensional Real Array |
| 5B | f\_matrix\_l | I-Code/Editor | VDT (Symbol Table) Entry, Field 3 Dimensional Boolean Array |
| 5C | f\_matrix\_s | I-Code/Editor | VDT (Symbol Table) Entry, Field 3 Dimensional String Array |
| 5D | f\_matrix\_c | I-Code/Editor | VDT (Symbol Table) Entry, Field 3 Dimensional Record Array |
| 5E | Unused | | |
| 5F | Unused | | |
| 60 | byte | I-Code/Editor | VDT (Symbol Table) Entry, Atomic Byte Variable |
| 61 | integer | I-Code/Editor | VDT (Symbol Table) Entry, Atomic Integer Variable |
| 62 | real | I-Code/Editor | VDT (Symbol Table) Entry, Atomic Real Variable |
| 63 | boolean | I-Code/Editor | VDT (Symbol Table) Entry, Atomic Boolean Variable |
| 64 | string | I-Code/Editor | VDT (Symbol Table) Entry, Atomic String Variable |
| 65 | record | I-Code/Editor | VDT (Symbol Table) Entry, Record Variable |
| 66 | Unused | | |
| 67 | Unused | | |
| 68 | vector\_b | I-Code/Editor | VDT (Symbol Table) Entry, 1 Dimensional Byte Array |
| 69 | vector\_i | I-Code/Editor | VDT (Symbol Table) Entry, 1 Dimensional Integer Array |
| 6A | vector\_r | I-Code/Editor | VDT (Symbol Table) Entry, 1 Dimensional Real Array |
| 6B | vector\_l | I-Code/Editor | VDT (Symbol Table) Entry, 1 Dimensional Boolean Array |
| 6C | vector\_s | I-Code/Editor | VDT (Symbol Table) Entry, 1 Dimensional String Array |
| 6D | vector\_c | I-Code/Editor | VDT (Symbol Table) Entry, 1 Dimensional Record Array |
| 6E | Unused | | |
| 6F | Unused | | |
| 70 | table\_b | I-Code/Editor | VDT (Symbol Table) Entry, 2 Dimensional Byte Array |
| 71 | table\_i | I-Code/Editor | VDT (Symbol Table) Entry, 2 Dimensional Integer Array |
| 72 | table\_r | I-Code/Editor | VDT (Symbol Table) Entry, 2 Dimensional Real Array |
| 73 | table\_l | I-Code/Editor | VDT (Symbol Table) Entry, 2 Dimensional Boolean Array |
| 74 | table\_s | I-Code/Editor | VDT (Symbol Table) Entry, 2 Dimensional String Array |
| 75 | table\_c | I-Code/Editor | VDT (Symbol Table) Entry, 2 Dimensional Record Array |
| 76 | Unused | | |
| 77 | Unused | | |
| 78 | matrix\_b | I-Code/Editor | VDT (Symbol Table) Entry, 3 Dimensional Byte Array |
| 79 | matrix\_i | I-Code/Editor | VDT (Symbol Table) Entry, 3 Dimensional Integer Array |
| 7A | matrix\_r | I-Code/Editor | VDT (Symbol Table) Entry, 3 Dimensional Real Array |
| 7B | matrix\_l | I-Code/Editor | VDT (Symbol Table) Entry, 3 Dimensional Boolean Array |
| 7C | matrix\_s | I-Code/Editor | VDT (Symbol Table) Entry, 3 Dimensional String Array |
| 7D | matrix\_c | I-Code/Editor | VDT (Symbol Table) Entry, 3 Dimensional Record Array |
| 7E | Unused | | |
| 7F | Unused | | |
| 80 | byte | I-Code/Editor | Instruction, Atomic Byte Variable |
| 80 | p\_byte | I-Code/Editor | VDT (Symbol Table) Entry, Parameter Atomic Byte Variable |
| 80 | DIR | I-Code | File Access Mode |
| 81 | integer | I-Code/Editor | Instruction, Atomic Integer Variable |
| 81 | p\_integer | I-Code/Editor | VDT (Symbol Table) Entry, Parameter Atomic Integer Variable |
| 81 | READ+DIR | I-Code | File Access Mode |
| 82 | real | I-Code/Editor | Instruction, Atomic Real Variable |
| 82 | p\_real | I-Code/Editor | VDT (Symbol Table) Entry, Parameter Atomic Real Variable |
| 82 | WRITE+DIR | I-Code | File Access Mode |
| 83 | boolean | I-Code/Editor | Instruction, Atomic Boolean Variable |
| 83 | p\_boolean | I-Code/Editor | VDT (Symbol Table) Entry, Parameter Atomic Boolean Variable |
| 83 | UPDATE+DIR | I-Code | File Access Mode |
| 84 | string | I-Code/Editor | Instruction, Atomic String Variable |
| 84 | p\_string | I-Code/Editor | VDT (Symbol Table) Entry, Parameter Atomic String Variable |
| 84 | EXEC+DIR | I-Code | File Access Mode |
| 85 | record | I-Code/Editor | Instruction, Record Variable |
| 85 | p\_atomic | I-Code/Editor | Instruction, Parameter Atomic Variable |
| 85 | p\_record | I-Code/Editor | Instruction, Parameter Record Variable |
| 85 | p\_record | I-Code/Editor | VDT (Symbol Table) Entry, Parameter Record Variable |
| 85 | vector | I-Code/Editor | Instruction, 1 Dimensional Array Variable |
| 85 | p\_vector | I-Code/Editor | Instruction, Parameter 1 Dimensional Array Variable |
| 85 | table | I-Code/Editor | Instruction, 2 Dimensional Array Variable |
| 85 | p\_table | I-Code/Editor | Instruction, Parameter 2 Dimensional Array Variable |
| 85 | matrix | I-Code/Editor | Instruction, 3 Dimensional Array Variable |
| 85 | p\_matrix | I-Code/Editor | Instruction, Parameter 3 Dimensional Array Variable |
| 85 | READ+EXEC+DIR | I-Code | File Access Mode |
| 86 | vector | I-Code/Editor | Instruction, 1 Dimensional Array |
| 86 | p\_vector | I-Code/Editor | Instruction, Parameter 1 Dimensional Array |
| 86 | WRITE+EXEC+DIR | I-Code | File Access Mode |
| 87 | table | I-Code/Editor | Instruction, 2 Dimensional Array |
| 87 | p\_table | I-Code/Editor | Instruction, Parameter 2 Dimensional Array |
| 87 | UPDATE+EXEC+DIR | I-Code | File Access Mode |
| 88 | matrix | I-Code/Editor | Instruction, 3 Dimensional Array |
| 88 | p\_matrix | I-Code/Editor | Instruction, Parameter 3 Dimensional Array |
| 88 | p\_vector\_b | I-Code/Editor | VDT (Symbol Table) Entry, Parameter 1 Dimensional Byte Array |
| 89 | atomic\_m | I-Code/Editor | Instruction, Atomic Variable Mirror |
| 89 | record\_m | I-Code/Editor | Instruction, Record Variable Mirror |
| 89 | f\_atomic\_m | I-Code/Editor | Instruction, Field Atomic Variable Mirror |
| 89 | f\_record\_m | I-Code/Editor | Instruction, Field Record Variable Mirror |
| 89 | p\_vector\_i | I-Code/Editor | VDT (Symbol Table) Entry, Parameter 1 Dimensional Integer Array |
| 8A | f\_vector\_m | I-Code/Editor | Instruction, Field 1 Dimensional Array Mirror |
| 8A | p\_vector\_r | I-Code/Editor | VDT (Symbol Table) Entry, Parameter 1 Dimensional Real Array |
| 8B | f\_table\_m | I-Code/Editor | Instruction, Field 2 Dimensional Array Mirror |
| 8B | p\_vector\_l | I-Code/Editor | VDT (Symbol Table) Entry, Parameter 1 Dimensional Boolean Array |
| 8C | f\_matrix\_m | I-Code/Editor | Instruction, Field 3 Dimensional Array Mirror |
| 8C | p\_vector\_s | I-Code/Editor | VDT (Symbol Table) Entry, Parameter 1 Dimensional String Array |
| 8D | blit | I-Code/Editor | BYTE Constant (Literal) |
| 8D | p\_vector\_c | I-Code/Editor | VDT (Symbol Table) Entry, Parameter 1 Dimensional Record Array |
| 8E | ilit | I-Code/Editor | INTEGER Constant (Literal) |
| 8F | rlit | I-Code/Editor | REAL Constant (Literal) |
| 90 | " | I-Code/Editor | STRING Constant (Literal) - Beginning |
| 90 | p\_table\_b | I-Code/Editor | VDT (Symbol Table) Entry, Parameter 2 Dimensional Byte Array |
| 91 | $ | I-Code/Editor | Hexadecimal Constant (Literal) |
| 91 | p_table_i | I-Code/Editor | VDT (Symbol Table) Entry, Parameter 2 Dimensional Integer Array |
| 92 | ADDR() | I-Code/Editor | |
| 92 | p_table_r | I-Code/Editor | VDT (Symbol Table) Entry, Parameter 2 Dimensional Real Array |
| 93 | addr | I-Code/Editor | Second Byte of ADDR() |
| 93 | p_table_l | I-Code/Editor | VDT (Symbol Table) Entry, Parameter 2 Dimensional Boolean Array |
| 94 | SIZE() | I-Code/Editor | |
| 94 | p_table_s | I-Code/Editor | VDT (Symbol Table) Entry, Parameter 2 Dimensional String Array |
| 95 | size | I-Code/Editor | Second Byte of SIZE() |
| 95 | p_table_c | I-Code/Editor | VDT (Symbol Table) Entry, Parameter 2 Dimensional Record Array |
| 96 | POS() | I-Code/Editor | |
| 97 | ERR | I-Code/Editor | |
| 98 | MOD() | I-Code/Editor | Byte/Integer |
| 98 | p_matrix_b | I-Code/Editor | VDT (Symbol Table) Entry, Parameter 3 Dimensional Byte Array |
| 99 | MOD() | I-Code/Editor | Real |
| 99 | p_matrix_i | I-Code/Editor | VDT (Symbol Table) Entry, Parameter 3 Dimensional Integer Array |
| 9A | RND() | I-Code/Editor | |
| 9A | p_matrix_r | I-Code/Editor | VDT (Symbol Table) Entry, Parameter 3 Dimensional Real Array |
| 9B | PI | I-Code/Editor | |
| 9B | p_matrix_l | I-Code/Editor | VDT (Symbol Table) Entry, Parameter 3 Dimensional Boolean Array |
| 9C | SUBSTR() | I-Code/Editor | |
| 9C | p_matrix_s | I-Code/Editor | VDT (Symbol Table) Entry, Parameter 3 Dimensional String Array |
| 9D | SGN() | I-Code/Editor | |
| 9D | p_matrix_c | I-Code/Editor | VDT (Symbol Table) Entry, Parameter 3 Dimensional Record Array |
| 9E | SGN() | I-Code/Editor | |
| 9F | SIN() | I-Code/Editor | |
| A0 | COS() | I-Code/Editor | |
| A0 | subr | I-Code/Editor | Named Subroutine |
| A1 | TAN() | I-Code/Editor | |
| A2 | ASN() | I-Code/Editor | |
| A3 | ACS() | I-Code/Editor | |
| A4 | ATN() | I-Code/Editor | |
| A5 | EXP() | I-Code/Editor | |
| A6 | ABS() | I-Code/Editor | |
| A7 | ABS() | I-Code/Editor | |
| A8 | LOG() | I-Code/Editor | |
| A9 | LOG10() | I-Code/Editor | |
| AA | SQRT() | I-Code/Editor | |
| AB | SQR() | I-Code/Editor | Becomes SQRT() in the Code |
| AC | INT() | I-Code/Editor | Byte/Integer |
| AD | INT() | I-Code/Editor | Real |
| AE | FIX() | I-Code/Editor | Byte/Integer |
| AF | FIX() | I-Code/Editor | Real |
| B0 | FLOAT() | I-Code/Editor | Byte/Integer |
| B1 | FLOAT() | I-Code/Editor | Real |
| B2 | SQ() | I-Code/Editor | Byte/Integer |
| B3 | SQ() | I-Code/Editor | Real |
| B4 | PEEK() | I-Code/Editor | |
| B5 | LNOT() | I-Code/Editor | Logical NOT |
| B6 | VAL() | I-Code/Editor | |
| B7 | LEN() | I-Code/Editor | |
| B8 | ASC() | I-Code/Editor | |
| B9 | LAND() | I-Code/Editor | Logical AND |
| BA | LOR() | I-Code/Editor | Logical OR |
| BB | LXOR() | I-Code/Editor | Logical XOR |
| BC | TRUE | I-Code/Editor | |
| BD | FALSE | I-Code/Editor | |
| BE | EOF() | I-Code/Editor | |
| BF | TRIM$() | I-Code/Editor | |
| C0 | MID$() | I-Code/Editor | |
| C1 | LEFT$() | I-Code/Editor | |
| C2 | RIGHT$() | I-Code/Editor | |
| C3 | CHR$() | I-Code/Editor | |
| C4 | STR$() | I-Code/Editor | Byte/Integer |
| C5 | STR$() | I-Code/Editor | Real |
| C6 | DATE$ | I-Code/Editor | |
| C7 | TAB | I-Code/Editor | |
| C8 | ritc | I-Code/Editor | Real-Byte/Integer Type Conversion |
| C8 | fix1 | I-Code/Editor | Fix Top of Stack |
| C9 | fix2 | I-Code/Editor | Fix Second on Stack |
| CA | fix3 | I-Code/Editor | Fix Third on Stack |
| CB | irtc | I-Code/Editor | Byte/Integer-Real Type Conversion |
| CB | flt1 | I-Code/Editor | Float Top of Stack |
| CC | flt2 | I-Code/Editor | Float Second on Stack |
| CD | NOT() | I-Code/Editor | |
| CE | - | I-Code/Editor | (Monadic) Negate Byte/Integer |
| CF | - | I-Code/Editor | (Monadic) Negate Real |
| D0 | AND | I-Code/Editor | |
| D1 | OR | I-Code/Editor | |
| D2 | XOR | I-Code/Editor | |
| D3 | \> | I-Code/Editor | Byte/Integer Comparison Operator |
| D4 | \> | I-Code/Editor | Real Comparison Operator |
| D5 | \> | I-Code/Editor | String Comparison Operator |
| D6 | \< | I-Code/Editor | Byte/Integer Comparison Operator |
| D7 | \< | I-Code/Editor | Real Comparison Operator |
| D8 | \< | I-Code/Editor | String Comparison Operator |
| D9 | \<\> | I-Code/Editor | Byte/Integer Comparison Operator \>\< is converted to \<\> in the code |
| DA | \<\> | I-Code/Editor | Real Comparison Operator \>\< is converted to \<\> in the code |
| DB | \<\> | I-Code/Editor | String Comparison Operator \>\< is converted to \<\> in the code |
| DC | \<\> | I-Code/Editor | Boolean Comparison Operator \>\< is converted to \<\> in the code |
| DD | = | I-Code/Editor | Byte/Integer Comparison Operator |
| DE | = | I-Code/Editor | Real Comparison Operator |
| DF | = | I-Code/Editor | String Comparison Operator |
| E0 | = | I-Code/Editor | Boolean Comparison Operator |
| E1 | \>= | I-Code/Editor | Byte/Integer Greater/Equal Operator |
| E2 | \>= | I-Code/Editor | Real Greater/Equal Operator |
| E3 | \>= | I-Code/Editor | String Greater/Equal Operator |
| E4 | \<= | I-Code/Editor | Byte/Integer Less/Equal Operator |
| E5 | \<= | I-Code/Editor | Real Less/Equal Operator |
| E6 | \<= | I-Code/Editor | String Less/Equal Operator |
| E7 | + | I-Code/Editor | Byte/Integer Add Operator |
| E8 | + | I-Code/Editor | Real Add Operator |
| E9 | + | I-Code/Editor | String Concatenate Operator |
| EA | - | I-Code/Editor | Byte/Integer Subtract Operator (Dyadic) |
| EB | - | I-Code/Editor | Real Subtract Operator (Dyadic) |
| EC | \* | I-Code/Editor | Byte/Integer Multiply Operator |
| ED | \* | I-Code/Editor | Real Multiply Operator |
| EE | / | I-Code/Editor | Byte/Integer Divide Operator |
| EF | / | I-Code/Editor | Real Divide Operator |
| F0 | ^ | I-Code/Editor | Exponent Operator |
| F1 | \*\* | I-Code/Editor | Exponent Operator |
| F2 | atomic\_m | I-Code/Editor | Instruction, Atomic Variable Mirror |
| F2 | record\_m | I-Code/Editor | Instruction, Record Variable Mirror |
| F2 | p\_atomic\_m | I-Code/Editor | Instruction, Parameter Atomic Variable Mirror |
| F2 | p\_record\_m | I-Code/Editor | Instruction, Parameter Record Variable Mirror |
| F3 | vector\_m | I-Code/Editor | Instruction, 1 Dimensional Array Mirror |
| F3 | p\_vector\_m | I-Code/Editor | Instruction, Parameter 1 Dimensional Array Mirror |
| F4 | table\_m | I-Code/Editor | Instruction, 2 Dimensional Array Mirror |
| F4 | p\_table\_m | I-Code/Editor | Instruction, Parameter 2 Dimensional Array Mirror |
| F5 | matrix\_m | I-Code/Editor | Instruction, 3 Dimensional Array Mirror |
| F5 | p\_matrix\_m | I-Code/Editor | Instruction, Parameter 3 Dimensional Array Mirror |
| F6 | f\_atomic | I-Code/Editor | Instruction, Field Atomic Variable |
| F6 | f\_record | I-Code/Editor | Instruction, Field Record Variable |
| F7 | UPDATE | Editor | File Access Mode |
| F7 | f\_vector | I-Code/Editor | Instruction, Field 1 Dimensional Array |
| F8 | EXEC | Editor | File Access Mode |
| F8 | f\_table | I-Code/Editor | Instruction, Field 2 Dimensional Array |
| F9 | DIR | Editor | File Access Mode |
| F9 | f\_matrix | I-Code/Editor | Instruction, Field 3 Dimensional Array |
| FA | Unused | | |
| FB | Unused | | |
| FC | Unused | | |
| FD | Unused | | |
| FE | Unused | | |
| FF | " | I-Code/Editor | STRING Constant (Literal) - Terminator |
