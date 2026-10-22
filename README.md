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
## Credits and provenance

- **Clown and the original research contributors:** credit for the original SRC research workflows and collected knowledge belongs to their authors. The package is publicly circulated under the researcher handle **Clown**, as reflected in [community discussion](https://t.me/s/linux_do_channel/496534).
- **baianquanzu:** maintains a [public archive of the Chinese `clown-src-6k-skill` package](https://github.com/baianquanzu/6kskill/tree/main/clown-src-6k-skill). This archive is a provenance reference.
- **Honghanb0 and contributors:** the [jiaoxunzhixing project](https://github.com/Honghanb0/jiaoxunzhixing) documents the `vuln-engine` rule-library extension included in this collection.
- **James Kettle and PortSwigger Research:** the knowledge base references [Practical Web Cache Poisoning](https://portswigger.net/research/practical-web-cache-poisoning); credit for that research remains with its original author and publisher.
- **Other cited researchers and tool authors:** existing source links and references inside the knowledge base are retained. Their research and tools remain their work.

This repository contributes an English translation and an overview README. It does not claim authorship of the original research, endorsement by the researchers, or any additional license over the source material.
