# md-things

## About This Repository

`md-things` is a personal technical knowledge base, reference archive, and reading collection written in Markdown. It centralizes technical notes, quick-setup guides, architecture references, troubleshooting records, study materials, and summaries of material read and maintained on GitHub.

Documents are organized by subject rather than by project. Each note should remain useful as an independent reference, with enough context, commands, configuration, and verification detail for future reuse. The repository may also contain public learning notes and practical examples that help visitors understand a topic or reproduce a documented setup.

## Directory Structure

```text
.
├── cybersecurity/
│   ├── assets/
│   │   └── .gitkeep
│   └── ctf-web-command-injection.md
├── sysadmin/
│   ├── assets/
│   │   └── .gitkeep
│   └── nginx-reverse-proxy.md
├── webdev/
│   ├── assets/
│   │   └── .gitkeep
│   └── nextjs-fastsetup.md
├── hardware/
│   ├── assets/
│   │   └── .gitkeep
│   └── usb-audio-troubleshooting.md
├── general/
│   ├── assets/
│   │   └── .gitkeep
│   └── technical-reading-notes.md
├── templates/
│   ├── ctf-writeup-template.md
│   ├── general-note-template.md
│   ├── sysadmin-runbook-template.md
│   └── technical-spec-template.md
├── .gitignore
└── README.md
```

Topic files live directly inside their domain directory. For example, `webdev/nextjs-fastsetup.md` belongs directly in `webdev/`, while images and other local attachments belong in `webdev/assets/`.

## Repository Scope

### `cybersecurity/`

Security research and practical learning notes, including CTF write-ups, exploitation notes, cryptography, reverse engineering references, vulnerability analysis, and defensive observations. Document authorization and lab boundaries for offensive techniques.

### `sysadmin/`

Systems and infrastructure references, including Linux administration, network topology, server configurations, Docker, service operations, monitoring, backups, and automation guides.

### `webdev/`

Web development references, including framework setup guides such as Next.js and Laravel, ORM schemas, database tuning, API design, deployment notes, and full-stack patterns.

### `hardware/`

Hardware references, including device maintenance logs, hardware troubleshooting, component notes, firmware observations, and audio or peripheral configurations.

### `general/`

Reading and general learning material, including book summaries, language study notes, writing references, and non-technical documentation that does not fit another domain.

### `templates/`

Reusable Markdown starting points for CTF write-ups, system administration runbooks, technical specifications, and general notes. Templates are drafting aids; completed documents belong in the relevant domain directory.

## Author Guidelines and Formatting Standards

- Use lowercase `kebab-case.md` filenames, such as `nextjs-fastsetup.md`.
- Keep each topic as a standalone Markdown file directly inside its domain directory.
- Store local images and other document assets in that domain's `assets/` subdirectory.
- Use descriptive headings and short sections that support scanning and later maintenance.
- Declare syntax highlighting on every fenced code block, such as `bash`, `powershell`, `yaml`, `json`, `sql`, or `typescript`.
- Explain prerequisites, assumptions, expected output, and verification steps for operational instructions.
- Remove secrets, private keys, credentials, tokens, personal identifiers, and sensitive environment data before committing.
- Record dates and update notes when a document describes a version-sensitive tool or procedure.
- Start new documents from the closest template in `templates/`, then remove sections that do not apply.
- Keep generated files, editor metadata, and unrelated project artifacts out of the repository.

## Navigation Tip

On GitHub, press `t` to open the file finder and jump directly to a file. Use `/` to search repository contents, then filter results for `.md` files or search for a technology, command, or concept.
