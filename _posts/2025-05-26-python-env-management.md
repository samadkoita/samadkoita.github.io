---
title: Practical guide to Python environment management
description: Without the jargon
author: samadkoita
date: 2025-05-26 14:10:00 +0800
categories: [Tech]
tags: []
---

You run `pip install some-package` and suddenly your working project breaks with cryptic import errors. 
I've had this issue multiple times — trying out open-source projects, following setup instructions, only to end up with dependency conflicts that make no sense.

The latest case was while setting up a new project that used uv, the new Rust-based Python tool. I had already created a Conda environment for the project, but the setup instructions also included creating a virtual environment and using uv pip sync and other uv-specific commands. I followed the steps, but ended up with a bunch of dependency errors that left me pretty confused. Some things were being installed in unexpected places, and tools were stepping on each other.

That's when I realized I've been working with these tools for a long time — pip, virtualenv, conda, pyenv, etc. — but I've never really taken the time to understand how they all fit together from the ground up.

So I decided to start fresh: from what happens when you first install Python on your system, to how environments work, how different tools manage dependencies, and where things tend to go wrong.

## Installing Python

I installed Python using the homebrew command on my Mac.

```bash
brew install python@3.11
```

This installs the Python binaries under Homebrew's Cellar (e.g., `/opt/homebrew/Cellar/python@3.11/`) and symlinks the `python3` and `pip3` executables into `/opt/homebrew/bin/`.

```bash
which python3
```

When you run `python3`, your system looks through the folders listed in your `$PATH` to find the right Python program to run. That Python executable knows where to find its built-in modules (like `os` and `math`) and also the folder where additional libraries (like `requests`, `numpy`, etc.) are stored. That folder is called `site-packages`.

To check where Python will install or load packages from:

```bash
python3 -m site
# samadkoita@Samads-MacBook-Pro python-deepsearch % python3 -m site 
# sys.path = [
#     '/Users/samadkoita/Desktop/python-deepsearch',
#     '/opt/homebrew/Cellar/python@3.11/3.11.12_1/Frameworks/Python.framework/Versions/3.11/lib/python311.zip',
#     '/opt/homebrew/Cellar/python@3.11/3.11.12_1/Frameworks/Python.framework/Versions/3.11/lib/python3.11',
#     '/opt/homebrew/Cellar/python@3.11/3.11.12_1/Frameworks/Python.framework/Versions/3.11/lib/python3.11/lib-dynload',
#     '/opt/homebrew/lib/python3.11/site-packages',
#     '/opt/homebrew/opt/python@3.11/Frameworks/Python.framework/Versions/3.11/lib/python3.11/site-packages',
# ]
```

In the list above:
- The paths that include `lib/python3.11` (for example, `/opt/homebrew/.../lib/python3.11`) are where Python's default, built-in modules live — things like `os`, `math`, and `json`.
- The paths that end in `site-packages` are where third-party libraries go — these are the packages you install yourself using pip, like `requests` or `pdbpp`.

**Built-ins:**
```bash
samadkoita@Samads-MacBook-Pro python-deepsearch % ls /opt/homebrew/Cellar/python@3.11/3.11.12_1/Frameworks/Python.framework/Versions/3.11/lib/python3.11/*.py | head
/opt/homebrew/Cellar/python@3.11/3.11.12_1/Frameworks/Python.framework/Versions/3.11/lib/python3.11/__future__.py
/opt/homebrew/Cellar/python@3.11/3.11.12_1/Frameworks/Python.framework/Versions/3.11/lib/python3.11/__hello__.py
/opt/homebrew/Cellar/python@3.11/3.11.12_1/Frameworks/Python.framework/Versions/3.11/lib/python3.11/_aix_support.py
/opt/homebrew/Cellar/python@3.11/3.11.12_1/Frameworks/Python.framework/Versions/3.11/lib/python3.11/_bootsubprocess.py
/opt/homebrew/Cellar/python@3.11/3.11.12_1/Frameworks/Python.framework/Versions/3.11/lib/python3.11/_collections_abc.py
/opt/homebrew/Cellar/python@3.11/3.11.12_1/Frameworks/Python.framework/Versions/3.11/lib/python3.11/_compat_pickle.py
/opt/homebrew/Cellar/python@3.11/3.11.12_1/Frameworks/Python.framework/Versions/3.11/lib/python3.11/_compression.py
/opt/homebrew/Cellar/python@3.11/3.11.12_1/Frameworks/Python.framework/Versions/3.11/lib/python3.11/_markupbase.py
/opt/homebrew/Cellar/python@3.11/3.11.12_1/Frameworks/Python.framework/Versions/3.11/lib/python3.11/_osx_support.py
/opt/homebrew/Cellar/python@3.11/3.11.12_1/Frameworks/Python.framework/Versions/3.11/lib/python3.11/_py_abc.py
```

**Third-party:**
```bash
samadkoita@Samads-MacBook-Pro python-deepsearch % ls /opt/homebrew/Cellar/python@3.11/3.11.12_1/Frameworks/Python.framework/Versions/3.11/lib/python3.11/site-packages | grep pdbpp
_pdbpp_path_hack
pdbpp_hijack_pdb.pth
pdbpp_utils
pdbpp-0.11.6.dist-info
pdbpp.py
```

Every new third party library I now add is added to this site-packages path.

## The First Collision – Global site-packages

After installing Python and using it for a while, I had added a bunch of libraries globally using pip install - I installed `requests`, `pandas` and it worked fine. Let's say I want to run an open source repository on GitHub. This uses a different version of pandas. I installed the new packages, but now when I run the old project, things start breaking. I realize all the packages I installed using pip were going into the same folder.

If only I could separate out packages for different projects, so that project A can use one set of versions, project B can use the other. That's where virtual environments come in.

## Virtual Environments 101

To avoid the mess of global site-packages, Python gives us virtual environments — a way to keep dependencies isolated to each project.

At a basic level, it's just a self-contained folder that has:
- A copy (or link) to the Python interpreter
- A fresh, empty site-packages folder for third-party libraries
- Scripts like pip and python that point to the environment instead of the system-wide ones

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install requests
```

Now, your terminal's `python` and `pip` commands will use the ones inside `.venv/`, not the global ones.
All third party packages are now installed in `.venv/lib/python3.x/site-packages/`.

```bash
python -m site
```

You can verify the environment prefix by running `import sys; print(sys.prefix)` in your virtual environment.

There are also other core tools that are part of the pip related workflows like (pip-tools, etc.)

## Multiple Pythons? – Managing Different Python Versions

At some point, I needed to run an older open-source project that hadn't been updated in a while. I cloned the repo, followed the setup instructions, and immediately ran into a wall — the project required Python 3.9, but my system had Python 3.11 installed.

I tried installing the dependencies anyway, but a few of them failed to build, and others complained about version mismatches. Even though I was using a virtual environment, it didn't help — the virtual environment was still using the same Python binary as my system.

To really fix this, I needed a way to have multiple versions of Python installed side-by-side, and use the correct one for each project.

I downloaded and installed Python 3.9 using `brew install python3.9`. This gave me access to a separate `python3.9` binary alongside my system's `python3.11`. From there, I could create a virtual environment with the exact version I needed:

I could also install dependencies using `python3.9 -m pip install -r requirements.txt`, that would write the dependencies into the corresponding folder in `/opt/homebrew/…/python3.9/`, or create a virtual environment and (`python3.9 -m venv .venv` then `source .venv/bin/activate`)

This worked — but managing different versions like this quickly becomes tedious, especially when switching between projects or setting up new ones. That's where pyenv comes in.

pyenv simplifies the whole process by letting you install and manage multiple Python versions in a more organized way. Instead of manually downloading different versions and keeping track of paths, I could just run:

```bash
pyenv install 3.9
pyenv local 3.9
```

Whether done manually or using pyenv, the key idea here is: virtual environments isolate packages, but to isolate Python itself, you need to control which interpreter you're using in the first place.

## System-Level Divergence – When venv Isn't Enough

At some point, I was working on two different machine learning projects. Both used PyTorch — but one needed CUDA 11.8, and the other was stuck on CUDA 10.2 due to driver compatibility on a shared GPU server. I figured I could manage this by creating separate virtual environments for each project, like I always do.

So I created two `.venv` folders, activated each one, installed the right PyTorch version using pip. It didn't work as expected.

Both environments kept picking up the same system-installed CUDA libraries — the ones linked in `/usr/local/cuda/`. Even though the Python packages were isolated, they were still depending on shared system-level libraries underneath. Changing versions in one project affected the other, and I couldn't get both projects working cleanly at the same time. I had to resort to manually updating the CUDA path each time I was switching between projects.

Virtual environments isolate Python packages, but not system-level dependencies. For this, we need a tool that can manage both Python and system libraries together. Here is where conda can provide unique value.

When I first started using Conda, I assumed it was just another way to manage Python environments — like venv, but with a different command. What I didn't realize at the time is that Conda solves an entirely different class of problems.

Conda environments isolate everything: the Python interpreter, the libraries you install, and the system-level dependencies they rely on.

With conda you can create two environments:

```bash
conda create -n torch11 python=3.10 pytorch torchvision pytorch-cuda=11.8 -c pytorch -c nvidia
conda create -n torch10 python=3.8 pytorch torchvision cudatoolkit=10.2 -c pytorch -c nvidia
```

This removed the need for handling CUDA drivers globally.

Conda environments are stored in separate folders, just like virtualenvs, and you activate them with: `conda activate torch11`. From there, you can use Python and install other packages using either `conda install` or `pip`.

**How it works?**
Conda maintains its own internal package repository, and those packages come bundled with all their binary dependencies. Conda will install precompiled versions of those libraries inside the environment — completely isolated from your system. This is different from pip.

`conda install cudatoolkit=11.8` => installed inside your conda env.

When you run this, you are pulling from conda channels and not the default PyPI. This means that not all packages are available via `conda install`. While you can still install them using pip inside a conda env, it may re-introduce the same issues as before and can potentially break a conda env. It's also slow and heavier. It is best used only when your project needs system-level libs.

## Modern Tooling Layer - Smoother Developer Experience

After understanding venv, pip, conda, I started looking for something that is more optimized for a smooth developer experience, the 80% use-case for most day-to-day projects, especially the ones that didn't need system-level dependencies like CUDA or FFmpeg. Poetry and uv were the most popular.

### Poetry

Poetry is a very popular tool to manage Python projects, especially if you are building a library that is published. It takes care of the work of multiple separate tools - pip, venv, pip-tools, setuptools, twine, etc. and wraps them into a single workflow. This includes - creating & using a venv, installing & checking dependencies, creating a lock file, managing project metadata, publishing to PyPI.

**Old workflow:**
1. Manually create virtual env
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   ```
2. Install packages with pip
   ```bash
   pip install requests fastapi
   ```
3. Freeze current versions to a requirements file
   ```bash
   pip freeze > requirements.txt
   ```
4. Write a setup.py file
   ```python
   from setuptools import setup
   setup(name='myapp', version='0.1.0', install_requires=['requests', 'fastapi'])
   ```
5. Publish using (twine upload + setup.py)

This uses 4 different tools.

**In poetry, this can be simplified:**
```bash
poetry new myapp
poetry add requests fastapi
```

This automatically creates the virtual env, adds those packages to `pyproject.toml` [requirement that must be present if you want to publish to PyPI], lock exact versions to `poetry.lock`.

Run the app in your virtual environment:
```bash
poetry run python app.py
```

Publish using:
```bash
poetry publish
```

### uv

uv is a new Python packaging tool that acts as a replacement for the following tools:
- Pip (for installing packages)
- Virtual env (for managing environments)
- Pip-tools (generating lock files)

Main advantage is speed and simplicity - it is 10x-100x faster than pip!

It also works with pip-style, and has a very low learning curve & no changes to workflow.

**Create a virtual environment:**
```bash
uv venv .venv
```

**Install packages:**
```bash
uv pip install requests fastapi
```

**Generate a lockfile:**
```bash
uv pip compile requirements.in -o requirements.txt
```

**Reproducibly install everything:**
```bash
uv pip sync
```

It uses the same `pyproject.toml`, `requirements.txt` - no special setup needed.

## When to Use What

After going through this journey, here's my simple decision framework:

- **I'll use `uv`** for most everyday projects - it's faster than pip and just as simple
- **I'll use `venv + pip`** only when uv isn't available on the project or if uv breaks something
- **I'll use `pyenv + uv`** when I need a specific Python version that's not my system default
- **I'll use `conda`** when I need system-level libraries like CUDA, scientific computing packages, or cross-platform consistency
- **I'll use `poetry`** when I want integrated project management with dependency resolution or building a library
