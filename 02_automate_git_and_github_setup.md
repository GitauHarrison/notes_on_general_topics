# Stop Re-Typing Your Git & GitHub Config: Automate It on macOS and Linux

If you’ve ever set up a new Mac or Linux machine, you probably know the drill:

*   Install Git
    
*   Set your name and email
    
*   Generate SSH keys
    
*   Add them to GitHub
    
*   Tweak `~/.ssh/config`
    
*   Maybe juggle multiple GitHub accounts…
    

It’s boring, repetitive, and easy to mess up.

In this article, I’ll show you how to automate Git and GitHub configuration on macOS and Linux using a few small shell scripts. After this, new machines become:

```python
git clone .gitcd ./setup.sh   
```

…and you’re done.

What we’re automating
---------------------

We’ll automate these steps:

*   Configure global Git identity (name, email, default branch).
    
*   Generate and register SSH keys for GitHub.
    
*   Configure ~/.ssh/config so Git uses the right key automatically.
    
*   Optionally support multiple GitHub accounts on the same machine.
    
*   Wrap everything in a single ./setup.sh so you don’t have to remember commands.
    

All sensitive data is provided via prompts at runtime — no hardcoded emails or keys in your repo.

Want the executable files? Check out the repo [git_github_configurations](https://github.com/GitauHarrison/git_github_configurations).

The building blocks
-------------------

We’ll use three scripts:

1.  `setup_github_single.sh` — for a single GitHub account on the machine.
    
2.  `setup_github_multi.sh` — for multiple GitHub accounts (e.g. personal + work).
    
3.  `setup.sh`— a small wrapper that lets you choose between 1 or 2.
    

You can put these in a dedicated repo, for example: _git\_github\_configurations_.

A quick primer: `./`, shebangs, and `chmod +x`
------------------------------------------

Before the scripts, three small Unix basics:

### ./setup.sh

*   `./` means “in the current directory”.
    
*   `./setup.sh` = “run the file setup.sh located right here”.
    
*   If you just type setup.sh, the shell only looks in `$PATH` (e.g. `/usr/bin`, `/usr/local/bin`, not in the current directory, for security reasons.
    

### Shebang: #!/usr/bin/env bash

The first line in our scripts:

```python 
#!/usr/bin/env bash   
```

tells the OS:

> “Use the tool env to find bash in the user’s PATH, and run this script with that bash.”

This is more portable than hardcoding `#!/bin/bash` because bash can live in different locations on different systems.

### chmod +x

By default, a file is just data. To run it as a program, you must mark it as **executable**:

```python
chmod +x setup.sh
```

This changes the file’s permissions so ./setup.sh is allowed to run.

Script 1: Single-account Git & GitHub setup
-------------------------------------------

This script configures one GitHub account on the current machine.

High-level steps:

*   Ask for your Git name, email, and default branch.
    
*   Set global Git configs:
    ◦ `user.name`
    ◦ `user.email`
    ◦ `init.defaultBranch`
    
*   Configure some sensible Git defaults.
    
*   Generate an **SSH key** (if needed) at `~/.ssh/id\_ed25519`
    
*   Start ssh-agent, add the key, and update `~/.ssh/`config for _github.com_.
    
*   On macOS, enable **UseKeychain** so your key passphrase can be stored in the system keychain.
    
*   Optionally run `gh auth login` if you have the GitHub CLI installed.
    
*   Print your public key so you can add it to GitHub.
    

Example script (trimmed for readability):

```python
#!/usr/bin/env bash
set -euo pipefail

echo "=== Git & GitHub single-account setup ==="

# Detect OS (macOS vs Linux)
UNAME_OUT="$(uname -s)"
case "${UNAME_OUT}" in
  Darwin*) OS="macos" ;;
  Linux*)  OS="linux" ;;
  *)       OS="other" ;;
esac

# --- Git identity ---
read -rp "Git user name (e.g. Your Name): " GIT_NAME
read -rp "Git email (e.g. you@example.com): " GIT_EMAIL

read -rp "Default git branch name [main]: " GIT_BRANCH
GIT_BRANCH=${GIT_BRANCH:-main}

git config --global user.name "$GIT_NAME"
git config --global user.email "$GIT_EMAIL"
git config --global init.defaultBranch "$GIT_BRANCH"

git config --global pull.rebase false
git config --global push.default simple
git config --global color.ui auto

# --- SSH key for GitHub ---
mkdir -p "$HOME/.ssh"
DEFAULT_KEY="$HOME/.ssh/id_ed25519"

echo "SSH key setup for GitHub"
read -rp "Generate a new SSH key at $DEFAULT_KEY if it doesn't exist? [Y/n]: " GEN_KEY
GEN_KEY=${GEN_KEY:-Y}

if [[ "$GEN_KEY" =~ ^[Yy]$ ]]; then
  if [[ -f "$DEFAULT_KEY" ]]; then
    read -rp "Key exists. Overwrite? [y/N]: " OVERWRITE
    OVERWRITE=${OVERWRITE:-N}
    if [[ "$OVERWRITE" =~ ^[Yy]$ ]]; then
      ssh-keygen -t ed25519 -C "$GIT_EMAIL" -f "$DEFAULT_KEY"
    fi
  else
    ssh-keygen -t ed25519 -C "$GIT_EMAIL" -f "$DEFAULT_KEY"
  fi
fi

# ... start ssh-agent, add key, configure ~/.ssh/config, optionally run gh auth login ...

echo "Single-account Git & GitHub setup complete."
```

See the full version from [this repo](https://github.com/GitauHarrison/git_github_configurations/blob/main/setup_github_single.sh); here it’s shortened to focus on the core ideas.

Script 2: Multi-account Git & GitHub setup
------------------------------------------

Now the fun part: **multiple GitHub accounts** on the same machine.

Goals:

*   Separate SSH keys per account: _id\_ed25519\_personal_, _id\_ed25519\_work_, etc.
    
*   Separate SSH host aliases: github-personal, github-work.
    
*   Separate Git identities via `~/.gitconfig-personal` and `~/.gitconfig-work`.
    
*   Automatic identity selection based on directory using `includeIf “gitdir:…/”`.
    

The flow:

1.  Ask how many accounts you want (e.g. 2: personal + work).
    
2.  For each account, ask for:
    
    *   Label: personal, work, etc.
    
    *   Git name and email for that account.
    
    *   Base directory where repos will live (e.g. _~/work_).
    
    *   Create an SSH key like _~/.ssh/id\_ed25519\_personal_ (if needed).
        ◦ Add a `Host github-<label>` block to `~/.ssh/config`.
    
    *   Create a per-account `~/.gitconfig-<label>`file.
    
    *   Append an includeIf block to your main `~/.gitconfig`.
    

Example (again, trimmed):

```python
#!/usr/bin/env bash
set -euo pipefail

echo "=== Git & GitHub multi-account setup ==="

mkdir -p "$HOME/.ssh"
SSH_CONFIG="$HOME/.ssh/config"
touch "$SSH_CONFIG"

# Start ssh-agent if needed
if ! pgrep -u "$USER" ssh-agent >/dev/null 2>&1; then
  eval "$(ssh-agent -s)" >/dev/null
fi

read -rp "How many GitHub accounts do you want to configure? " ACCOUNT_COUNT

for ((i=1; i<=ACCOUNT_COUNT; i++)); do
  echo "=== Account #$i ==="
  read -rp "Short account label (e.g. personal, work): " LABEL
  LABEL="${LABEL// /-}"

  read -rp "Git user name for '$LABEL': " GIT_NAME
  read -rp "Git email for '$LABEL': " GIT_EMAIL

  read -rp "Base directory for '$LABEL' repos (e.g. \$HOME/work): " BASE_DIR
  BASE_DIR="${BASE_DIR/#\~/$HOME}"

  DEFAULT_KEY="$HOME/.ssh/id_ed25519_${LABEL}"
  read -rp "SSH key path for '$LABEL' [$DEFAULT_KEY]: " KEY_PATH
  KEY_PATH=${KEY_PATH:-$DEFAULT_KEY}

  if [[ ! -f "$KEY_PATH" ]]; then
    ssh-keygen -t ed25519 -C "$GIT_EMAIL" -f "$KEY_PATH"
  fi
  ssh-add "$KEY_PATH" || true

  HOST_ALIAS="github-${LABEL}"
  if ! grep -qE "^Host ${HOST_ALIAS}(\s|\$)" "$SSH_CONFIG"; then
    {
      echo ""
      echo "Host ${HOST_ALIAS}"
      echo "  HostName github.com"
      echo "  User git"
      echo "  IdentityFile ${KEY_PATH}"
      echo "  IdentitiesOnly yes"
      echo "  AddKeysToAgent yes"
    } >> "$SSH_CONFIG"
  fi

  ACCOUNT_GITCONFIG="$HOME/.gitconfig-${LABEL}"
  cat > "$ACCOUNT_GITCONFIG" <<EOF
[user]
  name = ${GIT_NAME}
  email = ${GIT_EMAIL}
EOF

  GITCONFIG_MAIN="$HOME/.gitconfig"
  touch "$GITCONFIG_MAIN"
  if ! grep -q "gitdir:${BASE_DIR}/" "$GITCONFIG_MAIN"; then
    {
      echo ""
      echo "[includeIf \"gitdir:${BASE_DIR}/\"]"
      echo "  path = ${ACCOUNT_GITCONFIG}"
    } >> "$GITCONFIG_MAIN"
  fi

  echo "For '$LABEL': clone with git@${HOST_ALIAS}:<owner>/<repo>.git"
done
```

After running this:

*   Put your work repos under `~/work/`, personal under `~/personal/`, etc.
    
*   Clone using the host alias:
    

    ```python 
    git clone git@github-work:your-org/your-repo.gitgit clone git@github-personal:your-user/your-repo.git   
    ```

Git will automatically use the correct `user.name` and `user.email` based on the directory.

Script 3: setup.sh — the entrypoint
-----------------------------------

To avoid remembering which script to run, add a simple wrapper:

```python
git clone git@github-work:your-org/your-repo.git
git clone git@github-personal:your-user/your-repo.git
```

Now you have one command to remember: `./setup.sh`.

Putting it in a repo
--------------------

1.  Create a new repo, e.g. _git\_github\_configurations_.
    
2.  Add:
    

    *   `setup_github_single.sh`
        
    *   `setup_github_multi.sh`
        
    *   `setup.sh`
        
    *   A README.md explaining how it all works.
    

3\. Make the scripts executable:

```python
chmod +x setup.sh setup_github_single.sh setup_github_multi.sh
```

4\. Commit and push.

Using it on a new machine
-------------------------

On a fresh Mac or Linux box:

```python
git clone git@github.com:<you>/git_github_configurations.git
cd git_github_configurations
./setup.sh
```

Then:

*   Choose 1 for a single account, or 2 for multiple accounts.
    
*   Answer the prompts (name, email, base directories, etc.).
    
*   Copy the printed public key(s) and add them to:
    
    *   GitHub → Settings → SSH and GPG keys.
    

From then on:

*   `git pull` and `git push` will use SSH keys, not passwords.
    
*   On macOS, the key passphrase can live in the keychain.
    
*   With multiple accounts, Git will pick the right identity based on where the repo lives.
    

Why this matters
----------------

Small bits of automation like this make a big difference over time:

*   You avoid mistakes when setting up new machines.
    
*   You keep your workflow consistent across macOS and Linux.
    
*   You can support multiple GitHub accounts without manual config juggling.
    
*   You free your brain to focus on actual coding instead of redoing setup steps.
    

If you want to go further, you can plug this into your dotfiles or bootstrap scripts so that every new machine starts from a known-good Git & GitHub setup.

Learn the manual way (optional)
-------------------------------

If you’d like to understand all the steps we automated, here are more detailed guides:

*   [Mac: Configure Your MacBook To Use More Than One GitHub Account](https://gist.github.com/BolderLearnerTechSchool/3d09b8301a9a956542846860d005f0ab#Step-2-Ensure-ssh-agent-is-running-and-keys-load-automatically)
    
*   [Linux: Configure Your Computer To Use More Than One GitHub Account](https://gist.github.com/BolderLearnerTechSchool/0086026d177af7320a8fa3e707fb3f30)
    

Once you’re comfortable with the manual process, the automation scripts feel a lot less “magic” and more like a codified checklist.

Conclusion
----------

Automating your Git and GitHub configuration is one of those quality-of-life improvements that pays off every time you spin up a new machine — or switch between multiple GitHub identities.

You don’t need a big framework or dotfiles mega-repo. A small, focused setup repo with three scripts is enough to:

*   Standardize your Git identity.
    
*   Manage SSH keys cleanly.
    
*   Handle single or multiple GitHub accounts.
    
*   Make every new machine feel familiar in minutes.
    

If you build your own version of these scripts, consider keeping them in a public repo and writing your own walkthrough — someone else will definitely find it useful.