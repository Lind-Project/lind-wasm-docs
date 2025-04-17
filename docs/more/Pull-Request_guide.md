# GitHub Pull Request Guide

## Overview
This document outlines the project guidelines for submitting pull requests (PRs).

## Setup
- **All commits must be signed:** [GitHub GPG Key Guide](https://docs.github.com/en/authentication/managing-commit-signature-verification/telling-git-about-your-signing-key)

## Code Formatting
- Code must adhere to our style guidelines [Style Guide](styleguide.md)
- Code must pass **Clippy**: [Clippy](https://doc.rust-lang.org/clippy/)
- Code must be formatted using **cargo-fmt**: [Cargo Fmt](https://doc.rust-lang.org/cargo/commands/cargo-fmt.html)

## Naming Conventions
### Branch Naming
- **Good:** `select-bugfix`
- **Bad:** `yourname1`

### PR Title
- **Good:** `Adding mmap functionality`
- **Bad:** `My PR`

## PR Description
- Provide a **detailed description** of the aim of your PR.
- Large PRs may require multiple paragraphs, but bullet points are acceptable if they clearly convey necessary details.
- **Key areas to review** should be specifically highlighted.

## Tagging & Review Process
- Tag **two peers** for an initial review.
- Once peer reviews are addressed, the PR will be sent to maintainers for a final review.
- If tagged for review, please respond **within 48 hours**.
- If unable to review in time, notify the PR author so they can tag someone else.

