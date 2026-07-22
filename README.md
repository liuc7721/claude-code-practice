# claude-code-practice

Practice project for using Claude Code with the full GitHub workflow:
commit, push, GitHub Actions CI, and pull requests.

## Scripts

```bash
npm install
npm run build   # compile TypeScript
npm test        # run tests with vitest
npm run dev     # run src/index.ts directly
```

## Branch protection test

This line was committed directly to `main` to test whether the branch
protection rule actually blocks direct pushes on a free-plan private repo.
