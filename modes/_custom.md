# Custom Instructions -- career-ops

<!-- ============================================================
     THIS FILE IS YOURS. It will NEVER be auto-updated.

     Put your own house rules, custom workflows, and automations
     here -- anything you want the agent to ALWAYS do (or never do).

     This is for PROCEDURAL rules ("HOW I want things done").
     For WHO you are (archetypes, narrative, comp, negotiation),
     use modes/_profile.md instead. Keeping the two separate keeps
     each one readable.

     The agent reads this file alongside the system instructions;
     your rules here take precedence over the defaults, as long as
     they don't break the Data Contract (your files are never
     touched, and we never auto-submit an application for you).

     Because this is a user-layer file, anything you write here
     survives `node update-system.mjs`. Put customizations HERE,
     not in CLAUDE.md / modes/_shared.md / other system files --
     those get overwritten on update.
     ============================================================ -->

## House Rules

<!-- Rules the agent should always follow. Examples:
     - Always write evaluation summaries in British English.
     - Never include a photo in my CV (US / ATS-first market).
     - Cap each batch run at 20 listings unless I say otherwise.
     - If a report scores below 6, skip the cover letter. -->

- **CV / resume: indented sub-bullets in `cv.md` belong to the bullet above them.**
  Treat a top-level bullet plus its indented sub-bullets as one unit. When
  reordering for a JD, move the whole unit; never separate a sub-bullet from its
  parent, move it under a different parent, or place it before its parent. Units
  may be reordered freely; sub-bullets may be reordered within their own unit.
  Applies to every role in `cv.md` and every output format (`pdf`, `latex`,
  `latex-tex`, `text`, batch).

- **CV / resume section order: Work Experience is the first section; Education is
  the last section.** Applies to every generated CV in every format. The templates
  do not do this on their own, so after building and before compiling:
  - **LaTeX** (`build-cv-latex.mjs` output): Education is the first section in
    `templates/cv-template.tex`. Move the block that starts at the
    `%%%%... Education ...%%%%` banner comment and runs up to the next `%%%%` banner
    so it sits immediately before the `%%%%  END  %%%%` line. Experience then
    becomes the first section automatically.
  - **HTML** (`build-cv-html.mjs` output, `pdf` mode): Experience already comes
    first. Move the block from `<!-- EDUCATION -->` up to the next `<!-- ... -->`
    section marker so it sits immediately before `<!-- END -->`.
  - Verify with `pdftotext -raw` that the first section heading is the experience
    heading and the last is Education, and that the CV still fits its page limit.

## Custom Workflows

<!-- Multi-step routines you run often, given a short name. Examples:
     - "weekly review": scan my saved portals, evaluate the new roles,
       then give me a one-paragraph summary of the top 3.
     - "prep <company>": pull the JD, generate STAR stories from
       article-digest.md, and draft 5 likely interview questions. -->

(none yet -- add yours above)

## Output Preferences

<!-- How you like results formatted. Examples:
     - Reports: lead with the score and the one-line verdict.
     - Show the per-step token breakdown after a batch run.
     - Save PDFs date-first: YYYY-MM-DD-company.pdf -->

- **CV / resume: never include a Professional Summary section.** Applies to every
  generated CV (`pdf`, `latex`, `latex-tex`, `text`, batch): omit the `summary`
  field from the render payload entirely. The summary in `cv.md` stays as source
  material only. `build-cv-html.mjs` still emits an empty "Professional Summary"
  heading when `summary` is absent (summary is not in `OPTIONAL_SECTIONS`), so
  after building the HTML and before `generate-pdf.mjs`, delete the block from
  `<!-- PROFESSIONAL SUMMARY -->` through its closing `</div>` (up to the next
  section comment). Verify with `grep 'section-title">Professional Summary'` → no match.

- **CV / resume contact line: no icons; use text labels.** Label the contact links
  as `Email: …`, `LinkedIn: …`, `GitHub: …`, separated by ` | `. Applies to every
  generated CV in every format.
  - **LaTeX** (`build-cv-latex.mjs` output): the template prefixes each link with a
    FontAwesome icon (`\raisebox{-0.2\height}\faEnvelope\ `, `\faLinkedin\ `,
    `\faGithub\ `) and joins the links with `~`. After building and before
    compiling, replace each icon prefix with its text label (`Email: `,
    `LinkedIn: `, `GitHub: `) and replace the `~` between the email/LinkedIn and
    LinkedIn/GitHub links with ` $|$ `. Keep the `\href` and `\underline` around
    the address itself.
  - **HTML** (`build-cv-html.mjs` output, `pdf` mode): the contact row has no
    icons; prefix the email, LinkedIn, and GitHub link text with the same labels.
  - Verify with `pdftotext -raw` that the line reads
    `Email: … | LinkedIn: … | GitHub: …` and that no `\fa` icon command remains in
    the `.tex`.

## Off-Limits

<!-- Things the agent must never do for you. Examples:
     - Never auto-fill or submit an application without showing me first.
     - Never edit a system file to customize my setup -- put it here. -->

(none yet -- add yours above)
