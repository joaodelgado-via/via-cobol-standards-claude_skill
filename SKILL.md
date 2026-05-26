---
name: via-cobol
description: >
  VIA COBOL code generation skill. Use this skill whenever the user asks to write, generate,
  create, or modify any COBOL code, programs, modules, copybooks, CL scripts, or any IBM i
  artefacts. Also trigger when the user mentions COBOL in the context of batch programs,
  online programs, DB2 access, file handling, module calls, or IBM i development — even if
  they don't say "skill" or "standard". This skill ensures every piece of generated code
  follows VIA Consulting's naming conventions, structural patterns, error-handling rules,
  and coding style. Never generate COBOL without consulting this skill first.
---

# VIA COBOL Generation Standards

You are generating COBOL/IBM i code for VIA Consulting. Every artefact you produce — programs,
modules, copybooks, CL scripts, DDS — must comply with the rules below. When in doubt, apply
the rule; do not invent alternatives.

---

## 1. Object Naming — Always 8 Characters

```
XX  T  ·····
│   │   │
│   │   └── Identifier (varies by type — see tables below)
│   └────── Type code
└────────── Application code (e.g. PS = Pagamento de Salários)
```

The user will tell you the application code (`XX`). Ask for it if it is not provided.

### 1.1 Type Codes

| Code | Object |
|------|--------|
| `I`  | Online (Interactive) program |
| `B`  | Batch program |
| `M`  | Module (callable subprogram) |
| `C`  | Copybook |
| `L`  | CL program |
| `F`  | Physical file |
| `L`  | Logical file |
| `T`  | DB2 table |
| `VT` | DB2 view |

### 1.2 Online Programs — `XXTOOOAA`

`OOO` = option code, `AA` = action code.

| `AA` | Meaning |
|------|---------|
| `01` | Validate request |
| `02` | Query |
| `03` | Validate execution |
| `04` | Insert |
| `06` | Modify |
| `08` | Delete |
| `00` | List |
| `9AA`| List query (special case: `0AA = 900`) |

### 1.3 Batch Programs — `XXTNSSSS`

| `N` | Reserved for |
|-----|--------------|
| `4` | Scheduled chains (rotinadas) |
| `8` | Table-alteration programs (`XXB8###S`) |
| `9` | Table-purge programs (`XXB9###S`) |

`SSSS = 0000` is reserved for the main chain entry point.

### 1.4 Copybooks — `XXTCSSSS`

| Category (`C`) | Holds |
|----------------|-------|
| `K` | Constants |
| `L` | Linkage |
| `W` | Working storage |
| `P` | Procedure (callable paragraphs) |
| `V` | Online I/O messages |
| `D` | DCLGEN (DB2-generated table declarations) |
| `R` | Routine — CALL boilerplate |
| `F` | File record layout |

Special constants copybooks:
- `XXCK0001` = error messages
- `XXCK0002` = general constants
- `XXCK0003` = core/Banka errors

Online message copybooks: `XXCVSS00` = input mapping, `XXCVSS01` = output mapping.

### 1.5 CL Programs — `XXLNSSSS`

| `N` | Reserved for |
|-----|--------------|
| `4` | Main chain (`SSSS=0000`) |
| `7` | Generic batches |
| `9` | Purge chains |

Table-alteration CL: `XXL7C###` (e.g. `PSL7C001` for table `PST01`).

### 1.6 DB2 Tables and Views

Pattern: `XXTSS_descriptivo`

- `PGT01` = table 01
- `PGVT0101` = view 01 over table 01
- `00` sequence is reserved for indexes

### 1.7 Field Naming — `T·····`

| `T` | Type |
|-----|------|
| `Z` | Sequence (numeric key) |
| `N` | Numeric |
| `S` | Saldo / balance (15,2) |
| `M` | Montante / amount (13,2) |
| `C` | Code (alphanumeric) |
| `G` | Description / text |
| `D` | Date `X(10)` |
| `H` | Timestamp `X(26)` |
| `T` | Rate (3,5) |
| `O` | Time |

---

## 2. Standard Program Structure

Every program has exactly three top-level paragraphs:

```
1000-INICIO       Initialization & input validation
2000-PROCESSA     Main business logic — must be readable at a glance
3000-FIM          Cleanup: COMMIT/ROLLBACK, file closes
```

Sub-paragraphs use the same prefix number as the top-level they belong to (e.g. `2100-VALIDA-DADOS`, `2200-GRAVA-TABELA`).

### 2.1 Paragraph Invocation Rule

Every paragraph call must use `THRU` with a matching exit label, always:

```cobol
PERFORM 2100-VALIDA-DADOS THRU 2100-VALIDA-DADOS-EXIT.
```

Each paragraph ends with:

```cobol
2100-VALIDA-DADOS-EXIT.
    EXIT.
```

### 2.2 Guard Pattern

Every paragraph (after the very first init paragraph) must be guarded by an error check before doing any work:

```cobol
* Online:
IF VIAL0002-SEM-ERROS
   PERFORM 2100-VALIDA-DADOS THRU 2100-VALIDA-DADOS-EXIT
END-IF.

* Batch:
IF AQCW0009-SEM-ERROS
   PERFORM 2100-VALIDA-DADOS THRU 2100-VALIDA-DADOS-EXIT
END-IF.
```

### 2.3 One Paragraph = One Action

Each paragraph does exactly one of the following:
- One SQL access (SELECT / INSERT / UPDATE / DELETE)
- One CALL to a module
- One business validation

This keeps `2000-PROCESSA` readable as a high-level narrative of what the program does.

### 2.4 Debug File

Every paragraph starts by writing to the debug file `VIAF0015`, guarded by a debug flag:

```cobol
2100-VALIDA-DADOS.
    IF SW-DEBUG
       PERFORM VIAP0016-DEBUG THRU VIAP0016-DEBUG-EXIT
    END-IF.
    ...
```

---

## 3. Coding Style

- **Indentation**: Logic starts at column 20. Each `IF`/`EVALUATE` level adds 3 spaces. `TO` clauses align at column 52.
- **Quotes**: Always use single quotes (apostrophes / plicas). Never double quotes — required for mainframe portability.
- **Variables**: All declared under a single `01` level in WORKING-STORAGE.
  - Names ≤ 10 characters.
  - `CON-` prefix for constants.
  - `WS-` prefix for working variables.
  - `SW-` prefix for switches/flags.
- **Initial values**: Every declared variable must have `VALUE` (`SPACES` or `ZEROES`).
- **Paragraph names**: Describe functionality, not technical action. Use `2100-VALIDA-CLIENTE`, not `2100-EXEC-SQL-SELECT`.

---

## 4. Working Storage Declarations

```cobol
WORKING-STORAGE SECTION.
01 WS-VARIAVEIS.
   05 WS-CONTADOR        PIC 9(05)  VALUE ZEROES.
   05 WS-DESCRICAO       PIC X(40)  VALUE SPACES.
   05 SW-DEBUG           PIC X(01)  VALUE SPACES.
      88 SW-DEBUG-ON     VALUE 'S'.
   05 CON-VERSAO         PIC X(08)  VALUE 'V1.0.0  '.
```

---

## 5. Error Handling

### 5.1 Status Variables per Resource

Never reuse a `SQLCODE` or `FILESTATUS` across multiple tables/files. Declare one status variable per resource:

```cobol
01 WS-INT0100-STATUS  PIC S9(09) VALUE ZEROS.
   88 WS-INT0100-ST-OK     VALUE +0.
   88 WS-INT0100-ST-NOTFND VALUE +100.
```

### 5.2 Context-Specific Error Copybooks

| Context    | DB2 check                      | File check                     | Error indicator |
|------------|-------------------------------|-------------------------------|-----------------|
| **Online** | `VIAP0003` / `VIAW0005`        | `VIAP0011` / `VIAW0011`        | `VIAL0002`      |
| **Batch**  | `VIAP0012` / `VIAW0012`        | `VIAP0011` / `VIAW0011`        | `AQCW0009`      |
| **Module** | EVALUATE on `DB2-OK`/`DB2-DUPREC` | (uses `ETCL0002`)           | `ETCL0002`      |

### 5.3 File Status

Always check `FILESTATUS`:
- `'00'` = OK
- `'10'` = EOF
- Anything else = error → route through the appropriate error handler

---

## 6. Module Calls — Required Pattern

```cobol
* Before the CALL — initialize standard area (online or batch as applicable):
* Online: PERFORM ETCP0002-FORM-ONLINE-IN THRU ETCP0002-FORM-ONLINE-IN-EXIT.
* Batch:  PERFORM ETCP0003-STD-INIT       THRU ETCP0003-STD-INIT-EXIT.

CALL ARCL0110-ROTINA USING ETCL0002-COMMAREA
                           ARCL0110-DATA
   ON EXCEPTION
      SET  ETCL0002-ERROS-ARQ   TO TRUE
      MOVE VIAK0001-PROG-NOTFND TO ETCL0002-NR-ERRO
      MOVE ETCL0002-IND-ERROS   TO ETCL0002-CAPLIC-ERRO
      MOVE ARCL0110-ROTINA      TO ETCL0002-CPROGRAM.

* After the CALL:
* Online: PERFORM ETCP0002-STD-END THRU ETCP0002-STD-END-EXIT.
* Batch:  PERFORM ETCP0003-STD-END THRU ETCP0003-STD-END-EXIT.
```

Rules:
- `ETCL0002` **always goes first** in the USING list.
- Standard BANKA fields (`PGMC`, `USERC`, `PGMQC`, `RCODE`, …) live in `ETCL0002`, never in the module-specific copybook.
- Always handle `ON EXCEPTION` — never leave a CALL without it.

---

## 7. Batch Program Mandatory Frame

### 7.1 1000-INICIO

```cobol
1000-INICIO.
    IF AQCW0009-SEM-ERROS
       PERFORM AQCP0100-REG-INICIO    THRU AQCP0100-REG-INICIO-EXIT
    END-IF.
    IF AQCW0009-SEM-ERROS
       PERFORM AQCP0100-OBT-DADOS-ARQ THRU AQCP0100-OBT-DADOS-ARQ-EXIT
    END-IF.
    ...
1000-INICIO-EXIT.
    EXIT.
```

### 7.2 3000-FIM

```cobol
3000-FIM.
    IF AQCW0009-SEM-ERROS
       PERFORM AQCP0100-REG-FIM       THRU AQCP0100-REG-FIM-EXIT
    END-IF.
    * Unconditional — always run after closes:
    PERFORM AQCP0009-TRATA-RCODE      THRU AQCP0009-TRATA-RCODE-EXIT.
    PERFORM AQCP0002-TRATA-SYNCPOINT  THRU AQCP0002-TRATA-SYNCPOINT-EXIT.
3000-FIM-EXIT.
    EXIT.
```

`AQCW0009-RCODE` values: `0` = OK · `1` = Warning · `2` = Functional Error · `3` = Fatal Error.

---

## 8. Online Frame — Director Pattern

- `VIA00100` is the front-controller; it reads the fixed part of the program name and dispatches.
- All online programs receive `VIAL0001` (standard linkage) and `VIAL0002` (error indicator).
- Every PERFORM in an online program is guarded by `IF VIAL0002-SEM-ERROS`.
- Debug routine: `VIAP0016`. Debug file: `VIAF0015`.
- I/O copybooks: `XXCVSS00` (input mapping) and `XXCVSS01` (output mapping).

---

## 9. COPY / COPYBOOK Usage

Include copybooks with:

```cobol
COPY XXCK0001.   *> Error constants
COPY XXCK0002.   *> General constants
COPY ARCL0110R.  *> Module CALL boilerplate (category R)
```

Always use the `COPY` verb (not inline declarations) for:
- Shared constants
- Linkage definitions
- DCLGEN table declarations
- Module CALL boilerplate

---

## 10. DB2 Access Pattern

```cobol
EXEC SQL
    SELECT INT-CNOME
      INTO :WS-NOME
      FROM PST01_CLIENTES
     WHERE INT-CCHAVE = :WS-CHAVE
END-EXEC.

EVALUATE TRUE
   WHEN WS-INT0100-ST-OK
      ... process result ...
   WHEN WS-INT0100-ST-NOTFND
      SET VIAL0002-ERRO-NEGOCIO TO TRUE
      MOVE VIAK0001-REG-NOTFND TO VIAL0002-NR-ERRO
   WHEN OTHER
      PERFORM VIAP0003-DB2-ERRO THRU VIAP0003-DB2-ERRO-EXIT
END-EVALUATE.
```

`SQLCODE` reference: `0` = OK · `+100` = not found · negative = error.

---

## 11. Banka / Core Encapsulation

- `ETR00XXX` — direct Banka call
- `ETR80XXX` — Banka call wrapped with VIA linkage on top of `ETR00XXX`
- Every program/routine invoked with BANKA standard area fields: `PGMC`, `TTR`, `JOBNC`, `USERC`, `PGMQC`, `RCODE`, `ESTR`, `APTR` — followed by its own fields.
- Only the entry point (front-end or batch main program) formats these fields. Intermediate programs pass them through unchanged.

---

## 12. Libraries (IBM i Source Physical Files)

| Library      | Contents |
|--------------|----------|
| `QCBLSRC`    | COBOL programs |
| `QCBLLESRC`  | Copybooks |
| `QDDSRC`     | DDS — table/file definitions |
| `QPRTFSRC`   | Screen maps |
| `QCLSRC`     | CL programs |

Compiler: `COBCMP` · Create command: `CRTSQLCBLI` · PDM actions: `CO` (quality check), `DV` (dev compile).

---

## 13. Quick Checklist Before Finalising Generated Code

Before presenting any generated COBOL artefact, verify:

- [ ] Object name is exactly 8 characters and follows the `XX T ···` pattern
- [ ] All variables prefixed (`WS-`, `SW-`, `CON-`) and initialised with `VALUE`
- [ ] Single quotes used throughout — no double quotes
- [ ] Every paragraph ends with `-EXIT. EXIT.`
- [ ] Every paragraph call uses `PERFORM … THRU …-EXIT.`
- [ ] All paragraphs (except the first init one) are guarded by the appropriate `SEM-ERROS` condition
- [ ] Debug write to `VIAF0015` at the start of every paragraph
- [ ] Each paragraph does exactly one action (one SQL / one CALL / one validation)
- [ ] Module CALLs follow the required pattern with `ON EXCEPTION`
- [ ] `ETCL0002` is first in every USING list
- [ ] Batch programs include the mandatory `AQCP0100` and `AQCP0002`/`AQCP0009` frame calls
- [ ] DB2 status evaluated with one 88-level variable per table — never reusing `SQLCODE`
