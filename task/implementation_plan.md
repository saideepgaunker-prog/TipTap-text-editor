# High-Performance Tiptap Rich Text Editor Implementation Plan

## 1. Executive Summary & Problem Analysis

Based on `d:/SJ/tiptap/task/crrent stuture`, the existing writing experience is built around **Quill with the Snow toolbar** (`RichTextEditor.jsx` wrapped inside `ManuscriptDraftEditor.jsx`). 

### Limitations of Current Quill Setup:
1. **Feature Gaps**: No Find & Replace, no comments/inline feedback, no track changes/suggestion mode, no tables, no callouts/admonitions, no slash commands (`/`), and no contextual floating bubble menu.
2. **Rigid Formatting & Styling**: Fixed Snow theme styling with limited customization, lacks modern typography controls, dark/light themes, and distraction-free Zen mode.
3. **HTML / Delta Disconnect**: Quill forces internal Delta conversions while the database, Word export, and autosave expect normalized HTML strings with specific attributes (`class="ql-align-*"`, `line-height`, `margin-bottom`, `font-size: 12pt`, etc.).
4. **Performance on Long Documents**: Quill can suffer from stuttering caret and sluggish undo/redo on large manuscripts.

### The Objective:
Rebuild the editor using **Tiptap v2 (ProseMirror foundation)** to deliver:
- **Flawless Backward Compatibility**: Matches 100% of legacy behavior (Tab first-line indent, Enter inheritance, Word paste cleanup, Times New Roman 12pt double-spacing defaults, HTML string I/O).
- **Major Performance & Speed Boost**: Native ProseMirror transaction engine ensures buttery-smooth 60fps typing, instant undo/redo, and virtualized resilience on 50,000+ word manuscripts.
- **Superior Features & Look**: Modern Notion/Medium-grade aesthetics, dark/light modes, floating selection bubble menu, `/` slash command palette, rich tables, document outline sidebar, inline comments, find & replace, live word goals, version snapshots, and DOCX/PDF/HTML/Markdown exports.

---

## 2. Feature & Architecture Comparison

| Feature / Behavior | Current Quill Editor | Proposed Modern Tiptap Rebuild |
| :--- | :--- | :--- |
| **Engine Foundation** | Quill DOM Tree + Delta in-memory | ProseMirror Schema + Tiptap v2 Headless Extension Pipeline |
| **Default Manuscript Style** | Times New Roman, 12pt, 2.0 line spacing | Configurable presets (Manuscript Standard, Modern Novel, Academic, Screenplay) |
| **Headings & Hierarchy** | H1, H2, H3, Normal | H1, H2, H3, H4, Subheading, Body, Monospace Code block |
| **Font Family & Size** | Times New Roman, Arial (pt/px) | Times New Roman, Arial, Georgia, Inter, Merriweather, JetBrains Mono (10pt–32pt) |
| **Line & Para Spacing** | 1, 1.15, 1.5, 2 (Space after: 0, 6, 12, 18pt) | Custom LineHeight & MarginBottom extension with exact pt values matching backend docx engine |
| **Indentation (Tab / Shift+Tab)** | 5-space `&nbsp;` or text-indent | Custom `IndentExtension` supporting both 5-space `&nbsp;` & CSS `text-indent: 0.5in`, with outdent on Shift+Tab |
| **Smart Enter** | Carries paragraph indent (except headings/lists) | Smart keymap extension preserving line spacing, indent, and style context |
| **Word / Docs Paste** | Basic regex cleanup | AST-based DOM sanitization cleaning MS Word MsoNormal junk while preserving true indents & blank lines |
| **Selection Toolbar** | None (only fixed top toolbar) | **Floating Bubble Menu** with instant bold, italic, color, comment, and link actions |
| **Block Insertion** | Click top toolbar only | **Slash Commands (`/`)** for fast block insertion |
| **Find & Replace** | ❌ Not available | ✅ Full Find & Replace modal with regex/case match & batch replace |
| **Tables** | ❌ Not available | ✅ Interactive Grid Table with cell merge, row/column operations & header rows |
| **Comments & Review** | ❌ Not available | ✅ Inline comment threads, highlights, and sidebar review panel |
| **Document Outline** | ❌ Not available | ✅ Live Table of Contents heading tree with 1-click jump |
| **Metrics & Goals** | Basic word count | Live words, characters, reading time, sentences, + customizable session word target bar |
| **Autosave & Recovery** | 800ms debounce to DB + LocalStorage | 800ms debounce with visual status ("Saving...", "Saved", "Offline draft restored") + snapshot history |
| **Export Formats** | Backend-only Word download | Client-side **DOCX (.docx)**, clean **HTML**, **Markdown (.md)**, **Print/PDF** preview |

---

## 3. Detailed Component & Extension Plan

### A. Custom Tiptap Extensions (`src/extensions/`)
1. **`FontFamilyExtension`**: Allows setting font family inline or at paragraph level (`Times New Roman`, `Arial`, `Georgia`, `Inter`, `Merriweather`, etc.).
2. **`FontSizeExtension`**: Applies exact `pt` values (`10pt`, `11pt`, `12pt`, `14pt`, `16pt`, `18pt`, `24pt`) via style attributes, fully compatible with backend docx parser.
3. **`LineHeightExtension`**: Renders `style="line-height: X;"` on paragraph and heading nodes.
4. **`ParagraphSpacingExtension`**: Sets `style="margin-bottom: Xpt;"` (None, 6pt, 12pt, 18pt, 24pt).
5. **`ManuscriptIndentExtension`**: 
   - Intercepts `Tab` when cursor is not in a list or table to insert 5 non-breaking spaces `\u00A0\u00A0\u00A0\u00A0\u00A0` or apply first-line indent.
   - Intercepts `Shift+Tab` to strip leading indent.
   - Intercepts `Enter` to inherit paragraph indent without breaking headings or alignments.
6. **`CalloutExtension`**: Custom node for author notes, chapter summaries, character sheets, and warning boxes.
7. **`CommentExtension`**: Custom inline mark for highlighted reviewer comments with unique thread IDs.
8. **`SlashCommandExtension`**: ProseMirror suggestion plugin triggering a floating command menu upon typing `/`.

### B. UI & Toolbar Architecture (`src/components/`)
1. **`Toolbar.jsx`**:
   - Undo / Redo buttons with shortcut hints.
   - Presets dropdown ("Standard Manuscript (Double 12pt TNR)", "Drafting Mode", "Clean Sans").
   - Heading Selector (H1, H2, H3, Body).
   - Font Family & Size dropdowns.
   - Formatting buttons (Bold, Italic, Underline, Strikethrough, Subscript, Superscript, Code).
   - Text Color & Highlight Color popovers with curated color palette and reset option.
   - Alignment buttons (Left, Center, Right, Justify).
   - Line Spacing & Paragraph Bottom Spacing selectors.
   - List controls (Bullet list, Numbered list, Task checklist).
   - Insert buttons (Link, Table, Callout box, Divider line, Blockquote).
   - View toggles (Outline drawer, Comments sidebar, Find & Replace modal, Zen full-screen mode, Dark/Light theme).
2. **`BubbleMenu.jsx`**: Sleek floating toolbar appearing over highlighted text for rapid formatting and commenting.
3. **`SlashCommandMenu.jsx`**: Keyboard-navigable popup menu triggered by `/`.
4. **`FindReplaceModal.jsx`**: Floating drawer to search occurrences, highlight matches in the document, and replace single or all instances.
5. **`CommentsSidebar.jsx`**: Sidebar displaying all active comment threads, timestamps, author names, and resolve/delete actions.
6. **`DocumentOutline.jsx`**: Real-time table of contents showing H1/H2/H3 hierarchy with click-to-scroll.
7. **`StatsBar.jsx`**: Live footer with word count, character count, estimated reading time, reading level, and daily word goal progress bar.
8. **`VersionHistoryModal.jsx`**: Browse automatic local snapshots, preview past versions, and restore with one click.

### C. Data Flow & Normalization Utilities (`src/utils/`)
1. **`htmlNormalizer.js`**:
   - Ensures output matches backend specifications: `<p class="ql-align-center" style="line-height: 2; margin-bottom: 12pt;"><span style="font-size: 12pt;">Chapter One</span></p>`.
   - Forces empty paragraphs to `<p><br></p>` on load so blank lines are never lost.
   - Cleans up and normalizes non-standard tags.
2. **`wordPasteParser.js`**:
   - Filters clipboard paste data to eliminate Microsoft Word `MsoNormal` junk and redundant spans while preserving paragraph boundaries, indents, italics, and bolds.
3. **`docxExporter.js`**:
   - Generates true formatted `.docx` files on the fly using `docx` library with correct margins, font family, font size, and line spacing.

---

## 4. Proposed File Structure

```
d:/SJ/tiptap/
├── package.json
├── vite.config.js
├── index.html
├── task/
│   ├── crrent stuture
│   └── implementation_plan.md
├── src/
│   ├── main.jsx
│   ├── App.jsx
│   ├── index.css
│   ├── components/
│   │   ├── Editor/
│   │   │   ├── TipTapEditor.jsx
│   │   │   ├── Toolbar.jsx
│   │   │   ├── BubbleMenu.jsx
│   │   │   ├── SlashCommandMenu.jsx
│   │   │   ├── TableControls.jsx
│   │   │   ├── FindReplaceModal.jsx
│   │   │   ├── CommentsSidebar.jsx
│   │   │   ├── DocumentOutline.jsx
│   │   │   ├── StatsBar.jsx
│   │   │   └── VersionHistoryModal.jsx
│   │   └── UI/
│   │       ├── Dropdown.jsx
│   │       ├── Tooltip.jsx
│   │       ├── Modal.jsx
│   │       └── ColorPicker.jsx
│   ├── extensions/
│   │   ├── FontSize.js
│   │   ├── FontFamily.js
│   │   ├── LineSpacing.js
│   │   ├── ParagraphSpacing.js
│   │   ├── ManuscriptIndent.js
│   │   ├── SlashCommand.js
│   │   ├── Callout.js
│   │   └── CommentMark.js
│   ├── utils/
│   │   ├── htmlNormalizer.js
│   │   ├── wordPasteParser.js
│   │   ├── docxExporter.js
│   │   └── storage.js
│   └── constants/
│       ├── typography.js
│       └── sampleManuscript.js
```

---

## 5. Verification & Testing Plan

### A. Automated Compilation & Package Integrity
- Ensure `package.json` installs cleanly and Vite builds with zero warnings or errors.
- Confirm all Tiptap v2 extensions and ProseMirror dependencies resolve smoothly.

### B. Manuscript Format Verification
- Verify Times New Roman 12pt double spaced output by default.
- Test `Tab` key in normal text: verify 5 spaces / indent is inserted.
- Test `Shift+Tab`: verify indent is stripped.
- Test `Enter`: verify line spacing & indent preservation.
- Test empty paragraphs: verify `<p><br></p>` is preserved without collapsing.

### C. Advanced Features Verification
- **Slash Commands**: Type `/table`, `/h1`, `/callout`, `/quote` to verify rapid insertion.
- **Bubble Menu**: Select text and verify floating menu appears with inline actions.
- **Tables**: Insert 3x3 table, add rows/columns, merge cells, test table styling.
- **Comments**: Highlight text, add comment, verify highlight badge and sidebar list.
- **Find & Replace**: Search words, cycle through matches, replace single/all.
- **Autosave & Snapshots**: Type content, observe 800ms autosave indicator, reload page and verify recovery.
- **DOCX / HTML / MD Export**: Download `.docx` and `.html` and verify fidelity.
