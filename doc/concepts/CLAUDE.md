# How to produce these documents

Provide this sort of prompt to claude:

We need to deeply understand how $CONCEPT and referenced implementations work
within this library. Please write a `doc/concepts/$CONCEPT.md` research
document. Keep the final document under 350 lines -- use terse, precise, and
concise language.

- Start by reading `$REPO_ROOT/CLAUDE.md`,
  `$REPO_ROOT/lib/Image/ExifTool/README`, and any related docs in
  `$REPO_ROOT/doc/concepts/*.md`.

- As you grep the source for what to study, be sure to reference relevant files
  in `$REPO_ROOT/doc/modules/*.md` for a quick summary of these files --
  ExifTool source files are large, and creating these files are separate
  (nontrivial) jobs -- leverage that prior work!

- We want to keep documentation DRY and minimize reading for our new engineers.
  For all content that you would have included in in your doc, if
  `$REPO_ROOT/lib/Image/ExifTool/README` already covers the same content,
  replace your content with a reference to that README.

- Especially include odd, surprising or "tribal knowledge" discoveries that
  could assist new engineers in tasks relating to what you are documenting.
