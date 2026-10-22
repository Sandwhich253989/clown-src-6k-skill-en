# clown-src-6k-skill-en

This repository is an **English translation of the original `clown-src-6k-skill` collection**. It is a translation copy, not an independently authored security framework. The original material, organization, and technical guidance belong to their original authors.

The collection brings together agent instructions, security-research rules, a vulnerability-testing knowledge base, and vulnerability-engine configuration. Its workflows cover black-box bug-bounty research and white-box code auditing.

## Contents

| Path | Overview |
| --- | --- |
| `AGENTS.md` | Global agent instructions and references to the research rules. |
| `config.toml` | Original application configuration, copied unchanged. |
| `docs/user-guide/` | User documentation for the terminal coding assistant; these files were already in English. |
| `rules/` | Research workflows, scope management, reporting conventions, and agent behavior rules. |
| `skills/skill/` | The skill entrypoint and its vulnerability-testing knowledge base. |
| `vuln-engine/` | Vulnerability rule definitions, scanner and integration configurations, and a test report. |
| `给朋友的提示词.txt` | Translated setup directions and the accompanying prompt. |

## Translation approach

Chinese explanatory prose has been translated into English. The original directory names and filenames are preserved, including Chinese names, so existing references continue to resolve. Markdown structure, code blocks, command syntax, URLs, and technical identifiers are retained. Chinese text may remain where it is a literal path, filename, placeholder, or test value.

Configuration files are copied unchanged, as permitted by the translation scope. Existing English documentation is retained. Apparent irregularities in the original documents are preserved rather than silently corrected.

The translated instructions describe the original author's intended workflows; translating them does not verify every technical claim or establish compatibility with a particular tool version. Consult the original `clown-src-6k-skill` material when resolving ambiguities in meaning.