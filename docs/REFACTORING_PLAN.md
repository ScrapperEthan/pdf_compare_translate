# Refactoring plan for pdf_compare_translate

The goal is to turn a single long script into a set of focused modules that are easier to test and extend. The steps below assume a command-line workflow that loads PDFs, extracts text, runs translation, and emits a comparison report.

## Target architecture
- `cli/` – argument parsing and high-level orchestration (thin wrapper around services).
- `pdf_io/` – PDF loading, text extraction, and normalization utilities.
- `translate/` – translation client abstraction (wrapping providers such as DeepL or OpenAI) plus retry/backoff helpers.
- `compare/` – diffing logic (paragraph alignment, similarity scoring, highlighting differences).
- `output/` – rendering utilities for producing annotated PDFs, HTML, or plain-text reports.
- `config.py` – configuration loader (environment variables, defaults, and validation).
- `logging_setup.py` – shared logging formatters and per-module loggers.
- `tests/` – unit and integration tests for each module.

## Migration steps
1. **Stabilize interfaces:** identify the core entrypoints in the existing script (e.g., `main`, `translate_document`, `compare_documents`). Write small smoke tests to lock in their current behavior.
2. **Extract configuration:** move constants and environment reads into `config.py`. Replace direct `os.getenv`/hard-coded paths with a typed `Settings` object.
3. **Split I/O concerns:** carve out PDF reading/writing into `pdf_io` functions (e.g., `load_pdf(path)`, `extract_paragraphs(pdf)`). Update the CLI to call these helpers.
4. **Isolate translation:** wrap external API calls in the `translate` module with a clear interface (`Translator.translate(text, source, target)`). This enables mocking during tests.
5. **Modularize comparison logic:** relocate diff routines to `compare` with pure functions for scoring and alignment. Keep them free of I/O for easier testing.
6. **Refine output formatting:** gather report builders in `output` to keep presentation separate from business logic. Accept structured data (e.g., lists of diffs) instead of raw text blobs.
7. **Add tests incrementally:** as each module moves, add unit tests under `tests/` to cover the extracted functions and to prevent regressions.
8. **Clean the CLI:** once modules are in place, simplify the CLI to argument parsing plus calls to the new service functions. Avoid duplicating logic.

## Coding standards
- Prefer pure functions that accept data and return results instead of mutating globals.
- Avoid passing configuration through many layers—inject dependencies at the top of the call stack.
- Keep functions small; if a function exceeds ~40 lines or mixes concerns, consider splitting it.
- Add docstrings for public functions and describe expected inputs/outputs.
- Guard external API calls with timeouts and retries; surface clear error messages to the CLI.

## Example layout
```
pdf_compare_translate/
├─ cli/
│  └─ main.py
├─ compare/
│  ├─ align.py
│  └─ diff.py
├─ translate/
│  ├─ client.py
│  └─ providers.py
├─ pdf_io/
│  └─ reader.py
├─ output/
│  ├─ html.py
│  └─ pdf.py
├─ config.py
├─ logging_setup.py
└─ tests/
   └─ test_compare.py
```

## How to proceed
- Start with the smallest, least coupled piece (PDF reading) and move upward.
- Keep commits small and testable. After each extraction, run the smoke tests and validate CLI output on a sample document pair.
- Once the modules are in place, consider introducing type hints and static analysis (e.g., `mypy`) to guard against interface drift.
