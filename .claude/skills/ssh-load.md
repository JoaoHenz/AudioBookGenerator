Ensure an SSH key is loaded in the agent before any operation that needs it (e.g. git push).

## Process

1. Run `ssh-add -l` to check if any keys are loaded
2. If keys are already loaded, done — no action needed
3. If no keys are loaded, list available keys from `~/.ssh/` (look for files without `.pub` extension that have a matching `.pub`)
4. If there's only one key, load it automatically with `ssh-add`
5. If there are multiple keys, show the list and ask the user which one to add
6. If the only available key fails to load, let the user know
