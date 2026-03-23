# AGENTS.md - System Design Interview Repository

## Overview

This repository contains Chinese translations of system design interview materials, primarily markdown documentation. It is a **documentation-only repository** with no executable code, build systems, or tests.

The repository includes both Simplified Chinese (original translations) and Traditional Chinese translations (placed in the traditional_chinese/ directory).

## Repository Structure

```
/Users/roger/Work/misc/interview/SystemDesign/
├── CHAPTER *.md              # Volume 1 chapters (Simplified Chinese)
├── Volume2/CHAPTER *.md      # Volume 2 chapters (Simplified Chinese)
├── traditional_chinese/      # Traditionally translated chapters
│   └── CHAPTER *.md          # Volume 1 chapters (Traditional Chinese)
├── images/                   # Diagrams and figures
├── README.md                 # Main index
└── AGENTS.md                 # This file
```

## Build/Lint/Test Commands

This repository does not contain executable code. There are no build, lint, or test commands.

- **No npm/node**: Not a JavaScript/TypeScript project
- **No Python**: Not a Python project
- **No Go/Rust/Java**: No compiled language projects
- **No tests**: No test files or test frameworks

If code files are added in the future, typical commands would be:

```bash
# JavaScript/TypeScript
npm install
npm run build
npm run lint
npm run test
npm run test -- --testNamePattern="specific test"

# Python
pip install -r requirements.txt
python -m pytest
python -m pytest -k "test_name"

# Go
go build ./...
go vet ./...
go test ./...
go test -run TestName ./...
```

## Code Style Guidelines

Since this is a documentation repository, the following guidelines apply to markdown files and any future code contributions.

### Markdown Formatting

1. **File Naming**: Use meaningful, descriptive names with kebab-case for any new files
2. **Headers**: Use ATX-style headers (# ## ###) with a space after the hash
3. **Line Length**: Keep lines under 120 characters when practical
4. **Code Blocks**: Use fenced code blocks with language identifiers
5. **Links**: Use relative links for internal references, absolute URLs for external resources
6. **Images**: Place images in the `images/` directory organized by chapter

### Markdown Style Rules

```markdown
# Good header
## Good subheader

Paragraph text with proper spacing.

- Use bullet points for lists
- Keep items concise
- Use consistent indentation

1. Numbered lists for sequences
2. Use consistently

| Table | Header |
|-------|--------|
| Data  | Cells  |

[Link text](relative/path.md)
![Alt text](../images/chapter1/figure1-1.png)
```

### Naming Conventions

- **Files**: Use descriptive names (e.g., `design-youtube.md` vs `dy.md`)
- **Directories**: Use kebab-case (e.g., `system-design/`, not `systemDesign/`)
- **Image files**: Use descriptive names with chapter/figure prefixes (e.g., `figure1-1.png`)

### Error Handling

Not applicable to documentation. If code is added:

- Use try/catch for error-prone operations
- Provide meaningful error messages
- Handle edge cases explicitly

### Type Safety

Not applicable to markdown. If code is added:

- Use TypeScript for new JavaScript/TypeScript code
- Enable strict mode
- Define interfaces for data structures

### Imports

Not applicable to markdown. Standard module import patterns for the language used.

## General Guidelines for Agents

1. **Do not modify existing documentation content** - This is translated content from copyrighted material
2. **Do not add new system design content** - The content is intentionally limited to translations
3. **You may**: Fix typos, improve formatting, add missing images, organize files
4. **Images are read-only** - Do not modify or generate images
5. **Respect the directory structure** - Keep files organized by chapter/volume

## Future Development

If code examples or exercises are added to this repository:

1. Create a separate directory for code (e.g., `code/`, `examples/`, `exercises/`)
2. Include appropriate package.json, requirements.txt, or build configuration
3. Add test files with proper test coverage
4. Follow the language-specific style guide for the code
5. Document build and test commands in this file

## Notes for Agentic Coding

- This repository is for **documentation only**
- Any code changes should be minimal and well-justified
- Prioritize readability and maintainability of documentation
- When in doubt, ask the user before making structural changes
