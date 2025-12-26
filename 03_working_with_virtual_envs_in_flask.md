# Working with Virtual Environments in Flask

When building Flask applications, getting your **Python version**, **virtual environment**, and **dependency management** right early saves a lot of pain later.

This note focuses on three tools you will see a lot in modern Flask work:

- **pyenv** – chooses which Python version you use
- **pyenv-virtualenv** – creates per-project virtual environments on top of pyenv
- **Poetry** – manages dependencies and (optionally) virtual environments

We’ll walk through two real-world scenarios:

1. **Starting a brand new Flask project** – choosing between:
   - pyenv + pyenv-virtualenv + `pip` / `requirements.txt`
   - pyenv + Poetry
   - Poetry **without** pyenv
2. **Working on an existing Flask app** that currently uses `requirements.txt` (maybe with `python3 -m venv venv`) and migrating to:
   - pyenv / pyenv-virtualenv
   - Poetry (with or without pyenv)

Throughout, we’ll be mindful of operating systems:

- **macOS** – often uses Homebrew for Python, pyenv, and pipx.
- **Ubuntu / Linux** – often uses `apt` + `pyenv` built from source.
- **Windows** – best experience is usually **WSL (Ubuntu on Windows)**, so we’ll treat that like Ubuntu.

The focus here is **environment and dependency management**, not your Flask routes or templates.

---

## 1. Concepts: Python version vs virtual env vs dependencies

There are three separate layers you should keep straight:

1. **Python version** (interpreter)
   - Example: `Python 3.11.9`, `Python 3.12.6`.
   - Managed globally by the OS (system `python3`), by **pyenv**, or by a package manager like Homebrew/apt.

2. **Virtual environment** (isolated site-packages)
   - Example: `.venv/` created by `python -m venv`, or a pyenv-virtualenv like `my-app-env`.
   - Keeps each project’s packages separate.

3. **Dependencies** (packages like `Flask`, `SQLAlchemy`, etc.)
   - Installed into the active virtualenv via `pip` or Poetry.
   - Usually tracked by `requirements.txt` or `pyproject.toml` + `poetry.lock`.

You can mix and match tools. For example:

- Use **pyenv** for Python version + **Poetry** for dependencies and env.
- Use **system Python** + `python -m venv` + `requirements.txt`.
- Use **pyenv-virtualenv** + `pip` only.

The rest of this note shows concrete flows and explains what each step does.

---

## 2. Scenario A – New Flask project

You’re starting from scratch and want a good, repeatable way to manage your Python version, virtualenv, and dependencies.

We’ll show three common setups:

1. **pyenv + pyenv-virtualenv + `pip` + `requirements.txt`**
2. **pyenv + Poetry (recommended)**
3. **Poetry without pyenv**

### 2.1 pyenv + pyenv-virtualenv + `pip` + `requirements.txt`

This is a good choice when:

- You want explicit control over Python versions with pyenv.
- You or your infra still expect `requirements.txt`.

#### 2.1.1 Install pyenv (high level)

Full instructions are in this separate note:

- [Using a newer Python version on macOS with pyenv & virtualenv](./01_new_python_version_macOS_virtualenv.md)

But in short:

- **macOS (Homebrew)** – typical steps:

  ```bash path=null start=null
  brew install pyenv pyenv-virtualenv
  ```

  Then add the init snippet to your shell (`~/.zshrc` or `~/.bashrc`) as described in that note.

- **Ubuntu / WSL** – typical steps:

  - Install build deps: `sudo apt install build-essential curl git ...`
  - Clone pyenv: `git clone https://github.com/pyenv/pyenv.git ~/.pyenv`
  - Clone pyenv-virtualenv: `git clone https://github.com/pyenv/pyenv-virtualenv.git ~/.pyenv/plugins/pyenv-virtualenv`
  - Add the init snippet to `~/.bashrc` or `~/.zshrc`.

After installation, restart your shell and verify:

```bash path=null start=null
type pyenv
pyenv --version
```

#### 2.1.2 Choose a Python version with pyenv

```bash path=null start=null
pyenv install 3.11.9         # or another version you need
mkdir my_flask_app
cd my_flask_app
pyenv local 3.11.9
```

- `pyenv install 3.11.9` – builds and installs that Python version under `~/.pyenv/versions`.
- `mkdir my_flask_app` / `cd my_flask_app` – create and enter your project directory.
- `pyenv local 3.11.9` – writes a `.python-version` file in this directory so that `python`/`pip` here use pyenv’s 3.11.9.

You can confirm:

```bash path=null start=null
python -V
which python
pyenv version
```

#### 2.1.3 Create a pyenv virtualenv for the project

```bash path=null start=null
pyenv virtualenv 3.11.9 my-flask-env
pyenv local my-flask-env
```

- `pyenv virtualenv 3.11.9 my-flask-env` – creates a new virtualenv named `my-flask-env` based on Python 3.11.9.
- `pyenv local my-flask-env` – updates `.python-version` so this directory auto-activates `my-flask-env` when you `cd` into it.

Now, simply entering this directory activates the env (no `source .venv/bin/activate` needed):

```bash path=null start=null
cd my_flask_app
python -V      # should show 3.11.9 in the pyenv env
which python   # should point inside ~/.pyenv/versions/my-flask-env/bin
```

#### 2.1.4 Install Flask and create `requirements.txt`

```bash path=null start=null
pip install Flask python-dotenv Flask-Mail
pip freeze > requirements.txt
```

- `pip install ...` – installs packages into the active pyenv virtualenv.
- `pip freeze > requirements.txt` – captures all installed packages + exact versions into `requirements.txt`.

To recreate this environment elsewhere:

```bash path=null start=null
cd my_flask_app
pyenv local my-flask-env    # or recreate the env with pyenv virtualenv
pip install -r requirements.txt
```

From here, you run your Flask app in the usual way (`flask run`, `python main.py`, etc.), with pyenv-virtualenv ensuring the right env is active.

---

### 2.2 pyenv + Poetry (recommended for new projects)

Here we let:

- **pyenv** select the Python version.
- **Poetry** manage dependencies and its own virtualenv.

This is a very modern and portable setup.

#### 2.2.1 Ensure pyenv is set to the Python you want

```bash path=null start=null
pyenv install 3.11.9               # if not already installed
mkdir my_flask_app
cd my_flask_app
pyenv local 3.11.9
python -V                          # should be 3.11.9
```

#### 2.2.2 Install pipx and Poetry (OS-specific)

Before installing Poetry, it helps to understand **pipx**:

- `pipx` is a small tool that installs Python applications (like Poetry, `black`, `ruff`, etc.) in **their own isolated environments**, and then puts lightweight shims on your `PATH`.
- This means:
  - The tools themselves don’t pollute your project’s virtualenv.
  - You can upgrade or remove a tool without touching your app dependencies.

You install `pipx` once per machine, then use it to install Poetry and other global CLI tools.

**On macOS (Homebrew):**

```bash path=null start=null
brew install pipx
pipx ensurepath
# restart shell (close and reopen your terminal)
pipx --version
pipx install poetry
```

- `brew install pipx` – installs the `pipx` program.
- `pipx ensurepath` – adds the directory where pipx stores its shims (usually `~/.local/bin`) to your shell `PATH`.
- After restarting the shell, `pipx --version` confirms it’s available.
- `pipx install poetry` – installs Poetry into its own isolated environment and exposes the `poetry` command.

**On Ubuntu / WSL:**

```bash path=null start=null
sudo apt install pipx
pipx ensurepath
# or, if apt doesn't have pipx:
python3 -m pip install --user pipx
python3 -m pipx ensurepath
# restart shell
pipx --version
pipx install poetry
```

- `sudo apt install pipx` – installs pipx from your distro, if available.
- Or `python3 -m pip install --user pipx` – installs pipx just for your user.
- `pipx ensurepath` – updates your `PATH` so pipx-installed tools are visible.
- `pipx install poetry` – installs Poetry in an isolated way, separate from your app envs.

Once this is done, you can run `poetry` from any terminal without activating a venv first.

#### 2.2.3 Initialize Poetry in the project

From the project directory:

```bash path=null start=null
poetry init
```

Poetry walks you through prompts (name, version, description, Python range, etc.). A typical interactive session looks like this:

```text path=null start=null
This command will guide you through creating your pyproject.toml config.

Package name [my-flask-app]: my_flask_app
Version [0.1.0]: 0.1.0
Description []: Simple Flask demo app
Author [Your Name <you@example.com>, n to skip]: Your Name <you@example.com>
License []: Proprietary (BLC)
Compatible Python versions [^3.11]: >=3.11,<4.0

Would you like to define your main dependencies interactively? (yes/no) [yes]: no
Would you like to define your development dependencies interactively? (yes/no) [yes]: no
```

What each of these means:

- **Package name** – an internal identifier; for apps you can use something like `my_flask_app`.
- **Version** – the current version of your app (`0.1.0` is a good starting point). If you follow semantic versioning:
  - `MAJOR.MINOR.PATCH`, e.g. `1.2.3`.
- **Description** – a short human-readable summary; helps future you and teammates.
- **Author** – name/email; for BLC projects, you can use a team address like `Bolder Learner Consulting <support@bolderlearner.com>`.
- **License** – free text. If you have a **custom/proprietary license**, type something like `Proprietary (BLC)`. The full legal text still lives in `LICENCE`.
- **Compatible Python versions** – a constraint like `>=3.11,<4.0` tells Poetry which Python versions this project supports. This is important for collaborators and CI.
- **Interactive dependencies** – for an existing or more complex app, it’s usually easier to answer `no` here and add dependencies later with `poetry add` (or import from `requirements.txt`).

After you answer these prompts, Poetry writes a `pyproject.toml` file containing this metadata.

Because this app is **not a published library package**, you can optionally disable package mode (if you only want Poetry for env + dependencies):

```toml path=null start=null
[tool.poetry]
package-mode = false
```

This tells Poetry not to try to install a root package and instead just manage dependencies and the virtualenv.

#### 2.2.4 Add Flask and other dependencies

```bash path=null start=null
poetry add Flask python-dotenv Flask-Mail
```

- Updates `pyproject.toml` with these dependencies.
- Creates/updates `poetry.lock` with exact versions.
- Installs them into Poetry’s virtualenv for this project.

#### 2.2.5 Install and run the app

For you or any collaborator:

```bash path=null start=null
poetry install
poetry run flask run
```

- `poetry install` – creates/reuses the project’s virtualenv and installs all dependencies.
- `poetry run flask run` – runs `flask run` inside that env.

You can also open a subshell inside the env:

```bash path=null start=null
poetry shell
flask run
```

---

### 2.3 Poetry without pyenv

You don’t **have** to use pyenv. For small or throwaway projects, or when the system Python is already what you want, you can rely on:

- **macOS** – `python3` from Homebrew or the OS.
- **Ubuntu / WSL** – `python3` from `apt`.

#### 2.3.1 Install Poetry via pipx as above

Follow the pipx + `pipx install poetry` steps from 2.2.2, but skip any pyenv commands.

#### 2.3.2 Create and configure the project

```bash path=null start=null
mkdir my_flask_app
cd my_flask_app
poetry init
# answer prompts, set Python version like ">=3.11,<4.0"
```

Optionally, keep the virtualenv inside the project:

```bash path=null start=null
poetry config virtualenvs.in-project true
```

Tell Poetry which Python to use (usually just `python3`):

```bash path=null start=null
poetry env use python3
```

Then add dependencies and run just like in 2.2:

```bash path=null start=null
poetry add Flask python-dotenv Flask-Mail
poetry install
poetry run flask run
```

If later you decide to adopt pyenv, you can **first** set `pyenv local ...` in the project directory, then re-run `poetry env use python3` to make Poetry’s env use that interpreter.

---

## 3. Scenario B – Existing Flask app using `requirements.txt`

Now consider a project you already have:

- Uses `python3 -m venv venv` or `.venv`.
- Tracks dependencies with `requirements.txt` (maybe from `pip freeze`).

You want to move toward a more robust setup using pyenv and/or Poetry.

We’ll cover three paths:

1. **Stay on `requirements.txt` but adopt pyenv / pyenv-virtualenv**
2. **Migrate to Poetry + pyenv**
3. **Migrate to Poetry without pyenv**

### 3.1 Keep `requirements.txt` but use pyenv-virtualenv

This is a minimal change that improves Python version control without changing how you manage dependencies.

1. **Verify the current app works** (with your existing venv):

   ```bash path=null start=null
   python3 -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt
   flask run
   ```

2. **Install and configure pyenv + pyenv-virtualenv** (see 2.1.1 and the separate macOS note).

3. **Create a pyenv virtualenv for this project**:

   ```bash path=null start=null
   cd /path/to/your/flask-app
   pyenv install 3.11.9        # if needed
   pyenv virtualenv 3.11.9 my-existing-app-env
   pyenv local my-existing-app-env
   ```

   - This ensures that whenever you `cd` into this project, the `my-existing-app-env` virtualenv is active.

4. **Install dependencies into the pyenv env**:

   ```bash path=null start=null
   pip install -r requirements.txt
   flask run
   ```

   At this point you still use `requirements.txt`, but your Python version and env are controlled by pyenv.

### 3.2 Migrate an existing app to Poetry + pyenv

This is the pattern you’re adopting in your BLC Flask apps.

1. **Ensure the app currently works** (same as step 1 above).

2. **From the project root, initialize Poetry**:

   ```bash path=null start=null
   # deactivate any old venv first
   deactivate 2>/dev/null || true

   pyenv install 3.11.9          # if needed
   pyenv local 3.11.9

   poetry init
   ```

   - Answer the prompts
     - Keep name/version simple.
     - Set a realistic Python range (e.g. `>=3.11,<4.0`).
     - **Skip adding dependencies interactively**, since we’ll import from `requirements.txt`.
   - Optionally add:

     ```toml
     [tool.poetry]
     package-mode = false
     ```

     so Poetry is used only for dependencies/env, not packaging.

3. **Import dependencies from `requirements.txt`**:

   ```bash path=null start=null
   while read -r dep; do
       poetry add "$dep"
   done < requirements.txt
   ```

   - This walks each `package==version` line and calls `poetry add` for it, preserving pins.

4. **Install via Poetry using pyenv’s interpreter**:

   ```bash path=null start=null
   poetry env use python3   # this python is the one pyenv chose via pyenv local
   poetry install
   ```

5. **Run the app**:

   ```bash path=null start=null
   poetry run flask run
   ```

6. **Optionally regenerate `requirements.txt` from Poetry**:

   ```bash path=null start=null
   poetry export -f requirements.txt -o requirements.txt --without-hashes
   ```

   Now `pyproject.toml` / `poetry.lock` are the source of truth; `requirements.txt` is generated when needed.

### 3.3 Migrate to Poetry without pyenv

If you don’t want to use pyenv on a particular machine (or can’t), you can still migrate the same app to Poetry using the system `python3`.

1. **Verify the current setup works** (as before).
2. **Install pipx + Poetry** (Section 2.2.2) without any pyenv steps.
3. **Initialize Poetry in the project**:

   ```bash path=null start=null
   deactivate 2>/dev/null || true
   poetry init
   # answer prompts; set Python version like ">=3.11,<4.0"
   ```

4. **Import dependencies from `requirements.txt`** using the same `while read -r dep; do ...` loop.
5. **Tell Poetry which Python interpreter to use** (usually just `python3`):

   ```bash path=null start=null
   poetry env use python3
   poetry install
   ```

6. **Run the app**:

   ```bash path=null start=null
   poetry run flask run
   ```

Later, if you adopt pyenv, you can set `pyenv local` in the project directory and re-run `poetry env use python3` to switch Poetry’s env to that interpreter.

---

## 4. OS notes: macOS, Ubuntu, and Windows (WSL)

- **macOS**
  - Use Homebrew for Python, pyenv, pipx, and Poetry.
  - Typical pattern:

    ```bash path=null start=null
    brew install pyenv pyenv-virtualenv pipx
    pipx ensurepath
    pipx install poetry
    ```

- **Ubuntu / Linux**
  - Use `apt` for base Python and pipx, and git+build deps for pyenv:

    ```bash path=null start=null
    sudo apt install python3 python3-venv python3-pip pipx
    pipx ensurepath
    pipx install poetry
    # pyenv install per official docs (or your 01_new_python_version... note)
    ```

- **Windows**
  - Strongly recommended: **WSL (Ubuntu)**, then follow the Ubuntu steps above inside the WSL terminal.
  - Avoid mixing Windows-native Python and WSL Python for the same project.

---

## 5. Choosing a default workflow for your Flask apps

For serious Flask work going forward, a solid, teachable default is:

- Use **pyenv** to select the Python version (per project via `pyenv local`).
- Use **Poetry** to manage dependencies and the virtualenv.

A typical pattern you can reuse:

```bash path=null start=null
# One-time per machine
brew install pyenv pyenv-virtualenv pipx      # macOS example
pipx ensurepath
pipx install poetry

# Per project
mkdir my_flask_app
cd my_flask_app
pyenv install 3.11.9          # if needed
pyenv local 3.11.9
poetry init                   # answer prompts, skip adding deps interactively
# (optionally add [tool.poetry] package-mode = false)
poetry add Flask python-dotenv Flask-Mail
poetry install
poetry run flask run
```

You can then adapt this same pattern to existing projects by importing from `requirements.txt` as shown above.
