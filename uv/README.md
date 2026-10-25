# uv
An extremely fast Python package and project manager, written in Rust.

- 10-100x faster than pip. 
- Provides comprehensive project management, with a universal lockfile.

## Why uv is so fast?
uv is significantly faster than pip—often by an order of magnitude—because it was built from the ground up in Rust using modern packaging standards, rather than carrying 15 years of legacy compatibility code.

### Key Architectural & Engineering Reasons

- **Written in a Compiled Language (Rust):** Unlike pip, which must spin up the Python interpreter, load modules, and parse files (taking ~300ms just to start), uv is a single compiled binary that executes tasks immediately without needing a Python runtime.
- **Global Caching & Hard Links:** pip copies entire package files into each project directory. uv maintains a global cache and uses hard links (pointers) to share files across projects on disk without duplicating data or wasting space.
- **Parallel Downloads & Concurrency:** pip downloads packages sequentially (one at a time). uv uses asynchronous I/O and multi-threading (via Tokio and Rayon) to download and process dozens of packages in parallel.
- **Smart HTTP Range Requests:** When inspecting package metadata inside wheel (zip) archives, pip often downloads the entire archive. uv uses HTTP range requests or PEP 658 metadata to fetch only the exact bytes it needs (around 100 KB instead of megabytes).
- **Advanced Dependency Resolution (CDCL):** pip uses a basic backtracking approach that tests version combinations one by one. uv uses a Conflict-Driven Clause Learning (CDCL) algorithm (similar to chip design software) to learn from conflicts and skip millions of invalid combinations instantly.

## Installation - Mac using brew
`
brew install uv
`

## Project structure 
Link - https://docs.astral.sh/uv/guides/projects/

### uv init \<project>
```ZSH
uv init hello-world
cd hello-world
```
uv will create the following files and directories:

```
├── .git/
├── .gitignore
├── .python-version
├── pyproject.toml
├── README.md
└── src
    └── hello_world
        └── __init__.py
```

### uv run/uv sync
uv will create a virtual environment `.venv` and `uv.lock` file in the root of your project the first time you run a project command, i.e., `uv run`, `uv sync`, or `uv lock`.
<br/><br/>
A complete listing would look like:
```
.
├── .git/
├── .venv/
│   ├── bin
│   ├── lib
│   └── pyvenv.cfg
├── .gitignore
├── .python-version
├── README.md
├── src
│   └── hello_world
│       └── __init__.py
├── pyproject.toml
└── uv.lock
```

### pyproject.toml
pyproject.toml is a configuration file used by packaging tools like `uv`, as well as other tools such as linters, type checkers, etc. There are three possible TOML tables in this file.
<br/>
- The `[build-system]` table is strongly recommended. It allows you to declare which build backend you use and which other dependencies are needed to build your project.
- The `[project]` table is the format that most build backends use to specify your project’s basic metadata, such as the dependencies, your name, etc.
- The `[tool]` table has tool-specific subtables, e.g., [tool.hatch], [tool.black], [tool.mypy].

**A full example -**
```TOML
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "spam-eggs"
version = "2020.0.0"
dependencies = [
  "httpx",
  "gidgethub[httpx]>4.0.0",
  "django>2.1; os_name != 'nt'",
  "django>2.0; os_name == 'nt'",
]
requires-python = ">=3.8"
authors = [
  {name = "Pradyun Gedam", email = "pradyun@example.com"},
  {name = "Tzu-Ping Chung", email = "tzu-ping@example.com"},
  {name = "Another person"},
  {email = "different.person@example.com"},
]
maintainers = [
  {name = "Brett Cannon", email = "brett@example.com"}
]
description = "Lovely Spam! Wonderful Spam!"
readme = "README.rst"
license = "MIT"
license-files = ["LICEN[CS]E.*"]
keywords = ["egg", "bacon", "sausage", "tomatoes", "Lobster Thermidor"]
classifiers = [
  "Development Status :: 4 - Beta",
  "Programming Language :: Python"
]

[project.optional-dependencies]
gui = ["PyQt5"]
cli = [
  "rich",
  "click",
]
all = ["spam-eggs[gui, cli]"]

[project.urls]
Homepage = "https://example.com"
Documentation = "https://readthedocs.org"
Repository = "https://github.com/me/spam.git"
"Bug Tracker" = "https://github.com/me/spam/issues"
Changelog = "https://github.com/me/spam/blob/master/CHANGELOG.md"

[project.scripts]
spam-cli = "spam:main_cli"

[project.gui-scripts]
spam-gui = "spam:main_gui"

[project.entry-points."spam.magical"]
tomatoes = "spam:main_tomatoes"
```


### .python-version
The .python-version file contains the project's default Python version. This file tells uv which Python version to use when creating the project's virtual environment. 
<br/><br/>
**Note** - This can be different form python version mentioned in **pyproject.toml**. Eg. .python-verison can contain 3.15 and pyproject.toml can contain **requires-python = ">=3.14"**. The project will use exact python version mentioned in the **.python-version** file. Even if this version is not installed `uv` will install this version on `uv run` command. 

### .venv
The .venv folder contains your project's virtual environment, a Python environment that is isolated from the rest of your system. This is where uv will install your project's dependencies.
<br/>
- To run a command in the project environment, use `uv run`. Alternatively the project environment can be activated as normal for a virtual environment `source .venv/bin/activate`.
- When `uv run` is invoked, it will create the project environment if it does not exist yet or ensure it is up-to-date if it exists. The project environment can also be explicitly created with `uv sync`.
- For project dependencies, use `uv add` to add a package to the environment.


### uv add
To add a dependency:
```BASH
uv add httpx
```

### uv.lock
uv.lock is a cross-platform lockfile that contains exact information about your project's dependencies. Unlike the pyproject.toml which is used to specify the broad requirements of your project, the **lockfile contains the exact resolved versions of depenedencies that are installed in the project environment**. This file should be checked into version control, allowing for consistent and reproducible installations across machines.
<br/>

uv.lock is a human-readable TOML file but is managed by uv and **should not be edited manually**.
<br/>

It contains the following information for each dependencies - 
- name: "httpx"
- version: "4.15.1"
- source: { registry = "https://pypi.org/simple" }
- dependencies (`httpx`'s own dependencies): [
    { name = "anyio" },
    { name = "certifi" },
    { name = "httpcore" },
    { name = "idna" },
]
- source distribution (sdist): httpx-0.28.1.tar.gz
- binary distribution (wheels): httpx-0.28.1-py3-none-any.whl
<br/>

**Before `uv add` -**
<br/>

pyproject.toml
```TOML
[project]
name = "hello-world"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.14"
dependencies = []
```
uv.lock
```TOML
version = 1
revision = 3
requires-python = ">=3.14"

[[package]]
name = "hello-world"
version = "0.1.0"
source = { virtual = "." }
```

<br/>

**After `uv add` -**

shell - 
``` 
hello-world % uv add httpx    
Resolved 8 packages in 7ms
Installed 6 packages in 4ms
 + anyio==4.15.1
 + certifi==2026.10.23
 + h11==0.16.0
 + httpcore==1.0.9
 + httpx==0.28.1
 + idna==3.20
```
<br/>

pyproject.toml
```TOML
[project]
name = "hello-world"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.14"
dependencies = [
    "httpx>=0.28.1",
]
```

uv.lock
```TOML
version = 1
revision = 3
requires-python = ">=3.14"

[[package]]
name = "anyio"
version = "4.15.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "idna" },
    { name = "typing-extensions", marker = "python_full_version < '3.15'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/a9/d2/f4d173e22df740bc37b1db102b386ba719b66e95b0f0d751f556b387e6d2/anyio-4.15.1.tar.gz", hash = "sha256:9f28306018cbd6d329e64a36d58256edff76dd996fe423bc957326e578b82a94", size = 276966, upload-time = "2026-09-05T10:42:39.44Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/12/b8/4bd346e22b28902df4d651910f5242c28d84e4a5c2435ca5c3f797ed7e2e/anyio-4.15.1-py3-none-any.whl", hash = "sha256:6152fdbbf9a77fdec97731721bebf7c4c44f7c29b424b0065826173efc7ed101", size = 132079, upload-time = "2026-09-05T10:42:37.923Z" },
]

[[package]]
name = "certifi"
version = "2026.10.23"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/fd/11/b524cbd9bbdaca58608e4cf4053fd463417549e1959fa8d3aa18319f9555/certifi-2026.10.23.tar.gz", hash = "sha256:a9895e6e2758b371e0545e321a3a8fd1fcfd9829ada8824a936dfb16aaaf36cd", size = 136225, upload-time = "2026-10-23T11:36:16.478Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/32/d1/a3652fe2c19134d0d15922177821d863cbd7e842d53a21bbe7e8e3a1b5ac/certifi-2026.10.23-py3-none-any.whl", hash = "sha256:e9b54de76e530bb6cf0ccd036787103acb58fd42ba930ea8bf948ddbaa2f61d1", size = 135082, upload-time = "2026-10-23T11:36:14.763Z" },
]

[[package]]
name = "h11"
version = "0.16.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/01/ee/02a2c011bdab74c6fb3c75474d40b3052059d95df7e73351460c8588d963/h11-0.16.0.tar.gz", hash = "sha256:4e35b956cf45792e4caa5885e69fba00bdbc6ffafbfa020300e549b208ee5ff1", size = 101250, upload-time = "2025-04-24T03:35:25.427Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/04/4b/29cac41a4d98d144bf5f6d33995617b185d14b22401f75ca86f384e87ff1/h11-0.16.0-py3-none-any.whl", hash = "sha256:63cf8bbe7522de3bf65932fda1d9c2772064ffb3dae62d55932da54b31cb6c86", size = 37515, upload-time = "2025-04-24T03:35:24.344Z" },
]

[[package]]
name = "hello-world"
version = "0.1.0"
source = { virtual = "." }
dependencies = [
    { name = "httpx" },
]

[package.metadata]
requires-dist = [{ name = "httpx", specifier = ">=0.28.1" }]

[[package]]
name = "httpcore"
version = "1.0.9"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "certifi" },
    { name = "h11" },
]
sdist = { url = "https://files.pythonhosted.org/packages/06/94/82699a10bca87a5556c9c59b5963f2d039dbd239f25bc2a63907a05a14cb/httpcore-1.0.9.tar.gz", hash = "sha256:6e34463af53fd2ab5d807f399a9b45ea31c3dfa2276f15a2c3f00afff6e176e8", size = 85484, upload-time = "2025-04-24T22:06:22.219Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/7e/f5/f66802a942d491edb555dd61e3a9961140fd64c90bce1eafd741609d334d/httpcore-1.0.9-py3-none-any.whl", hash = "sha256:2d400746a40668fc9dec9810239072b40b4484b640a8c38fd654a024c7a1bf55", size = 78784, upload-time = "2025-04-24T22:06:20.566Z" },
]

[[package]]
name = "httpx"
version = "0.28.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "anyio" },
    { name = "certifi" },
    { name = "httpcore" },
    { name = "idna" },
]
sdist = { url = "https://files.pythonhosted.org/packages/b1/df/48c586a5fe32a0f01324ee087459e112ebb7224f646c0b5023f5e79e9956/httpx-0.28.1.tar.gz", hash = "sha256:75e98c5f16b0f35b567856f597f06ff2270a374470a5c2392242528e3e3e42fc", size = 141406, upload-time = "2024-12-06T15:37:23.222Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/2a/39/e50c7c3a983047577ee07d2a9e53faf5a69493943ec3f6a384bdc792deb2/httpx-0.28.1-py3-none-any.whl", hash = "sha256:d909fcccc110f8c7faf814ca82a9a4d816bc5a6dbfea25d6591d6985b8ba59ad", size = 73517, upload-time = "2024-12-06T15:37:21.509Z" },
]

[[package]]
name = "idna"
version = "3.20"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/f5/08/8eea9d4b8302028f3abb2c0813953f7aec26d33b7a8960ed760e65ff29fa/idna-3.20.tar.gz", hash = "sha256:a7db850025b95ded1eae8a46181a1a6c56c92c96f0e2b005d9ff8dc0210cab44", size = 216463, upload-time = "2026-09-17T14:11:04.752Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/58/a2/bb081bab032533a855d44de1d56f8e8426114ff1ba5d1f07a438a0a654f8/idna-3.20-py3-none-any.whl", hash = "sha256:ab7ae7122974553370f0bdb919e1a960b2cd1bc1ef0276416d896db81c14582c", size = 69583, upload-time = "2026-09-17T14:11:03.168Z" },
]

[[package]]
name = "typing-extensions"
version = "4.16.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/f6/cc/6253133b5bb138fc3306cebfbda2c520f545d36b5be2c7255cc528bb45d6/typing_extensions-4.16.0.tar.gz", hash = "sha256:dc983d19a509c94dba722ee6abd33940f7c05a89e243c47e907eb4db6f1a43e5", size = 113555, upload-time = "2026-07-02T08:40:05.92Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/49/d3/b8441a820a491ddfc024b0b0cf0393375b75ea13866d9c66727e54c2fc80/typing_extensions-4.16.0-py3-none-any.whl", hash = "sha256:481caa481374e813c1b176ada14e97f1f67a4539ce9cfeb3f350d78d6370c2e8", size = 45571, upload-time = "2026-07-02T08:40:04.659Z" },
]
```

<br/>

**Note -**
Our own project package was also created as dependency in `uv.lock` file - 
```TOML
[[package]]
name = "hello-world"
version = "0.1.0"
source = { virtual = "." }
dependencies = [
    { name = "httpx" },
]

[package.metadata]
requires-dist = [{ name = "httpx", specifier = ">=0.28.1" }]
```