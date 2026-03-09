# /triage-nda — NDA Pre-Screening

> If you see unfamiliar placeholders or need to check which tools are connected, see [CONNECTORS.md](../CONNECTORS.md).

Triage the NDA: @$1

Rapidly triage incoming NDAs against standard screening criteria. Classify the NDA for routing: standard approval, counsel review, or full legal review.

## Invocation

```
/triage-nda [company name]
/triage-nda [with file attached]
```

## Workflow

### Step 1: Determine Input Mode

Two modes are supported — detect which applies:

**Mode A — Company name only** (user typed a company name, no file attached):
- Record the company name from the argument or from the user's message.
- Proceed to Step 2 to locate the NDA on SharePoint.

**Mode B — File provided** (user attached or uploaded a DOCX/PDF directly in chat):
- Use the uploaded file as the NDA.
- Skip Step 2 and go directly to Step 3.
- Track that the file came from chat — this determines output behaviour in Step 8.

If neither a company name nor a file is present, ask: "Please type the company name or attach the NDA file."

### Step 2: Locate the NDA on SharePoint (Mode A only)

Using the `ms365` SharePoint connector, search for the company's folder under Blume's standard deal pipeline path:

**Base path**: `2. Pipeline > Deals 2026 > [Company Name]`

1. Search SharePoint for a folder matching `[Company Name]` inside `Deals 2026`.
2. Inside that company folder, look for a subfolder named `Admin` (or `Admin (NDAs etc)` or similar).
3. Retrieve any DOCX or PDF file that looks like an NDA (filename contains "NDA", "Non-Disclosure", "Confidentiality", or "CDA").
4. If exactly one NDA file is found, use it automatically — no need to ask the user.
5. If multiple NDA files are found, list them briefly and ask the user which to use.
6. If no NDA file is found, tell the user and ask them to attach the file directly.

Record the full SharePoint path of the retrieved file — this is where the output will be saved in Step 8.

7. Copy the original file from the OneDrive-synced local path to the VM working directory so Step 5 can start from a byte-perfect copy.

> **⚠️ IMPORTANT — Do NOT attempt Graph API download or token hunting.**
> The MS365 connector's access token is managed by Anthropic's cloud infrastructure. It is **never** available as an environment variable, in the Windows Credential Manager, MSAL cache, process environment, or any location accessible from the VM or local Windows shell. Searching for it is wasted effort and will always fail. Use the OneDrive sync path instead.

SharePoint document libraries are synced locally via OneDrive Business at:
`C:\Users\orlan\blumeequity.com\Blume Equity - Documents\`

The NDA file on Windows will typically be at:
`C:\Users\orlan\blumeequity.com\Blume Equity - Documents\2. Pipeline\Deals 2026\[Company Name]\Admin (NDAs etc)\[filename].docx`

Use Desktop Commander `list_directory` or `start_search` to confirm the exact subfolder name (it may be "Admin", "Admin (NDAs etc)", or similar). Then use `start_process` with PowerShell `Copy-Item` to copy the binary file to the VM outputs folder:

```powershell
Copy-Item -Path "C:\Users\orlan\blumeequity.com\Blume Equity - Documents\2. Pipeline\Deals 2026\[Company]\Admin (NDAs etc)\[filename].docx" -Destination "C:\Users\orlan\AppData\Roaming\Claude\local-agent-mode-sessions\<session-id>\<agent-id>\local_<local-id>\outputs\[filename].docx"
```

The VM outputs folder Windows path can be found via Desktop Commander `get_config` → `allowedDirectories`, or by checking the known mapping: the VM path `/sessions/<session-name>/mnt/outputs/` maps to a Windows path under `C:\Users\orlan\AppData\Roaming\Claude\local-agent-mode-sessions\`.

Record the exact OneDrive folder path used — this is where the output will be copied back in Step 8.

### Step 3: Read the NDA (token-efficient method)

**Always use python-docx via Bash — never use Desktop Commander's `read_file` on a DOCX.**

Desktop Commander's XML mode on DOCX files returns thousands of tokens of verbose namespace markup per chunk, and requires multiple reads to cover the full document. python-docx extracts clean paragraph text in a single pass for a fraction of the token cost.

```python
# Run via Bash: python3 << 'EOF'
from docx import Document

path = "/sessions/<session-name>/mnt/outputs/[filename].docx"
doc = Document(path)

print("=== ALL PARAGRAPHS ===")
for i, para in enumerate(doc.paragraphs):
    text = para.text.strip()
    if text:
        print(f"[{i}] {text}")
# EOF
```

The full paragraph list gives you everything you need for triage — typically 50–80 paragraphs, well under 3,000 tokens.

### Step 4: Screen and Classify

Apply Blume Equity's NDA playbook as defined in the **`nda-triage` skill**. The skill contains all screening criteria, classification rules (GREEN/YELLOW/RED), Blume-specific positions, what to flag, what NOT to flag, and the standard redline positions for common issues.

Load and follow the skill — do not duplicate its content here.

### Step 5: Generate Triage Report

Prepare a concise triage report using the format defined in the `nda-triage` skill. Output this report directly in chat. Do NOT embed it in the document — the document contains tracked changes only.

### Step 6: Create Redlined Document (YELLOW and RED only)

For **YELLOW and RED** classifications, automatically produce a redlined version of the NDA. Do this without asking — just do it. For GREEN, skip this step.

**Output file name**: `[original filename without extension] Blume comments.docx`
Example: `Cosuno NDA.docx` → `Cosuno NDA Blume comments.docx`

---

**CRITICAL: Preserve exact formatting — always start from a binary copy of the original file.**

Never create a new `Document()` from scratch. Always start by copying the original file byte-for-byte with `shutil.copy2()`, then open the copy and apply changes. This preserves all fonts, font sizes, paragraph styles, page margins, page size, headers, footers, section breaks, tables, and any other formatting exactly as-is.

```python
import shutil, os, datetime, random, copy
from docx import Document
from docx.shared import Pt, RGBColor
from docx.oxml.ns import qn
from docx.oxml import OxmlElement

original_path = "/path/to/original NDA.docx"
output_path = "/path/to/original NDA Blume comments.docx"

# Step 1: byte-perfect copy — preserves ALL formatting
shutil.copy2(original_path, output_path)

# Step 2: open the copy (never the original)
doc = Document(output_path)
```

---

**Document structure**:
The document is the original NDA body with tracked changes applied inline — nothing else. No triage report, no cover page, no separator. The triage report lives in chat only (Step 5). No reformatting of any kind — tracked changes only wrap the text content, never the paragraph or run formatting properties.

---

**How to apply tracked changes (deletions and insertions)**:

Tracked changes wrap text in `<w:del>` or `<w:ins>` XML elements. Apply them directly to the paragraph XML. **Never modify the paragraph's existing `<w:rPr>` (run properties) — only wrap the text content.**

```python
DATE = datetime.datetime.utcnow().strftime("%Y-%m-%dT%H:%M:%SZ")
AUTHOR = "Blume Equity"
_rev_counter = [1000]

def next_rev_id():
    _rev_counter[0] += 1
    return str(_rev_counter[0])

def tracked_delete_run(text_to_delete, existing_rpr_xml=None):
    """Return a <w:del> element containing the given text."""
    del_el = OxmlElement('w:del')
    del_el.set(qn('w:id'), next_rev_id())
    del_el.set(qn('w:author'), AUTHOR)
    del_el.set(qn('w:date'), DATE)
    r = OxmlElement('w:r')
    if existing_rpr_xml is not None:
        rpr_copy = copy.deepcopy(existing_rpr_xml)
        r.append(rpr_copy)
    dt = OxmlElement('w:delText')
    dt.set(qn('xml:space'), 'preserve')
    dt.text = text_to_delete
    r.append(dt)
    del_el.append(r)
    return del_el

def tracked_insert_run(new_text, existing_rpr_xml=None):
    """Return a <w:ins> element containing the given text (blue underline)."""
    ins_el = OxmlElement('w:ins')
    ins_el.set(qn('w:id'), next_rev_id())
    ins_el.set(qn('w:author'), AUTHOR)
    ins_el.set(qn('w:date'), DATE)
    r = OxmlElement('w:r')
    rpr = OxmlElement('w:rPr')
    if existing_rpr_xml is not None:
        for child in copy.deepcopy(existing_rpr_xml):
            rpr.append(child)
    color = OxmlElement('w:color')
    color.set(qn('w:val'), '0000FF')
    u = OxmlElement('w:u')
    u.set(qn('w:val'), 'single')
    rpr.append(color)
    rpr.append(u)
    r.append(rpr)
    t = OxmlElement('w:t')
    t.set(qn('xml:space'), 'preserve')
    t.text = new_text
    r.append(t)
    ins_el.append(r)
    return ins_el

def replace_text_in_para_with_tracked_change(para, old_text, new_text):
    """Find old_text in a paragraph and replace it with a tracked deletion + insertion."""
    full_text = para.text
    if old_text not in full_text:
        return False
    runs = para.runs
    for run in runs:
        if old_text in run.text:
            rpr_xml = run._r.find(qn('w:rPr'))
            before, _, after = run.text.partition(old_text)
            run_el = run._r
            for t in run_el.findall(qn('w:t')):
                run_el.remove(t)
            if before:
                t_el = OxmlElement('w:t')
                t_el.set(qn('xml:space'), 'preserve')
                t_el.text = before
                run_el.append(t_el)
            else:
                run_el.getparent().remove(run_el)
                run_el = None
            parent = para._p
            ref_el = run_el if run_el is not None else para._p
            idx = list(parent).index(ref_el) + 1 if run_el else len(list(parent))
            del_el = tracked_delete_run(old_text, rpr_xml)
            ins_el = tracked_insert_run(new_text, rpr_xml)
            parent.insert(idx, del_el)
            parent.insert(idx + 1, ins_el)
            if after:
                after_r = OxmlElement('w:r')
                if rpr_xml is not None:
                    after_r.append(copy.deepcopy(rpr_xml))
                after_t = OxmlElement('w:t')
                after_t.set(qn('xml:space'), 'preserve')
                after_t.text = after
                after_r.append(after_t)
                parent.insert(idx + 2, after_r)
            return True
    return False

def add_tracked_insertion_after_para(para, inserted_text):
    """Insert a new paragraph with tracked-insertion text immediately after the given paragraph."""
    new_p = OxmlElement('w:p')
    orig_ppr = para._p.find(qn('w:pPr'))
    if orig_ppr is not None:
        new_p.append(copy.deepcopy(orig_ppr))
    ins_el = tracked_insert_run(inserted_text)
    new_p.append(ins_el)
    para._p.addnext(new_p)

def mark_para_for_deletion(para):
    """Wrap every run in a paragraph in <w:del> tracked-change elements."""
    for run in para.runs:
        rpr_xml = run._r.find(qn('w:rPr'))
        text = run.text
        if not text:
            continue
        run_el = run._r
        for t in run_el.findall(qn('w:t')):
            run_el.remove(t)
        del_el = tracked_delete_run(text, rpr_xml)
        run_el.getparent().insert(list(run_el.getparent()).index(run_el), del_el)
        run_el.getparent().remove(run_el)
```

---

**Application strategy — locate and redline each issue**:

Search for key phrases from each flagged clause among the document's paragraphs. Match using substring search across `para.text`. For each issue, use the redline positions from the `nda-triage` skill to determine the replacement language:

- **Text to replace** (e.g., overbroad language, wrong term duration): use `replace_text_in_para_with_tracked_change(para, old_text, new_text)`
- **Missing provision to insert** (e.g., missing carveout, missing IAC retention clause): use `add_tracked_insertion_after_para(nearest_para, suggested_text)` to insert the new language as a tracked insertion immediately after the most relevant existing paragraph
- **Entire clause to delete** (e.g., non-compete, exclusivity): use `mark_para_for_deletion(para)` on every paragraph of that clause, and follow with `add_tracked_insertion_after_para` adding: `"[BLUME EQUITY: This provision is not acceptable in an NDA and should be deleted in full.]"`

**Common redline patterns**:

| Issue | Action |
|---|---|
| Overbroad confidential information definition | Replace overbroad language with narrowed scope |
| Term too long (e.g., "five (5) years") | Replace duration with "two (2) years" |
| Missing carveout | Insert tracked-insertion paragraph after carveout section with suggested wording |
| Non-solicitation too broad | Replace general hiring restriction with targeted solicitation language |
| No IAC/committee paper retention right | Insert tracked-insertion after return/destruction clause |
| Portfolio companies in affiliate definition | Insert parenthetical exclusion: "(for the avoidance of doubt, portfolio companies and investments of Blume Equity LLP shall not be treated as Affiliates for the purposes of this Agreement)" |
| Non-compete / exclusivity / standstill / IP | Mark clause paragraphs as tracked deletions + insert explanatory note |

**Fallback — if clause location is unreliable**: Rather than failing silently, append a clearly marked "BLUME EQUITY COMMENTS" section at the very end of the document (still as tracked insertions) listing each issue and the suggested replacement language in full. This ensures no issue is dropped even if the exact clause text couldn't be matched.

---

**Run-level inspection** (only when needed for redlining — not for triage):

If you need to inspect specific runs to get precise text boundaries for redlining, add this targeted block after the paragraph dump, only for the paragraphs you'll edit:

```python
# Run via Bash — only when you need run-level detail for redlining
for i, para in enumerate(doc.paragraphs):
    if i in [15, 19, 32, 37, 54]:  # only the paragraphs you'll touch
        print(f"\n--- PARA [{i}] RUNS ---")
        for j, run in enumerate(para.runs):
            if run.text:
                print(f"  run[{j}]: {repr(run.text)}")
```

---

**Final save**: Call `doc.save(output_path)` — no prompts, no confirmation.

### Step 7: Deliver Output

The delivery method depends on how the NDA was provided:

**If the file was uploaded directly in chat (Mode B)**:
- Save the redlined DOCX to the working directory
- Return it directly in the chat as a downloadable file
- Do NOT attempt to upload it anywhere

**If the file was retrieved from SharePoint (Mode A)**:
- Copy the redlined DOCX from the VM outputs folder back to the same OneDrive-synced folder recorded in Step 2. OneDrive will sync it to SharePoint automatically — **no Graph API upload is needed or possible**.

> **⚠️ IMPORTANT — Do NOT attempt Graph API upload or token hunting.**
> The MS365 access token is inaccessible from the VM or Windows environment. The correct approach is simply to write the output file into the OneDrive Business sync folder on the Windows filesystem — OneDrive handles the rest.

Use Desktop Commander `start_process` with PowerShell `Copy-Item`:

```powershell
Copy-Item -Path "C:\Users\orlan\AppData\Roaming\Claude\local-agent-mode-sessions\<session-id>\<agent-id>\local_<local-id>\outputs\[filename] Blume comments.docx" -Destination "C:\Users\orlan\blumeequity.com\Blume Equity - Documents\2. Pipeline\Deals 2026\[Company]\Admin (NDAs etc)\[filename] Blume comments.docx"
```

- After the copy succeeds, confirm in chat: "Saved `[filename] Blume comments.docx` to `[SharePoint folder path]` — OneDrive will sync automatically."
- Do NOT send the file in chat — it will appear in SharePoint once OneDrive syncs (typically within seconds)

### Step 8: Routing Suggestion

Based on the classification:

- **GREEN**: Route to Michelle, Eleanor, or Clare for signature. No further review needed.
- **YELLOW**: Share the redlined document with Michelle, Eleanor, or Clare — the specific issues are flagged in the document for a single review pass.
- **RED**: Do not sign. Share the redlined document with Michelle, Eleanor, or Clare for full review. Offer Blume's standard 2-way or 1-way NDA template as a counterproposal.

## Notes

- If the document is not actually an NDA (e.g., it's labeled as an NDA but contains substantive commercial terms), flag this immediately as a RED and recommend full contract review instead
- For NDAs that are part of a larger agreement (e.g., confidentiality section in an MSA), note that the broader agreement context may affect the analysis
- This is a pre-screening tool — Michelle, Eleanor, or Clare should review anything flagged before Blume commits
- The redlined document uses tracked changes that can be accepted/rejected in Microsoft Word
