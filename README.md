# Template Registry for `scaffold-repo`

The `scaffold-repo` CLI is a purely declarative execution engine. It contains no hardcoded opinions about your build system, your CI/CD pipeline, or your folder structures. All of that power is defined entirely within this **Template Registry**.

This guide explains how this repository is structured, how to write your own templates, configure language stacks, harness the engine's dynamic routing logic, and manage organizational compliance using profiles.

---

## 1. The Registry Directory Structure (Separation of Concerns)

A Template Registry is a single repository containing YAML configurations, Jinja2 templates, and licensing rules. All logic is contained within the [`templates/`](templates/) directory.

The engine expects this specific layout to cleanly separate organizational data from technical implementation:

```text
templates/
├── .scaffold-defaults.yaml    # Global router: defines global variables, prompts, and packages
├── stacks/                    # The technical implementations (e.g., c/cmake, python/standard)
├── profiles/                  # Organizational standards (author info, default licenses, tools)
├── mixins/                    # Modular, optional Jinja templates (changie, jekyll-site)
├── libraries/                 # The global dependency index (how to build/link external tools)
├── licenses/                  # SPDX logic and NOTICE file generators
├── app-resources/             # Special templates looped per sub-application inside a repo
└── resources/                 # Global assets like aliases.yaml or canonical license texts
````

### The Intuition: Why separate Stacks and Profiles?

If you put author information or license requirements directly into your Python templates, you have to duplicate them for your C++ templates. By separating them, the **Stack** (e.g., [`templates/stacks/python/standard/`](/templates/stacks/python/standard/)) only cares about *how to build Python*, while the **Profile** (e.g., [`templates/profiles/default.yaml`](/templates/profiles/default.yaml)) dictates *who owns the code and how it is licensed*. The engine magically merges them together during execution.

-----

## 2\. Profiles, Inheritance, & Entity Shorthands

To keep your registry DRY (Don't Repeat Yourself), YAML configuration files in this repository can inherit from one another using the `includes` array.

A perfect example is how organizational profiles are structured. Look at how the base [`default`](/templates/profiles/default.yaml) profile defines the primary author and license, while the [`default-knode`](/templates/profiles/default-knode.yaml) profile inherits those baselines and appends its own specific copyrights.

**Base Profile ([`templates/profiles/default.yaml`](/templates/profiles/default.yaml)):**

```yaml
authors:
  - andy_curtis
license_profile: apache-2.0

contributors:
  andy_curtis:
    name: Andy Curtis
    email: contactandyc@gmail.com
    contact: "Andy Curtis <contactandyc@gmail.com>"
    author: "Andy Curtis <contactandyc@gmail.com> [linkedin.com/in/andycurtis](https://linkedin.com/in/andycurtis) [github.com/contactandyc](https://github.com/contactandyc)"
    start_year: 1998

# Notice the dot-notation shorthand below!
copyrights:
  - contributors.andy_curtis
```

**Derived Profile ([`templates/profiles/default-knode.yaml`](/templates/profiles/default-knode.yaml)):**

```yaml
# 1. Inherit the base profile using the relative path from the templates/ directory
includes:
  - profiles/default

# 2. Append new contributors
contributors:
  knode:
    name: Knode.ai
    website: [https://www.knodeai.com/](https://www.knodeai.com/)
    contact: "Knode.ai"
    full_entity: "Knode.ai — [https://www.knodeai.com/](https://www.knodeai.com/)"
    start_year: 2024
    end_year: 2025

# 3. Override the copyrights array to include both parties
copyrights:
  - contributors.andy_curtis
  - contributors.knode
```

### The Entity Reference Shorthand

Instead of duplicating names, emails, and dates across your `copyrights` and `contacts` arrays, the engine features an **Entity Reference Shorthand**.

When the engine sees a dot-notation string like `contributors.andy_curtis` in the `copyrights` or `contacts` arrays, it dynamically traverses your configuration dictionary to find that exact object. It then automatically:

1.  Resolves the properties (like `name` and `full_entity`).
2.  Calculates a dynamic `year_span` (e.g., `1998–2026`) using the entity's `start_year` and the project's current year (or `end_year` if specified).  This will also consider the `date_created` in your project's local scaffold.yaml.
3.  Injects the fully resolved dictionary into the Jinja context so your License templates can iterate over it cleanly (`{% for cp in copyrights %}`).

-----

## 3\. The Data Merge Pipeline (Variable Context)

Because `scaffold-repo` acts as a fleet manager, building the variable context (what the Jinja templates actually see) happens in a deliberate, multi-stage process.

The engine resolves dependencies and layers data in this exact order **(where the last item wins and overwrites conflicts)**:

1.  **Global Base Defaults:** The pre-loaded state from [`templates/.scaffold-defaults.yaml`](/templates/.scaffold-defaults.yaml).
2.  **Stack Defaults:** The YAML from `templates/stacks/<stack>/<stack_type>/.scaffold-defaults.yaml`.
3.  **Profile YAML:** The organizational overrides requested by the user (e.g., `profile: default-knode`).
4.  **License Profile YAML:** The base license terms derived from the profile.
5.  **Local `scaffold.yaml`:** The repository's own local manifest. Any keys defined here permanently overwrite the inherited templates and profiles.

### Auto-Injected Variables & Discovery

The engine automatically calculates, globs, and injects derived variables into your Jinja context to save you from writing complex logic:

* `project_name`: The raw name (e.g., "my-python-app")
* `project_slug`: Hyphenated (e.g., "my-python-app")
* `project_snake`: Snake case (e.g., "my\_python\_app")
* `year`: The current year (or derived from `date_created`).
* **Auto-Discovery:** If the user does not explicitly define `library_sources` or `test_targets` in their `scaffold.yaml`, the engine automatically scans the file system (e.g., `src/**/*.c` and `tests/src/test_*.c`) and populates those arrays for you\!
* **Package Rollups:** Derived lists like `find_packages`, `link_libraries`, `apt_packages`, and `apt_dev_packages` are automatically computed by traversing the dependency graph.

**Important Escape Hatches:**

* **`kind: header_only`**: If set in a user's `scaffold.yaml` (common for C/C++), the engine logic conditionally treats the project as an INTERFACE library, skipping binary compilation.
* **`extra_targets: | ...`**: Used to inject raw text directly into the generated build files without forking the entire template (e.g., defining an `add_custom_target` in CMake).

-----

## 4\. The Global Library Index (Dependencies)

To allow developers to easily link dependencies in their `depends_on` arrays, you can define centralized archetypes in the [`templates/libraries/`](/templates/libraries/) directory.

The engine supports topologically sorting and linking different `kind`s of dependencies:

### System Libraries (`kind: system`)

Used for packages installed via OS package managers (like `apt`). The engine automatically bubbles up the `pkg` names into the `deps.apt_packages` Jinja array for use in things like Dockerfiles or CI scripts.

**Example ([`templates/libraries/system/CURL.yaml`](/templates/libraries/system/CURL.yaml)):**

```yaml
kind: system
pkg: libcurl4-openssl-dev
find_package: "CURL REQUIRED"
link: "CURL::libcurl"
depends_on:
  - system/OpenSSL
  - system/ZLIB
```

### Git / Source Libraries (`kind: git`)

Used for internal monorepo packages or external OSS dependencies that need to be fetched and compiled locally.

**Example ([`templates/libraries/common/libuv.yaml`](/templates/libraries/common/libuv.yaml)):**

```yaml
kind: git
url: [https://github.com/libuv/libuv.git](https://github.com/libuv/libuv.git)
branch: v1.48.0
shallow: true
build_steps:
  - cmake -S . -B build -DCMAKE_INSTALL_PREFIX=/usr/local -DBUILD_TESTING=OFF
  - cmake --build build -j"$(nproc)"
  - sudo cmake --install build
find_package: "libuv CONFIG REQUIRED"
link: "libuv::uv"
```

Once defined in the registry, a user can simply write `- system/CURL` or `- common/libuv` in their project's `depends_on` array, and the engine handles all linking, fetching, and transitive dependencies natively\!

-----

## 5\. Jinja Front-Matter & File Processing

Files ending in `.j2` are rendered as Jinja templates. Files without `.j2` are copied verbatim.

To tell the engine where a file *actually* belongs in the target repository, you use a special Jinja comment block at the very top of the `.j2` file.

**Example ([`templates/stacks/python/standard/src/__main__.py.j2`](/templates/stacks/python/standard/src/__main__.py.j2)):**

```jinja
{#- scaffold-repo: { dest: "src/{{ project_snake }}/__main__.py", updatable: false } -#}
import sys
...
```

### Supported Front-Matter Attributes:

* **`dest` (string):** The absolute output path relative to the target repository root. *Note: Jinja variables like `{{ project_snake }}` are fully evaluated in the path\!*
* **`context` (string):** Shifts the root of the Jinja variables. If `context: "deps"`, variables resolve relative to the `cfg["deps"]` dictionary.
* **`header_managed` (boolean):** If `true`, the engine injects and manages SPDX headers in this file. (Defaults to true automatically for standard code files).
* **`executable` (boolean):** Ensures the resulting file has `+x` execution permissions (like `build.sh`).

### The Lifecycle Flags: `updatable` vs. `on_init`

* **`updatable` (boolean):** Defaults to `true`. If set to `false`, the engine will generate the file, but will skip overwriting it on future `--update` runs if the file already exists on disk. Use this for seed files (like `src/main.c`) where you don't want to overwrite the developer's custom code later.
* **`on_init` (boolean):** Defaults to `false`. If set to `true`, the template is **strictly bound to the `--create` wizard**. During a standard `--update` across the fleet, the engine pretends this template does not even exist. Use this for bootstrap scripts or Day-0 instructional files.

### Strict Mode Warning

The Jinja environment runs in **Strict Mode**. If you try to access `{{ lib.branch }}` and it wasn't defined, the engine will crash. **Always use `.get()` or `| default()` for optional values:**

```jinja
# Bad: Will crash if branch is missing
git clone --branch {{ lib.branch }} 

# Good: Safely handles missing data
git clone {% if lib.get('branch') %}--branch {{ lib.get('branch') }}{% endif %}
```

-----

## 6\. Package Routing & Dynamic Prompts

You don't want every template rendering into every repository. The engine dynamically routes globs of files based on variable combinations defined in [`templates/.scaffold-defaults.yaml`](/templates/.scaffold-defaults.yaml).

```yaml
packages:
  base: true
  stack: true
  site: false

template_packages:
  base:
    - base/**
  stack:
    - stacks/{{ stack }}/base/**
    - stacks/{{ stack }}/{{ stack_type }}/**
  site:
    - mixins/jekyll-site/**
```

If a user configures a project with `stack: python/standard`, the engine seamlessly maps the `packages.stack` boolean to evaluate all templates inside [`templates/stacks/python/standard/`](/templates/stacks/python/standard/). If the user adds `packages: { site: true }` to their local manifest, the [`templates/mixins/jekyll-site/`](/templates/mixins/jekyll-site/) files are instantly included in the build plan.

### Dynamic Directory Prompts

Defaults files also declare interactive prompts to guide users during setup. Instead of hardcoding options, you can tell the engine to dynamically scan directories in your registry to generate prompt choices using `choices_from_dir`:

```yaml
init_prompts:
  - var: stack_type
    prompt: "Select C/C++ build system:"
    choices_from_dir: "stacks/{{ stack }}"
    exclude: ["base"]
```

-----

## 7\. Sub-Applications (`app-resources/`)

C/C++ and Rust repositories often contain a base library *and* several sub-applications (like CLI tools, demos, or examples). `scaffold-repo` handles this natively via the [`templates/app-resources/`](/templates/app-resources/) directory.

Any template placed in `templates/app-resources/<stack>/<stack_type>/` is **exempt from standard root-level processing**. Instead, it is evaluated in a loop for *every* app defined in the local `scaffold.yaml`.

To share routing and dependency logic across multiple apps, the user defines an `apps.context` block in their repository:

```yaml
apps:
  # The context applies to all apps below it
  context:
    dest: examples
    depends_on:
      - [https://github.com/contactandyc/a-map-reduce-library.git](https://github.com/contactandyc/a-map-reduce-library.git)
      
  01_word_count:
    binaries:
      - src/main.c
      
  02_streaming_recommender:
    binaries:
      - src/*.c
```

If a template exists at [`templates/app-resources/c/cmake/CMakeLists.txt.j2`](/templates/app-resources/c/cmake/CMakeLists.txt.j2), the engine automatically routes the template to `examples/01_word_count/CMakeLists.txt` and `examples/02_streaming_recommender/CMakeLists.txt`, injecting the correct localized context (`app_project_name`, dependencies, etc.) into the Jinja payload for each loop iteration.

-----

## 8\. Defining OSS Licenses and Header Injection

Licenses are managed dynamically so you can update copyrights universally across your fleet. A license YAML file (e.g., [`templates/licenses/apache-2.0.yaml`](/templates/licenses/apache-2.0.yaml)) looks like this:

```yaml
spdx: |
  {% for cp in copyrights %}
  SPDX-FileCopyrightText: {{ cp.year_span }} {{ cp.entity }}
  {% endfor %}
  SPDX-License-Identifier: Apache-2.0
license: LICENSE
license_canonical: resources/licenses/Apache-2.0.txt
notice: |
  {{ project_title }}
  {% for cp in copyrights %}
  © {{ cp.year_span }} {{ cp.full_entity }}
  {% endfor %}
  Licensed under Apache-2.0 (see LICENSE)
```

### Automatic Comment Syntax Translation

You only define the `spdx` block **once** in plain text. When `scaffold-repo` runs, it scans file extensions and automatically wraps the rendered SPDX text in the correct comment syntax for that language (e.g., `//` for C/C++, `#` for Python/Bash, `--` for SQL).

### Overrides vs. Extras

Users can target specific files in their repositories using globs in their local `scaffold.yaml`:

* **`license_overrides`:** Completely *replaces* the primary SPDX header for the matched files.
    * *Example:* The `the-lz4-library` overriding `src/impl/**` with a custom inline BSD-2-Clause license.
* **`license_extras`:** *Appends* an additional attribution block below the primary project license.
    * *Example:* The `the-macro-library` keeping the project's Apache-2.0 license, but appending the `dutch-national-flag` notice to `include/.../macro_dutch_flag_partition.h`.

The engine seamlessly applies the headers to the matched files and dynamically merges the required `NOTICE` blocks into the repository's root `NOTICE` file\!
