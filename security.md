# Security Policy

## What These Skills Do

These skills are instruction files that teach Claude how to query public federal APIs. They do not execute code on your machine, store credentials on disk, or transmit data to third parties. API calls go directly from Claude to the respective government API endpoints.

## API Keys

Some skills require free API keys (BLS, api.data.gov). These keys are stored in Claude's memory, not in the skill files. Never commit API keys to this repo.

## Reporting a Vulnerability

If you find a security issue with any skill (for example, a skill that could cause Claude to leak API keys or make unintended requests), please open an issue or contact the maintainer directly.

## Supported Versions

Only the latest version of each skill is supported. Update to the current version if you encounter issues.
