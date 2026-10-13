# AI Software Factory

This template runs three Gemini-backed GitHub Actions stages:

- **Triage** comments an initial assessment on newly opened issues and applies only suggested labels that already exist in the repository.
- **Implementation** runs when `ready-to-implement` is added to an issue, creates a proposed code change, and opens a draft pull request.
- **Review** comments on pull request diffs when a pull request is opened, updated, or reopened.

All generated changes are proposals for human review. The implementation stage refuses patches that modify GitHub automation, Git internals, environment files, binary files, symlinks, or submodules.

## Setup

1. Add a **`GEMINI_API_KEY` Actions secret** in the repository at **Settings → Secrets and variables → Actions → New repository secret**. For account-wide use, create an organization Actions secret and grant access only to the repositories that should run the factory. Never put the key in source, workflow YAML, issue text, or repository variables.
2. Enable Actions for the repository. If required to create draft PRs, enable **Allow GitHub Actions to create and approve pull requests** under **Settings → Actions → General**; this script never approves PRs. The workflow itself grants `contents`, `issues`, and `pull-requests` write permissions because implementation pushes a branch and creates a draft PR; restrict who can set `ready-to-implement` and review/merge generated PRs.
3. Create the `ready-to-implement` issue label in the repository. Triage uses only labels that already exist; it will not create labels automatically.
4. Open an issue to run triage. After reviewing the issue, add `ready-to-implement` to request an implementation draft. Open or update a pull request to run the review stage.
5. Optionally change `GEMINI_MODEL` in `.github/workflows/software-factory.yml` to `gemini-2.5-pro` if you prefer that model.

The workflow checks out the repository's default branch rather than pull request code, so dependency installation and the factory script do not execute untrusted PR contents. Dependency install scripts are disabled. The script uses the official `@google/genai` SDK and omits common credential and certificate filenames from code context; still, only run it in repositories whose code is appropriate to send to Google AI Studio. Gemini output is untrusted: inspect all generated diffs and review comments. Do not merge automatically.

Fork pull requests may have read-only `GITHUB_TOKEN` permissions, and repository secrets are not provided to workflows triggered by fork PRs. Their automated Gemini review therefore requires an explicitly trusted event/workflow configuration; do not expose the API key to untrusted fork code to work around this restriction.

This is a per-repository template, not an account-wide GitHub installation: apply it to each repository where it should run, or publish a template and create repositories from it. GitHub has repository and organization Actions secrets, but no personal-account-wide Actions secret; use an organization secret only for repositories in an organization and limit its repository access.

## Shared system prompt

The factory loads [`developer.md`](./developer.md) from the repository root on each run and passes its contents as Gemini's `systemInstruction`, adding a short stage-specific instruction. A standalone Node.js example using the same official SDK:

```js
import { readFile } from "node:fs/promises";
import { GoogleGenAI } from "@google/genai";

const systemInstruction = await readFile("developer.md", "utf8");
const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });
const response = await ai.models.generateContent({
  model: process.env.GEMINI_MODEL ?? "gemini-2.5-flash",
  contents: "Your task-specific prompt goes here.",
  config: { systemInstruction },
});
console.log(response.text);
```

Keep `developer.md` committed and review changes to it like code. It defines intended model behavior, not actual GitHub token permissions; the workflow YAML controls those permissions.

## Local development

Run `npm ci --ignore-scripts` to install the locked dependency and `npm run factory` to invoke the agent locally. The script expects `FACTORY_STAGE`, `GEMINI_API_KEY`, `GH_TOKEN`, `GITHUB_REPOSITORY`, and `GITHUB_EVENT_PATH`; do not use a production token or API key in an untrusted checkout.
