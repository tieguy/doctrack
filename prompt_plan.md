## Project Blueprint

### High-Level Architecture
1. **Core Infrastructure**: Project setup, CLI framework, and basic file operations
2. **Version Control Engine**: Commit system, metadata management, and version storage
3. **Diff System**: Integration with dwdiff and fallback mechanisms
4. **Export System**: Word document generation with redlines
5. **Integration**: Wire all components together with proper error handling

### Detailed Breakdown

#### Phase 1: Foundation (Steps 1-3)
- Project structure and dependencies
- Basic CLI with click
- File detection logic

#### Phase 2: Version Control (Steps 4-7)
- Version directory management
- Commit metadata structure
- Basic commit functionality
- Hash-based change detection

#### Phase 3: Diff Functionality (Steps 8-10)
- Diff command structure
- dwdiff integration
- Fallback diff implementation

#### Phase 4: History and Logging (Steps 11-12)
- Log command implementation
- Version retrieval utilities

#### Phase 5: Export System (Steps 13-15)
- Export command structure
- Pandoc integration
- Word redline generation

#### Phase 6: Integration and Polish (Steps 16-18)
- Error handling improvements
- Integration testing
- Documentation and packaging

## Implementation Prompts

### Step 1: Project Setup and Structure

Create a Python project structure for DocTrack, a CLI tool for document version control. Set up:

1. Create the following directory structure:
   - doctrack/
     - __init__.py
     - cli.py (main CLI entry point)
     - version_control.py (version management logic)
     - diff.py (diff functionality)
     - export.py (export functionality)
     - utils.py (shared utilities)
   - tests/
     - __init__.py
     - test_cli.py
     - test_version_control.py
     - test_diff.py
     - test_export.py
     - test_utils.py
   - setup.py
   - requirements.txt
   - requirements-dev.txt

2. Create setup.py with:
   - Package name: doctrack
   - Entry point: doctrack.cli:main
   - Dependencies: click>=8.0, python-docx>=0.8.11
   - Optional dependencies: pypandoc-binary>=1.11 (for [binary] extra)
   - Python requirement: >=3.8

3. Create requirements.txt with:
   - click>=8.0
   - python-docx>=0.8.11
   - pypandoc>=1.11

4. Create requirements-dev.txt with:
   - pytest>=7.0
   - pytest-cov>=4.0
   - black>=22.0
   - flake8>=5.0

5. In doctrack/__init__.py, set __version__ = "0.1.0"

Include proper docstrings and type hints. Make sure all test files have placeholder test functions that pass.

### Step 2: Basic CLI Framework

Implement the basic CLI framework using click in doctrack/cli.py:

1. Create a main click group called 'doctrack' with proper help text
2. Add placeholder commands for: commit, diff, log, export-redline
3. Each command should print a message like "Command X not yet implemented"
4. Add proper help text for each command based on the specification
5. Ensure the CLI can be run with `python -m doctrack` and shows help

Write tests in test_cli.py that:
- Test that all commands exist
- Test that help text is displayed correctly
- Test that each command can be invoked (even if just showing placeholder text)
- Use click.testing.CliRunner for testing

The CLI should follow this structure:
- doctrack commit [-m "message"]
- doctrack diff [version1] [version2]
- doctrack log
- doctrack export-redline [output.docx]

Make sure to handle the main entry point properly so it works both as a module and when installed.

### Step 3: File Detection Utilities

Implement file detection logic in doctrack/utils.py:

1. Create a function find_markdown_file(path: Path = None) -> Path that:
   - Takes an optional path parameter
   - If path is provided and is a .md file, return it
   - If path is a directory or None, scan current directory for .md files
   - If exactly one .md file found, return it
   - If zero files, raise FileNotFoundError with message "No markdown files found in current directory"
   - If multiple files, raise ValueError with message "Multiple markdown files found. Please specify filename."

2. Create a function ensure_directory(directory: Path) -> None that:
   - Creates the directory if it doesn't exist
   - Handles permission errors gracefully

3. Create constants for:
   - VERSIONS_DIR = "doctrack_versions"
   - EXPORTS_DIR = "doctrack_exports"
   - COMMITS_FILE = "commits.json"

Write comprehensive tests in test_utils.py that:
- Test single file detection
- Test multiple file error
- Test no file error
- Test explicit file path
- Test directory creation
- Use pytest fixtures and tmp_path for isolated testing

Update cli.py to use find_markdown_file in the commit command (just print the found file for now).

### Step 4: Version Control Data Models

Create the version control data models and basic operations in doctrack/version_control.py:

1. Create a Commit dataclass with:
   - id: str (e.g., "v001")
   - timestamp: str (ISO format)
   - message: str
   - filename: str
   - hash: str

2. Create a CommitHistory class that:
   - Loads/saves from commits.json
   - Has methods: add_commit(), get_commits(), get_latest_commit(), get_commit_by_id()
   - Handles the case when no commits.json exists yet
   - Uses proper JSON serialization/deserialization

3. Create a function generate_version_id(commits: List[Commit]) -> str that:
   - Returns "v001" for first commit
   - Returns next sequential version (v002, v003, etc.)
   - Properly zero-pads to 3 digits

4. Create a function calculate_file_hash(filepath: Path) -> str that:
   - Reads file content
   - Returns SHA256 hash of content
   - Handles encoding properly (utf-8)

Write tests in test_version_control.py that:
- Test Commit serialization/deserialization
- Test CommitHistory operations
- Test version ID generation
- Test hash calculation
- Test edge cases (empty history, file not found, etc.)

Make sure all functions have proper type hints and docstrings.

### Step 5: Basic Commit Functionality

Implement the commit functionality by updating doctrack/cli.py and creating commit logic:

1. In version_control.py, add a function commit_file(filepath: Path, message: str = None) -> Commit that:
   - Ensures doctrack_versions directory exists
   - Calculates hash of current file
   - Checks if file has changed since last commit (if any commits exist)
   - Generates new version ID
   - Copies file to versions directory with version filename
   - Creates and saves commit metadata
   - Returns the created Commit object

2. Update the commit command in cli.py to:
   - Accept optional -m/--message parameter
   - Find the markdown file using find_markdown_file()
   - Prompt for message if not provided (use click.prompt)
   - Call commit_file()
   - Show success message with version ID
   - Handle the "unchanged file" case with appropriate warning

3. Add error handling for:
   - File not found
   - Permission errors
   - Unchanged file (show warning but allow with --force flag)

Write integration tests that:
- Test successful commit
- Test commit with message
- Test commit without message (mock the prompt)
- Test unchanged file warning
- Test force commit of unchanged file
- Use temporary directories and files

Make sure the versions directory structure matches the spec (v001_original.md, etc.).

### Step 6: Change Detection Enhancement

Enhance the commit functionality with better change detection and metadata handling:

1. In version_control.py, add a method to CommitHistory:
   - has_changes(filepath: Path) -> bool that compares current file hash with latest commit
   - get_version_filepath(version_id: str) -> Path that returns the path to a version file

2. Update commit_file to:
   - Use CommitHistory.has_changes() for detection
   - Include the original filename in the version filename (e.g., v001_document.md)
   - Add better error messages for various failure cases

3. Create a function in utils.py:
   - format_timestamp() -> str that returns current time in ISO format
   - parse_version_filename(filename: str) -> tuple[str, str] that extracts version ID and original name

4. Add validation to ensure:
   - Commit messages don't exceed reasonable length (e.g., 200 chars)
   - Version IDs are generated correctly even if files are manually deleted
   - The system recovers gracefully from corrupted commits.json

Write additional tests that:
- Test the edge case of corrupted or manually edited commits.json
- Test version file naming with different input filenames
- Test hash comparison for whitespace-only changes
- Test recovery from missing version files

Update the CLI to show more informative messages during commit operations.

### Step 7: Version Retrieval Utilities

Create utilities for retrieving and working with versions in version_control.py:

1. Add methods to CommitHistory class:
   - get_version_content(version_id: str) -> str that reads a version file
   - get_version_pairs() -> List[Tuple[Commit, Commit]] for getting consecutive version pairs
   - resolve_version_spec(spec: str = None) -> Commit that handles "latest", version IDs, or None

2. Create a VersionResolver class that:
   - Handles version specifications like "v001", "latest", or None
   - Can resolve two version specs for diff operations
   - Provides clear error messages for invalid versions

3. Add utility functions:
   - list_versions() -> List[str] that returns all version IDs in order
   - get_versions_for_diff(v1: str = None, v2: str = None) -> Tuple[Commit, Commit]
   - The function should handle the spec's default behaviors:
     * No args: last two versions
     * One arg: that version vs current
     * Two args: those two versions

Write comprehensive tests for:
- Version resolution with various inputs
- Edge cases (only one version exists, no versions)
- Version content retrieval
- Version pair generation
- Current file vs version comparisons

These utilities will be the foundation for the diff and log commands.

### Step 8: Log Command Implementation

Implement the log command to show commit history:

1. Update the log command in cli.py to:
   - Load CommitHistory
   - Display commits in reverse chronological order
   - Format: "v003 - 2025-07-07 10:30:00 - Final review changes"
   - Handle case when no commits exist with helpful message
   - Use click.echo() for output

2. Add formatting utilities in utils.py:
   - format_commit_line(commit: Commit) -> str
   - parse_iso_timestamp(timestamp: str) -> str that creates human-readable format
   - truncate_message(message: str, max_length: int = 50) -> str

3. Add options to log command:
   - --limit/-n to show only last N commits
   - --oneline for compact format
   - --verbose/-v for full commit details (hash, filename)

4. Enhance output with click.style() for better readability:
   - Version IDs in cyan
   - Timestamps in yellow
   - Messages in default color

Write tests that:
- Test log output format
- Test empty history
- Test various display options
- Test output formatting
- Mock click.echo to capture output

Make sure the log command provides useful information at a glance.

### Step 9: Basic Diff Infrastructure

```text
Create the basic diff infrastructure in doctrack/diff.py:

1. Create a DiffEngine abstract base class with:
   - diff(content1: str, content2: str) -> str method
   - is_available() -> bool class method

2. Create a DwdiffEngine(DiffEngine) that:
   - Uses subprocess to call external dwdiff command
   - Implements is_available() by checking if dwdiff exists in PATH
   - Handles subprocess errors gracefully
   - Returns colored diff output

3. Create a SimpleDiffEngine(DiffEngine) that:
   - Uses Python's difflib as fallback
   - Produces unified diff format
   - Always returns True for is_available()

4. Create a get_diff_engine() -> DiffEngine function that:
   - Returns DwdiffEngine if available
   - Falls back to SimpleDiffEngine
   - Logs which engine is being used

5. Create a diff_versions(v1: Commit, v2: Commit) -> str that:
   - Loads content from both versions
   - Uses the diff engine to generate diff
   - Handles file reading errors

Write tests that:
- Test both diff engines (mock dwdiff for CI)
- Test engine selection
- Test diff output format
- Test error handling
- Use pytest fixtures for test content

The infrastructure should be extensible for future diff engines.

### Step 10: Diff Command Implementation

Implement the diff command using the diff infrastructure:

1. Update the diff command in cli.py to:
   - Accept optional version1 and version2 arguments
   - Use get_versions_for_diff() to resolve versions
   - Call diff_versions() to generate diff
   - Display diff with proper formatting
   - Handle all the default behaviors from the spec

2. Add diff options:
   - --no-color to disable colored output
   - --output/-o to save diff to file instead of displaying
   - --engine to force specific diff engine

3. Enhance diff output:
   - Add header showing which versions are being compared
   - Show file names and timestamps
   - Use click.echo_via_pager() for long diffs

4. Add special handling for current file diffs:
   - When comparing a version to current file
   - Show "Current file" instead of version ID
   - Handle case when current file doesn't exist

5. Error handling:
   - Clear messages when versions don't exist
   - Helpful error when no versions to compare
   - Warning when files are identical

Write integration tests that:
- Test all three diff scenarios (no args, one arg, two args)
- Test diff output
- Test file output option
- Test error cases
- Mock subprocess calls for dwdiff

The diff command should be the most flexible and useful command.

### Step 11: Export Infrastructure

Create the export infrastructure in doctrack/export.py:

1. Create a check_dependencies() function that:
   - Checks if pypandoc is installed
   - Tries to import pypandoc and checks for pandoc binary
   - Returns tuple (available: bool, error_message: str)

2. Create a PandocConverter class that:
   - Wraps pypandoc functionality
   - Has convert_to_docx(md_content: str, output_path: Path) method
   - Handles pypandoc exceptions gracefully
   - Falls back to basic conversion if needed

3. Create a RedlineGenerator class that:
   - Takes two Word documents (original and modified)
   - Generates a redline document using python-docx
   - Marks deletions with strikethrough
   - Marks additions with underline
   - Handles paragraph-level differences

4. Create utility functions:
   - generate_export_filename() -> str that creates timestamp-based names
   - ensure_exports_directory() that creates exports dir
   - markdown_to_docx(md_path: Path, docx_path: Path) wrapper

Write tests that:
- Test dependency checking (mock imports)
- Test pandoc conversion (mock pypandoc)
- Test redline generation with python-docx
- Test filename generation
- Create actual test Word documents for redline testing

The export system should gracefully degrade if dependencies are missing.

### Step 12: Export Command Implementation

Implement the export-redline command:

1. Update the export-redline command in cli.py to:
   - Accept optional output filename
   - Check dependencies before proceeding
   - Load first and latest versions
   - Generate redline document
   - Save to exports directory
   - Show success message with output path

2. Add the main export logic:
   - Convert original (v001) markdown to Word
   - Convert latest version markdown to Word
   - Generate redline between them
   - Clean up temporary files

3. Add export options:
   - --from/-f to specify starting version (default: v001)
   - --to/-t to specify ending version (default: latest)
   - --format to support future formats (for now just .docx)

4. Error handling:
   - Clear message if pypandoc not installed with installation instructions
   - Handle case when no versions exist
   - Handle Word document generation failures
   - Cleanup on failure

5. Progress indication:
   - Show "Converting original document..."
   - Show "Converting latest version..."
   - Show "Generating redline..."

Write integration tests that:
- Test successful export
- Test missing dependencies
- Test custom output filename
- Test version selection
- Mock the actual Word generation but test the flow

The export should produce professional redlines suitable for legal review.

### Step 13: Advanced Diff Features

Enhance the diff functionality with advanced features:

1. In diff.py, add configuration for dwdiff:
   - Create DwdiffOptions class with word-level, line-level, char-level modes
   - Add options for context lines
   - Add options for ignoring whitespace/case

2. Implement a unified diff format option:
   - Add UnifiedDiffEngine that produces patch-compatible output
   - Include file headers with timestamps
   - Support standard unified diff format

3. Add diff statistics:
   - Create DiffStats class that counts additions/deletions
   - Show summary after diff (X additions, Y deletions)
   - Calculate percentage changed

4. Implement side-by-side diff view:
   - Create SideBySideDiffEngine for terminal display
   - Handle terminal width detection
   - Provide option to switch between unified and side-by-side

5. Add caching for diff results:
   - Cache computed diffs in memory during session
   - Improve performance for repeated operations

Write tests for:
- Different diff formats
- Statistics calculation
- Option handling
- Cache behavior
- Terminal width handling

Update the diff command to expose these new options through CLI flags.

### Step 14: Error Handling and Recovery

Implement comprehensive error handling and recovery mechanisms:

1. Create a custom exception hierarchy in utils.py:
   - DocTrackError (base exception)
   - FileOperationError
   - VersionControlError
   - DependencyError
   - InvalidInputError

2. Add recovery mechanisms in version_control.py:
   - Backup commits.json before modifications
   - Atomic write operations (write to temp, then rename)
   - Verify file integrity after operations
   - Rollback on failure

3. Create a diagnostic command:
   - doctrack diagnose
   - Checks installation health
   - Verifies version directory integrity
   - Reports missing dependencies
   - Suggests fixes for common issues

4. Add transaction support:
   - Create Transaction context manager
   - Ensures all-or-nothing operations
   - Cleans up on failure
   - Logs operations for debugging

5. Improve error messages:
   - Add suggestions for fixing common errors
   - Include relevant context (filenames, versions)
   - Differentiate between user errors and system errors

Write tests that:
- Test each exception type
- Test recovery mechanisms
- Test atomic operations
- Simulate various failure scenarios
- Test diagnostic command output

Update all commands to use the new error handling system.

### Step 15: Configuration and Customization

Add configuration support for DocTrack:

1. Create a Config class in utils.py that:
   - Loads from .doctrackrc or doctrack.ini
   - Supports project-level and user-level configs
   - Has defaults for all settings
   - Validates configuration values

2. Add configurable options:
   - Default diff engine
   - Export formats
   - Version naming scheme
   - Ignore patterns for files
   - Custom version/export directories

3. Create a config command:
   - doctrack config --list (show all settings)
   - doctrack config get <key>
   - doctrack config set <key> <value>
   - doctrack config --init (create default config)

4. Implement environment variable support:
   - DOCTRACK_DIFF_ENGINE
   - DOCTRACK_VERSIONS_DIR
   - DOCTRACK_EXPORTS_DIR
   - Override config file settings

5. Add .doctrackignore support:
   - Ignore certain files when scanning
   - Support glob patterns
   - Similar to .gitignore

Write tests for:
- Config loading and precedence
- Invalid configuration handling
- Environment variable override
- Ignore patterns
- Config command operations

Update all commands to respect configuration settings.

### Step 16: Performance Optimization

Optimize DocTrack for performance with large documents:

1. Implement lazy loading in version_control.py:
   - Don't load all commit content upfront
   - Stream large files instead of loading into memory
   - Use generators where appropriate

2. Add progress indicators:
   - For long operations (export, large diffs)
   - Use click.progressbar()
   - Show estimated time remaining

3. Implement parallel processing:
   - For batch operations (future feature)
   - Use concurrent.futures for I/O operations
   - Maintain thread safety

4. Add performance benchmarking:
   - Time critical operations
   - Log slow operations in debug mode
   - Add --verbose flag to show timings

5. Optimize diff algorithms:
   - Cache computed hashes
   - Use incremental hashing for large files
   - Implement early exit for identical files

Write performance tests that:
- Test with large documents (1MB+)
- Measure operation times
- Test memory usage
- Verify optimization effectiveness
- Use pytest-benchmark

Update commands to use optimized implementations.

### Step 17: Integration Testing and Polish

Create comprehensive integration tests and polish the user experience:

1. Create end-to-end test scenarios in tests/integration/:
   - Full lawyer workflow test
   - Multi-version document test
   - Error recovery test
   - Performance test with large docs

2. Add interactive features:
   - Confirmation prompts for destructive operations
   - Interactive version selection for diff
   - Colored output throughout
   - Helpful command suggestions

3. Implement command aliases:
   - 'c' for commit
   - 'd' for diff
   - 'l' for log
   - 'e' for export-redline

4. Add shell completion:
   - Generate completion scripts for bash/zsh
   - Support filename completion
   - Support version ID completion

5. Create helpful status command:
   - Show current document
   - Show last commit info
   - Show uncommitted changes
   - Suggest next actions

Write tests that:
- Cover entire user workflows
- Test interactive features (with mocking)
- Verify shell completion
- Test command aliases
- Measure test coverage (aim for >90%)

Polish all user-facing messages and ensure consistent formatting.

### Step 18: Documentation and Packaging

Finalize the project with documentation and packaging:

1. Create comprehensive documentation:
   - README.md with quick start guide
   - CONTRIBUTING.md for developers
   - docs/ directory with detailed guides
   - Man page for doctrack command

2. Add inline help improvements:
   - Examples in --help output
   - Common use cases
   - Troubleshooting section

3. Create release automation:
   - setup.py with all metadata
   - MANIFEST.in for including docs
   - GitHub Actions for CI/CD
   - Automated testing on multiple Python versions

4. Add packaging configurations:
   - pyproject.toml for modern packaging
   - Support for pip install
   - Optional dependencies handling
   - Platform-specific considerations

5. Create demo materials:
   - Sample documents
   - Tutorial walkthrough
   - Video script for demo
   - Comparison with alternatives

Write final tests for:
- Package installation
- Entry point functionality
- Documentation examples
- Import tests
- Cross-platform compatibility

Ensure the project is ready for distribution on PyPI.