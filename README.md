# MCPA Prep Course

A complete, self-paced preparation course for the
**Linux Foundation Model Context Protocol Associate (MCPA)** certification,
built as a MkDocs Material documentation site and published to GitHub Pages.

The course is structured against the
[official exam domains](https://training.linuxfoundation.org/certification/model-context-protocol-associate-mcpa/)
and grounded in the **MCP specification revision `2026-07-28`**, the version
the exam references.

| Domain | Weight |
| --- | --- |
| 1. MCP Fundamentals | 16% |
| 2. Architecture & Components | 14% |
| 3. Interactions & Execution | 26% |
| 4. Security & Governance | 24% |
| 5. Use Cases & Ecosystem | 20% |

It contains study chapters, per-domain quizzes, two full weighted mock exams and
seven hands-on labs.

## Run it locally

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

Then open <http://127.0.0.1:8000>.

## Publish

Push to `main`. The GitHub Actions workflow in `.github/workflows/docs.yml`
builds the site and deploys it to the `gh-pages` branch. Enable
**Settings → Pages → Deploy from a branch → `gh-pages` / (root)** once.

## Disclaimer

Unofficial study material. Not affiliated with or endorsed by the Linux
Foundation, the Agentic AI Foundation or the MCP project. Practice questions are
original and are **not** real exam questions.
