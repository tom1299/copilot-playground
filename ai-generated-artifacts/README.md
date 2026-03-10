# Prompts and related generated artifacts

## Analyze the script for possible security related issues.
Related to the `copilot-install.sh`. Copilot analysis can be found at [security-analysis.md](./security-analysis.md).

## "Rename master to main and delete master"
* Model: Claude Sonnet 4.6
* Generated content:
    ```bash
    git branch -m master main
    git push origin main
    git push origin --delete master
    git remote set-head origin main
    ```
* Problem: No mention of remote / did not exist. Neither in workspace nor mentioned in the prompt.