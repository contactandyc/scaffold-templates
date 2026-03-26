# Template Registry for `scaffold-repo`

The `scaffold-repo` CLI is a purely declarative execution engine. It contains no hardcoded opinions about your build system, your CI/CD pipeline, or your folder structures. All of that power is defined entirely within this **Template Registry**.

This guide explains how this repository is structured, how to write your own templates, configure language stacks, harness the engine's dynamic routing logic, and manage organizational compliance.

---

## 1. How It All Fits Together (The Local Manifest)

Before diving into how to *write* templates, it is crucial to understand how a repository actually *consumes* them. 

When you run `scaffold-repo`, it looks at the target project's local `scaffold.yaml` file. This file acts as the bridge between the developer's specific code and this central Template Registry. 

Here is a real-world example we will use throughout this guide:

```yaml
project_title: A Map Reduce Library
version: 0.0.3
date_created: 2025-08-01

# 1. The Stack: Defines the technical implementation
stack: c/cmake

# 2. The Profile: Defines ownership and default licenses
profile: default-knode

# 3. Base Templates: Tells the engine where to fetch this exact registry
base_templates:
  repo: [https://github.com/contactandyc/scaffold-templates.git](https://github.com/contactandyc/scaffold-templates.git)
  ref: main

# 4. Dependencies: Links to external repos and centralized archetypes
depends_on:
  - [https://github.com/contactandyc/a-memory-library.git](https://github.com/contactandyc/a-memory-library.git)
  - system/ZLIB

# 5. Apps: Routes sub-components and example binaries
apps:
  context:
    dest: examples
  01_word_count:
    binaries:
      - src/main.c

# 6. Escape Hatches: Injects raw code directly into the rendered build files
extra_targets: |
  add_custom_target(indent
      COMMAND python3 ${CMAKE_SOURCE_DIR}/bin/indent-macros ${CMAKE_SOURCE_DIR}/include/*/*.h
      COMMENT "Indenting files..."
  )
````

### The Magic: Auto-Discovery

Notice what is *missing* from the file above? There is no explicit list of source files or test targets\!

If a developer leaves `library_sources` or `tests` out of their `scaffold.yaml`, the engine automatically globs the filesystem (e.g., scanning `src/**/*.c` and `tests/src/test_*.c`) and populates those arrays in the background. The engine injects this discovered data directly into the Jinja context, keeping the developer's manifest incredibly clean.

-----

## 2\. The Registry Directory Structure

To fulfill the manifest above, the engine expects this specific layout within the [`templates/`](/templates/) directory to cleanly separate organizational data from technical implementation:

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
```

### The Intuition: Why separate Stacks and Profiles?

If you put author information or license requirements directly into your Python templates, you have to duplicate them for your C++ templates. By separating them, the **Stack** (e.g., [`templates/stacks/python/standard/`](/templates/stacks/python/standard/)) only cares about *how to build Python*, while the **Profile** (e.g., [`templates/profiles/default.yaml`](/templates/profiles/default.yaml)) dictates *who owns the code and how it is licensed*. The engine magically merges them together during execution.

-----

## 3\. Resolving `stack:` (The Technical Foundation)

In our manifest, the user declared:

```yaml
stack: c/cmake
```

When the engine sees this, it navigates to [`templates/stacks/c/cmake/`](/templates/stacks/c/cmake/). Everything inside this folder represents the *technical implementation* of the project.

The engine will pull the [`CMakeLists.txt.j2`](/templates/stacks/c/cmake/CMakeLists.txt.j2), the [`build.sh.j2`](/templates/stacks/c/cmake/build.sh.j2), and the [`Dockerfile.j2`](/templates/stacks/c/cmake/Dockerfile.j2) from this directory. It will also load the local [`templates/stacks/c/cmake/.scaffold-defaults.yaml`](/templates/stacks/c/cmake/.scaffold-defaults.yaml) to establish baseline variables specifically for C++ (like `c_standard: 23` and `cmake_min_version: "3.20"`).

-----

## 4\. Resolving `profile:` (Ownership & Compliance)

In our manifest, the user declared:

```yaml
profile: default-knode
```

While the Stack cares about *how to build the code*, the Profile dictates *who owns it and how it is licensed*. The engine navigates to [`templates/profiles/default-knode.yaml`](/templates/profiles/default-knode.yaml).

To keep your registry DRY (Don't Repeat Yourself), profiles can inherit from one another using the `includes` array:

**Base Profile ([`templates/profiles/default.yaml`](/templates/profiles/default.yaml)):**

```yaml
authors:
  - andy_curtis
license_profile: apache-2.0

contributors:
  andy_curtis:
    name: Andy Curtis
    start_year: 1998
    # ... email, contact strings, etc.

# Notice the dot-notation shorthand below!
copyrights:
  - contributors.andy_curtis
```

**Derived Profile ([`templates/profiles/default-knode.yaml`](/templates/profiles/default-knode.yaml)):**

```yaml
# Inherit the base profile 
includes:
  - profiles/default

# Append new contributors
contributors:
  knode:
    name: Knode.ai
    start_year: 2024
    end_year: 2025

# Override the copyrights array to include both parties
copyrights:
  - contributors.andy_curtis
  - contributors.knode
```

### The Entity Reference Shorthand

When the engine sees a dot-notation string like `contributors.andy_curtis` in the `copyrights` array, it dynamically traverses the configuration dictionary to find that exact object. It calculates a dynamic `year_span` (e.g., `1998–2026`) using the entity's `start_year` and the project's current year, and injects the fully resolved dictionary into the Jinja context so your License templates can iterate over it cleanly.

-----

## 5\. The Data Merge Pipeline (Variable Context)

Because `scaffold-repo` acts as a fleet manager, building the final variable context happens in a deliberate, multi-stage pipeline. The engine resolves dependencies and layers data in this exact order **(where the last item wins and overwrites conflicts)**:

1.  **Global Base Defaults:** The pre-loaded state from [`templates/.scaffold-defaults.yaml`](/templates/.scaffold-defaults.yaml).
2.  **Stack Defaults:** Found via `stack: c/cmake` in [`templates/stacks/c/cmake/.scaffold-defaults.yaml`](/templates/stacks/c/cmake/.scaffold-defaults.yaml).
3.  **Profile YAML:** Found via `profile: default-knode` in [`templates/profiles/default-knode.yaml`](/templates/profiles/default-knode.yaml).
4.  **License Profile YAML:** The base license terms derived from the profile.
5.  **Local `scaffold.yaml`:** The repository's own local manifest permanently overwrites everything above it.

### Deriving Variables

Based on `project_title: A Map Reduce Library` in our manifest, the engine automatically calculates and injects derived variables into your Jinja context:

* `project_name`: "A Map Reduce Library"
* `project_slug`: "a-map-reduce-library"
* `project_snake`: "a\_map\_reduce\_library"
* `year`: Extracted from `date_created: 2025-08-01`.

-----

## 6\. Resolving `depends_on:` (The Global Library Index)

In our manifest, the user declared:

```yaml
depends_on:
  - [https://github.com/contactandyc/a-memory-library.git](https://github.com/contactandyc/a-memory-library.git)
  - system/ZLIB
```

The engine natively understands pure Git URLs. It will clone `a-memory-library`, read its `scaffold.yaml`, and automatically inject its `find_packages` and `link_libraries` requirements into the `deps` Jinja array for `A Map Reduce Library` to consume.

But what about `system/ZLIB`? To allow developers to easily link external or system-level dependencies, you define centralized archetypes in the [`templates/libraries/`](/templates/libraries/) directory.

The engine looks up [`templates/libraries/system/ZLIB.yaml`](/templates/libraries/system/ZLIB.yaml):

```yaml
kind: system
find_package: "ZLIB REQUIRED"
link: "ZLIB::ZLIB"
pkg: zlib1g-dev
```

The engine seamlessly merges these requirements. It adds `ZLIB REQUIRED` to the `deps.find_packages` array (for the [`CMakeLists.txt.j2`](/templates/stacks/c/cmake/CMakeLists.txt.j2)), and it bubbles `zlib1g-dev` into the `deps.apt_packages` array (for the [`Dockerfile.j2`](/templates/stacks/c/cmake/Dockerfile.j2))\!

-----

## 7\. Jinja Front-Matter & File Processing

Now that the context is fully built, the engine renders the files. Files ending in `.j2` are rendered as Jinja templates. Files without `.j2` are copied verbatim.

To tell the engine where a file *actually* belongs in the target repository, you use a special Jinja comment block at the very top of the `.j2` file.

**Example ([`templates/stacks/c/cmake/src/lib.c.j2`](/templates/stacks/c/cmake/src/lib.c.j2)):**

```jinja
{#- scaffold-repo: { dest: "src/{{ project_snake }}.c", updatable: false, on_init: true } -#}
#include "{{ project_slug }}/{{ project_snake }}.h"
#include <stdio.h>
...
```

*(Notice how the engine uses the derived `project_snake` variable right inside the `dest` path\!)*

### The Lifecycle Flags: `updatable` vs. `on_init`

* **`updatable` (boolean):** Defaults to `true`. If set to `false`, the engine generates the file but skips overwriting it on future `--update` runs. This is used for the `lib.c` seed file above, ensuring the engine never overwrites the developer's actual C code\!
* **`on_init` (boolean):** Defaults to `false`. If set to `true`, the template is **strictly bound to the `--create` wizard**. During a standard `--update` across the fleet, the engine pretends this template does not even exist.

-----

## 8\. Resolving `apps:` (Sub-Applications)

In our manifest, the user declared:

```yaml
apps:
  context:
    dest: examples
  01_word_count:
    binaries:
      - src/main.c
```

C/C++ and Rust repositories often contain a base library *and* several sub-applications (like CLI tools or examples). `scaffold-repo` handles this via the [`templates/app-resources/`](/templates/app-resources/) directory.

Any template placed in [`templates/app-resources/<stack>/<stack_type>/`](/templates/app-resources/c/cmake/) is **exempt from standard root-level processing**. Instead, it is evaluated in a loop for *every* app defined in the local `scaffold.yaml`.

Because the user defined a shared `context.dest: examples`, the engine automatically routes the template from [`templates/app-resources/c/cmake/CMakeLists.txt.j2`](/templates/app-resources/c/cmake/CMakeLists.txt.j2) into `examples/01_word_count/CMakeLists.txt`, automatically injecting the correct localized context (`app_project_name = "a_map_reduce_library_01_word_count"`) into the Jinja payload for that iteration.

-----

## 9\. Defining OSS Licenses and Header Injection

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
    * *Example:* A user overriding `src/third_party/**` with a custom inline BSD-2-Clause license.
* **`license_extras`:** *Appends* an additional attribution block below the primary project license.
    * *Example:* Keeping the project's Apache-2.0 license, but appending the `dutch-national-flag` notice to `include/.../macro_dutch_flag_partition.h`.

The engine seamlessly applies the headers to the matched files and dynamically merges the required `NOTICE` blocks into the repository's root `NOTICE` file\!
