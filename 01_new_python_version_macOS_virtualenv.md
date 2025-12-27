# Install A Different Version Of Python From The Default in MacOS And Work With Virtual Environments
![Photo by Christina Morillo: https://www.pexels.com/photo/person-typing-on-macbook-pro-1181282/](https://images.pexels.com/photos/1181282/pexels-photo-1181282.jpeg)

If you run:

```python
python --version
```

in your macOS terminal, you will notice that it comes with Python already. For me, it was _Python 3.9.6 (default, Dec 2 2025, 07:27:59)_. 

As of today, Dec 21, 2025, we have Python 3.14.2. _v3.14_ is obviously a mighty upgrade from _v3.9_, something you may be interested in doing. And there are valid convincing reasons why you'd want to consider an upgrade ASAP (feel free to look up the changes online).

macOS ships with its own system Python, which should not be modified or replaced.

 Instead, the correct approach is to install newer Python versions alongside the system one and control which version is used.

This tutorial uses `pyenv`, the safest and most flexible way to manage Python versions on macOS.

If you would like the functional script that you can use directly, see the repo [install_new_python_version_in_your_os](https://github.com/GitauHarrison/install_new_python_version_in_your_os).

### Table of Contents

- [Install A Different Version Of Python From The Default in macOS And Work With Virtual Environments](#install-a-different-version-of-python-from-the-default-in-macos-and-work-with-virtual-environments)
- [Install pre-requisites](#install-pre-requisites)
  - [Xcode Command Line Tools](#xcode-command-line-tools)
  - [Install and update homebrew](#install-and-update-homebrew)
  - [Install pyenv](#install-pyenv)
- [Install the latest Python version](#install-the-latest-python-version)
  - [Add scope](#add-scope)
  - [Possible error](#possible-error)
    - [Step 1: Install Required Dependencies](#step-1-install-required-dependencies)
    - [Step 2: Reinstall Python 3142 Correctly](#step-2-reinstall-python-3142-correctly)
    - [Step 3: Verify the Fix](#step-3-verify-the-fix)
  - [Confirm Python version](#confirm-python-version)
  - [Why you should fix this](#why-you-should-fix-this)
  - [Prevent this in the future](#prevent-this-in-the-future)
- [Virtual Environments](#virtual-environments)
  - [Option A: Traditional venv (Built-in Python)](#option-a-traditional-venv-built-in-python)
  - [Option B: pyenv-virtualenv (Recommended)](#option-b-pyenv-virtualenv-recommended)
    - [Understand scopes when working with pyenv-virtualenv](#understand-scopes-when-working-with-pyenv-virtualenv)
    - [How pyenv "Activates" a Virtual Environment](#how-pyenv-activates-a-virtual-environment)
    - [Local Scope (Recommended for Projects)](#local-scope-recommended-for-projects)
  - [Sample project: Working with pyenv-virtualenv](#sample-project-working-with-pyenv-virtualenv)
    - [Create virtual environment](#create-virtual-environment)
    - [Change directory](#change-directory)
    - [Explicitly "activate" it in the local scope](#explicitly-activate-it-in-the-local-scope)
    - [Activate and deactivate your virtual environment](#activate-and-deactivate-your-virtual-environment)
- [Key Concept to Remember](#key-concept-to-remember)
- [Cleaning up your virtual environments](#cleaning-up-your-virtual-environments)


## Install pre-requisites

There are a few things you need to do to get started

### Xcode Command Line Tools
`xcode` installs Apple's developer tools required to compile software. It includes `gcc`, `make`, and system headers and is required to build Python from source.

Run:

```python
xcode-select --install
```

### Install and update homebrew
homebrewis a popular, free, open-source package manager for macOS and Linux that simplifies installing and managing software, especially command-line tools, developer dependencies, and even GUI apps, making it easier to get the latest versions of programs not included with the operating system, essentially acting as an "app store for the command line".

Run:

```python
/bin/bash -c "$(curl -fsSL [https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"](https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)")
```

Then update its package list to the latest version:

```python
brew update
```

### Install pyenv
`pyenv` can download and install multiple versions of Python. It allows you to switch between versions without touching the the system Python. It has support for `global `, `local` and session-based version control.

Run:

```python
brew install pyenv
```
Then, add `pyenv` to your shell. If you do not know what shell you use, you can run this command in your terminal:

```python
echo $SHELL
```

Knowing what shell you have will allow you to update the PATH, ensuring `pyenv`'s Python versions take priority. By default, macOS uses `zsh`.

Now, you need to add pyenv to your shell (this will update `~/.zshrc`) . Run these commands in your terminal:

```python
export PYENV_ROOT="$HOME/.pyenv"

export PATH="$PYENV_ROOT/bin:$PATH"

eval "$(pyenv init -)"
```

- `PYENV_ROOT`: tells `pyenv` where it lives
- `PATH`: ensures `pyenv`'s Python versions take priority
- `pyenv init`: enables version switching automatically

To apply the changes, run:

```python
exec $SHELL
```

This will reload your shell making `pyenv` active immediately. It also ensures python resolves to `pyenv`'s managed versions.

Alternatively, you can open and update your `~/.zshrc` file using your text editor as follows:

```python
# Find out if you have the file .zshrc
ls -a ~               # you should see .zshrc

# If there, open it using nano
nano ~/.zshrc

# Paste this in the .zshrc that has just opened
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"

# Close the file by pressing ctrl + X and then 'y'

# Reload the file to apply the changes
exec $SHELL
```

## Install the latest Python version
Optionally, before you install any, you can list available versions by running:

```python
pyenv install -l | grep -E "^[ ]*3\.[0-9]+\.[0-9]+$" | tail
```

This lists installable CPython versions, filters only stable Python 3 release and shows the latest ones at the bottom.

And now, to install the latest version, run:

```python
pyenv install 3.14.2
```

It will download from source, compile it using system tools and intall it under `~/.pyenv/versions/`.

### Add scope
A great feature of `pyenv` is not only its ability to manage version of Python but also the freedom it provides you to determine what scope the downloaded version will run under. For example, use:

- `pyenv global <version>` for all global environments
- `pyenv local <version>` per-project folder
- `pyenv shell <version>` for the current shell session

In my case, I will make the latest version global by running:

```python
pyenv global 3.14.2
```

This will apply to all shells, will be used when no local or shell version is set, and will be stored in `~/.pyenv/versions/`.

### Possible error
During your installation, you may come across this error:

```python
Traceback (most recent call last):
  File "<string>", line 1, in <module>
    import lzma
  File "/Users/gitau/.pyenv/versions/3.14.2/lib/python3.14/lzma.py", line 28, in <module>
    from _lzma import *
ModuleNotFoundError: No module named '_lzma'
WARNING: The Python lzma extension was not compiled. Missing the lzma lib?
Installed Python-3.14.2 to /Users/gitau/.pyenv/versions/3.14.2
```

This is a c**ommon and well-understood macOS + pyenv issue**, and the good news is: **Your Python 3.14.2 did install**, but it was built **without LZMA support**.


_What actually happened?_

Python includes a **standard library module** called `lzma `.lzma depends on a system library called `liblzma` (from `xz`). During compilation, Python could not find liblzma. So Python was built **without** the `_lzma` extension

Python works normally, but anything that uses `lzma` (e.g. some compression tools, packaging, data science libraries) will fail.


_Why does this happen in macOS?_

On macOS, liblzma is not installed by default. pyenv builds Python from source. If a dependency is missing → that feature is silently skipped. This is why `pyenv` shows:

```python
Installed Python-3.14.2
```

but also prints a warning.


_So, how do you fix it?_

You **do not patch this after installation**. You must:

- Install missing system libraries
- Rebuild Python with pyenv

#### Step 1: Install Required Dependencies
Install `xz` (provides `liblzma`) and other common Python build deps:

```python
brew install xz openssl readline sqlite3
```

- `xz` → provides liblzma (fixes our error)
- `openssl` → SSL/TLS support
- `readline` → better interactive shell
- `sqlite3` → database support

#### Step 2: Reinstall Python 3.14.2 Correctly
First, remove the broken build:

```python
pyenv uninstall 3.14.2
```

Then reinstall:

```python
pyenv install 3.14.2
```

This time, pyenv will detect liblzma and compile `_lzma`.

#### Step 3: Verify the Fix
Test `lzma` import:

```python
python - <<EOF
import lzma
print("lzma works")
EOF
```

Expected output:

```python
pyenv: python: command not found

The `python' command exists in these Python versions:
  3.14.2

Note: See 'pyenv help global' for tips on allowing both
      python2 and python3 to be found
```

Add a scope to allow both python2 and python3 to be found.


_Set Python 3.14.2 as Default (Global)_

```python
pyenv global 3.14.2
```

Or


_Per-Project Python Version (Local Scope)_

```python
cd your-project
pyenv local 3.14.2
```

This creates a `.python-version` file in the directory (run `.python-version` to see the Python version in this scope). That version is used only in this project and automatically switches when you enter the folder.

Or


_Session_

```python
pyenv shell 3.14.2
```

To apply only to the current terminal session and reset when the terminal closes.

Test `lzma` import again. You should now see:

```python
lzma works
```

### Confirm Python version
You can now verify the globally-installed packages by running these:

```python
python --version
which python
which pip
```
You should see:

```python
Python 3.14.2
/Users/gitau/.pyenv/shims/python
/Users/gitau/.pyenv/shims/pip
```

### Why you should fix this
You might not notice this immediately, but missing `_lzma` can break:

- `pip` wheels and package extraction
- `tarfile` with `.xz`
- Some scientific and ML libraries
- Future Python tooling

### Prevent this in the future
Before installing any Python version with pyenv on macOS, run:

```python
brew install \
  openssl \
  readline \
  sqlite3 \
  xz \
  zlib
```

Then install Python.

## Virtual Environments
Python projects should never share dependancies. You should use virtual environments to isolate packages per project.

### Option A: Traditional venv (Built-in Python)
Traditionally, Python offers a built-in command to create a virtual environment. You can run this to create your first virtual environment:

```python
python3 -m venv venv
```

To use it, you will activate by running:

```python
source venv/bin/activate
```

Once done, you can deactivate this virtual environment by running:

```python
deactivate
```

This method is very built into Python, is simple and widely supported. However, you are tied to the Python version used at creation time and is less flexible for multi-version workflows.

### Option B: `pyenv-virtualenv` (Recommended)
You can also use pyenv to create a virtual environment. Begin by installing `pyenv-virtualenv` :

```python
brew install pyenv-virtualenv
```

Enable it in your shell as follows:

```python
echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.zshrc
exec $SHELL
```

#### Understand scopes when working with pyenv-virtualenv
A virtual environment exists to isolate one project from another. That makes it:

- Wrong for global scope (would affect everything)
- Risky for session scope (easy to forget)
- Perfect for local scope

When a virtualenv is set as `local`:

It activates automatically when you enter the project directory

- It deactivates automatically when you leave
- No manual commands needed day-to-day
- Zero chance of contaminating other projects

This is why most experienced Python developers consider:

_`pyenv local <virtualenv>` the gold standard workflow_

### How pyenv "Activates" a Virtual Environment
This is important to understand conceptually. When you activate a pyenv virtualenv, `pyenv`:

- Updates `which python`, `pip`, etc. are resolved
- Points them to the virtualenv's binaries
- Does _not_ modify your shell the way `source venv/bin/activate` does

Activation is done by **version selection**, not shell hacking. That's why activation can happen automatically.

#### Local Scope (Recommended for Projects)
- The virtualenv is tied to a directory
- pyenv reads .python-version
- Activation happens automatically

### Sample project: Working with pyenv-virtualenv
Let us presume you want to create new project called test. You'd probably begin by creating the project's directory like this:

```python
mkdir test
```
#### Create virtual environment
This creates an empty directory at your current location. While outside this new directory, create your new virtual environment:

```python
pyenv virtualenv 3.14.2 test_venv-3.14
```

A new virtual environment called `test_venv-3.14` tied to our new default Python _v3.14.2_ has been created. You can create as many virtual environments as you want.

>Virtualenvs are not tied to folders. They live in `~/.pyenv/versions/`. Once we localize our virtualenv, the project's folder will simply point to one virtual environment (if you have many created) using .python-version which will be in each project directory.

It is recommended to name your virtual environment based on your project and Python's version number. For example:

```python
pyenv virtualenv 3.14.2 blog-3.14
pyenv virtualenv 3.14.2 sacco-3.14
pyenv virtualenv 3.14.2 school-3.14
```

This makes debugging easy, teaching clearer and team collaboration safer.

#### Change directory
Once you have created your virtual environment, change your directory into the new folder:

```python
cd test
```

At this point, no env is active nor do we have `.python-version` yet. Remember, the file `.python-version` (which comes once you localize your virtual environment) is used to specify what virtual environment to use.

#### Explicitly "activate" it in the local scope
Run:

```python
pyenv local test_venv
```

This will create the `.python-version` file showing you what virtual environment you are using in your project. 

#### Activate and deactivate your virtual environment
Now, there is no such thing as "activating" or "deactivating" your virtual environment when working with `pyenv-virtualenv`. 

To activate, simply change directory into your project's folder, in this case, the project folder is called _test_. In my terminal, if I cd test, my virtual environment `test_venv-3.14` will be activated automatically.

#### change directory

```python
# change directory
cd test

# notice how the terminal changes to 
(test_vevn-3.14)$
```

To deactivate, simply get out of the project folder:

```python
cd ..
```

You will notice that the terminal changes from `(test_venv-3.14)$` to the original `$`. 

### Key Concept to Remember
Virtualenvs live globally, activation is local.

- Virtualenvs → `~/.pyenv/versions/`
- Projects → `.python-version` files
- Names → global identifiers

### Cleaning up your virtual environments
Deleting a project folder does not delete its virtual environment. This is by design because:

- Virtualenvs live globally in` ~/.pyenv/versions`
- Project folders only reference them via `.python-version`
- `pyenv` cannot guess whether an environment is still needed elsewhere

So cleanup is **your responsibility**.

You should not do this at all:

```python
rm -rf ~/.pyenv/versions
```

This would:

- Delete all Python versions
- Delete all virtualenvs
- Break pyenv completely
- Force you to reinstall Python

The correct way would be to use pyenv commands. First, list all availabe versions:

```python
pyenv versions

# Output
3.14.2 (set by /Users/gitau/.pyenv/version)
3.14.2/envs/arc-3.14
3.14.2/envs/first_env_with_pyenv
3.14.2/envs/test_virtualenv
arc-3.14 --> /Users/gitau/.pyenv/versions/3.14.2/envs/arc-3.14
first_env_with_pyenv --> /Users/gitau/.pyenv/versions/3.14.2/envs/first_env_with_pyenv
```

Virtualenvs appear **alongside** base Python versions.

To delete a specific virtual environment, run:

```python
pyenv uninstall arc-3.14
```

This safely:

- Removes the virtualenv directory
- Cleans up pyenv metadata
- Prevents broken references

You can delete multiple virtualenvs one by one. You can also bulk-delete as follows:

```python
# Identify base Python versions
pyenv versions --bare

# Base versions look like:
# 3.14.2
# 3.13.1

# Remove all virtualenvs interactively
pyenv virtualenvs

# Then uninstall selectively:
pyenv uninstall <env-name>
```

There is no official "remove all virtualenvs" command - this is intentional.

### Just remember:

_Use `pyenv uninstall` for cleanup - never `rm -rf ~/.pyenv/versions`._
