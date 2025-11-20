# Documentation Repository

This is a **separate git repository** for documentation, independent from the main solvingzero repository.

## Purpose

This repository tracks documentation and daily work notes without affecting the main codebase repository.

## Usage

### Working in this repository

```bash
cd solvingzero-jcm-docs/
git status
git add .
git commit -m "Your commit message"
```

### The main repository

The main repository will see `solvingzero-jcm-docs/` as a regular directory. Since it's not in `.gitignore`, it could theoretically be added, but as long as you don't run `git add solvingzero-jcm-docs/` in the parent repo, it will remain untracked there.

## Structure

- `2025_11_20/` - Daily work folder for November 20, 2025
  - Contains implementation plans, reviews, and documentation

## Notes

- This is a nested git repository
- The parent repository does not track this directory
- No changes to parent `.gitignore` are needed
- Keep documentation commits separate from code commits

