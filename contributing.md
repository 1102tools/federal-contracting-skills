# Contributing

Thanks for your interest in improving these tools.

## Reporting Issues

- **Bug**: Open an issue with the skill name, what you asked Claude, and the error or unexpected behavior.
- **Feature request**: Open an issue describing the workflow you want and why it would be useful.
- **API change**: If a federal API changes its endpoints or response format, open an issue with the details.

## Submitting Changes

1. Fork the repo
2. Create a branch for your change
3. Edit the relevant SKILL.md file(s)
4. Update the version number and add a changelog entry
5. Open a pull request with a clear description of what changed and why

## Skill Guidelines

- Keep main skill files under 500 lines so they fit in Claude's context reliably
- Move reference material (lookup tables, composite workflows) to a separate reference file
- Use the existing skill structure as a template
- Test your changes by installing the skill in Claude and running the workflows

## What We Need

- New federal API integrations
- Improvements to existing skill accuracy
- Better SOC code mappings
- Edge case handling
- Corrections to FAR/DFARS references
