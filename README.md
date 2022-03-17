
# Prepping The Data


## Text Replacements

Prepare text for inclusion into HTML template:

- `\`` => "'"
- `&` => "&amp;"
- `"` => "&quot;"
- `<` => "&lt;"
- `>` => "&gt;"
- `🅲🆁 🅻🅵` => "<p>"
  These are just a consequence of injecting our values into HTML.

- `'` => "''"
  Oracle's escape for the string-closing `'` is not the standard `\'`, but a *double apostrophe* (`''`).


Once the text has been injected, clear excessive vertical whitespace via:
- `<span><p>`  => `<span>`
- `<p></span>` => `</span>`


## Data Cleanup

All we have to identify students is "fname", "lname", "dob", and "student_number". Because these were all manually entered, they are *all* subject to input error (some are even placeholder values); and because the health plan data relies on student DCIDs, we *have* to resolve the student numbers to correct values before we can built the Oracle INSERTs.

Note that "simple typo" below includes digit/letter substitution, digit/letter transposition, additional digits/letters, missing digits/letters, and extra/missing punctuation (specifically, instances of `'`).

Best way to resolve this was:
- Add a "vetted" field so we can track which records have been vetted as good (or identified and corrected).
- Dump a set of known-good student identifiers from PowerSchool (DCID, student number, first name, last name, DOB).
- Match students across by all 4 fields, mark as vetted.
- Match students across by DOB, first name, last name; correct student numbers where the issue appears to be a simple typo, and flag as vetted.
- Match students across by DOB, last name, student number; correct first names where the issue appears to be a simple typo, and flag as vetted.
- Match students across by DOB, first name, student number; correct last names where the issue appears to be a simple typo, and flag as vetted.
- Match students across by first name, last name, student number; correct DOBs where the issue appears to be a simple typo, and flag as vetted. Note identifying DOB errors requires you also consider extra digits and disallowed dates would have been truncated or blocked by the input field, so you have to be *very* forgiving of errors.

Anything left at this point can't be matched cleanly, and since DOB and student number are the most likely to be incorrect, we have to rely on names. To be as inclusive as possible, I matched students across by first 4 letters of the first name and first 4 letters of the last name, alone. This will generate a very large list of *possible* matches. Disregard all obvious mismatches, and correct names wherever there is an obvious simple typo. Now repeat the whole process with better names to work off.


## Generating The Oracle INSERTs

There are numbered queries in both the Access file and within the repo (as the "access-sql" files). Queries "00" and "01" must be run in that order; the "02" queries can be run in any order; the "03" query comes last.

Running these all populates the "sql_inserts" table, whose `sql` data can then be run in PowerSchool's Oracle DB to actually insert the student health records (but only after manually running the template insert query in the repo).


## Other Gotchas

- Oracle has a hard 4000 character limit on string (VARCHAR2) literals, so we can't insert the full document template in one go. The solution is to break the document into many chunks (each smaller than 4K) that can all be cast into CLOB values and concatenated together inline. Note this also means all string replacement logic has to wrap things in CLOB casts and that no single string value being injected can itself be longer than 4000 chars.
