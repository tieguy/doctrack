# DocTrack - Document Version Control Tool Specification

## Overview
DocTrack is a Python CLI tool designed for legal professionals who edit documents with LLM assistance. It provides version control for markdown documents, text-based difference tracking, and Word redline generation for sharing with colleagues.

## Core Requirements

### Primary Use Case
1. Lawyer receives ORIGINAL_DOCUMENT.doc
2. Lawyer provides document to LLM via chat interface
3. LLM returns modified document or change descriptions
4. Lawyer reviews and creates LUIS_MODIFIED_DOCUMENT.md
5. Lawyer needs to track changes and generate Word redlines for colleagues

### Key Features
- Version control for markdown documents
- Text-based diffs using dwdiff
- Export to Word redlines (.docx)
- Simple CLI with subcommands
- Automatic file detection when only one .md file exists

## Architecture

### Technology Stack
- **Language**: Python 3.8+
- **Dependencies**:
  - `click` - CLI framework
  - `dwdiff` - Text comparison (external binary)
  - `pypandoc` or `pypandoc_binary` - Pandoc wrapper for document conversion
  - `python-docx` - Word document manipulation
  - `hashlib` - File change detection (built-in)
  - `datetime` - Timestamp handling (built-in)
  - `pathlib` - File system operations (built-in)

### Directory Structure
```
project_directory/
├── document.md                 # Working document
├── doctrack_versions/          # Version storage
│   ├── commits.json           # Commit metadata
│   ├── v001_original.md       # Version files
│   ├── v002_llm_modified.md
│   └── v003_final.md
└── doctrack_exports/          # Generated redlines
    └── redline_YYYYMMDD_HHMMSS.docx
```

### Data Models

#### Commit Metadata (commits.json)
```json
{
  "commits": [
    {
      "id": "v001",
      "timestamp": "2025-07-07T10:30:00Z",
      "message": "Original document",
      "filename": "document.md",
      "hash": "sha256_hash_of_content"
    }
  ],
  "current_version": "v003"
}
```

## Command Line Interface

### Base Command
```bash
doctrack <subcommand> [options]
```

### Subcommands

#### `commit`
**Purpose**: Save current version of document
**Syntax**: `doctrack commit [-m "message"]`
**Behavior**:
- Auto-detects single .md file in directory
- Prompts for commit message if `-m` not provided
- Warns if document unchanged since last commit
- Generates sequential version ID (v001, v002, etc.)
- Stores version file and updates metadata

#### `diff`
**Purpose**: Show differences between versions
**Syntax**: `doctrack diff [version1] [version2]`
**Behavior**:
- If no versions specified: show diff between last two commits
- If one version specified: show diff between that version and current
- If two versions specified: show diff between those versions
- Uses dwdiff for text comparison
- Output displayed in terminal

#### `log`
**Purpose**: Show commit history
**Syntax**: `doctrack log`
**Behavior**:
- Display all commits in reverse chronological order
- Show: version ID, timestamp, commit message
- Format: `v003 - 2025-07-07 10:30:00 - "Final review changes"`

#### `export-redline`
**Purpose**: Generate Word redline document
**Syntax**: `doctrack export-redline [output.docx]`
**Behavior**:
- Creates redline from original (v001) to latest version
- If no output filename provided: auto-generate with timestamp
- Saves to `doctrack_exports/` directory
- Uses pandoc + python-docx for Word generation

## Implementation Details

### File Detection Logic
1. Check if specific .md file passed as argument
2. If no argument, scan current directory for .md files
3. If exactly one .md file found: use it
4. If zero files: error "No markdown files found"
5. If multiple files: error "Multiple markdown files found, please specify"

### Version Storage
- Sequential numbering: v001, v002, v003, etc.
- Zero-padded to 3 digits for proper sorting
- Each version stored as separate file in `doctrack_versions/`
- Metadata stored in `commits.json`

### Change Detection
- Calculate SHA256 hash of file content
- Compare with hash of last commit
- Ignore whitespace-only changes
- Warn user if attempting to commit unchanged document

### Diff Implementation
- Use external `dwdiff` binary for text comparison
- Default options: word-level diff with color output
- Fallback to difflib if dwdiff unavailable
- Handle binary execution errors gracefully

### Word Export Process
1. Use `pypandoc` to convert original markdown to .docx
2. Use `pypandoc` to convert latest markdown to .docx
3. Use python-docx to create redline document showing differences
4. Mark deletions with strikethrough formatting
5. Mark additions with underline formatting
6. Save to exports directory

## Error Handling

### File System Errors
- **No markdown files**: "Error: No markdown files found in current directory"
- **Multiple markdown files**: "Error: Multiple markdown files found. Please specify filename."
- **File not readable**: "Error: Cannot read file {filename}"
- **Permission denied**: "Error: Permission denied writing to {directory}"

### Version Control Errors
- **No commits exist**: "Error: No commits found. Use 'doctrack commit' to create first commit."
- **Invalid version**: "Error: Version {version} does not exist"
- **Unchanged document**: "Warning: Document unchanged since last commit. Use -f to force."

### External Tool Errors
- **dwdiff not found**: "Warning: dwdiff not found. Using basic diff."
- **pypandoc import error**: "Error: pypandoc required for Word export. Install with: pip install pypandoc"
- **pandoc not found**: "Error: pandoc binary not found. Install pandoc or use: pip install pypandoc-binary"
- **Conversion failed**: "Error: Failed to convert document to Word format"

### Command Line Errors
- **Invalid subcommand**: "Error: Unknown command '{command}'. Use 'doctrack --help' for usage."
- **Missing required argument**: "Error: {argument} is required for {command}"

## Testing Plan

### Unit Tests
1. **File Detection**
   - Test single .md file detection
   - Test multiple file error handling
   - Test no file error handling

2. **Version Management**
   - Test sequential version numbering
   - Test metadata storage and retrieval
   - Test hash-based change detection

3. **Diff Functionality**
   - Test dwdiff integration
   - Test fallback diff mechanism using difflib
   - Test version comparison logic

4. **Word Export**
   - Test pypandoc integration
   - Test redline generation with python-docx
   - Test output file naming

### Integration Tests
1. **End-to-End Workflow**
   - Initialize new document
   - Commit multiple versions
   - Generate diffs between versions
   - Export Word redline

2. **Error Scenarios**
   - Missing external dependencies (dwdiff) and Python dependencies (pypandoc, python-docx)
   - File permission issues
   - Corrupted metadata

### Manual Testing Scenarios
1. **Typical Legal Document Workflow**
   - Import original Word document (converted to .md)
   - Commit LLM-modified version
   - Commit final reviewed version
   - Generate redline for colleagues

2. **Edge Cases**
   - Very large documents (>1MB)
   - Documents with special characters
   - Documents with complex formatting

## Installation and Setup

### Prerequisites
- Python 3.8+
- dwdiff (install via package manager: `brew install dwdiff` on macOS, `apt-get install dwdiff` on Ubuntu)

### Installation
```bash
pip install doctrack
```

This will automatically install pypandoc and python-docx. For users who prefer pandoc bundled:
```bash
pip install doctrack[binary]  # includes pypandoc-binary
```

### Development Setup
```bash
git clone https://github.com/user/doctrack
cd doctrack
pip install -e .
pip install -r requirements-dev.txt
```

## Future Enhancements (Out of Scope)
- Git integration for advanced version control
- Support for other document formats (PDF, HTML)
- Web interface for non-technical users
- Integration with specific LLM platforms
- Collaborative editing features
- Advanced diff algorithms
- Template system for legal documents

## Success Criteria
1. Tool successfully tracks document versions
2. Generates accurate diffs between versions
3. Produces Word redlines compatible with legal review processes
4. Handles common error scenarios gracefully
5. Provides intuitive CLI interface for legal professionals
6. Maintains data integrity across all operations

## Acceptance Tests
1. **Basic Workflow**: User can commit original document, LLM version, and final version
2. **Diff Generation**: User can view differences between any two versions
3. **Word Export**: Generated redlines are readable and accurate in Microsoft Word
4. **Error Handling**: Tool provides helpful error messages for common mistakes
5. **Performance**: Tool handles typical legal document sizes (10-100 pages) efficiently