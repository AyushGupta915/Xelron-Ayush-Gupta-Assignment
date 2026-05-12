# Part 2: Pull Request Analysis

## Repository Selected: beetbox/beets

---

## PR 1: [#3877 — Web readonly](https://github.com/beetbox/beets/pull/3877)

### PR Summary

The web plugin for beets had been exposing a REST API which was accepting HTTP DELETE and PATCH requests with no authentication at all. Anyone with access to the server would be able to delete and modify entries in the library. This issue was mentioned in pull request number 3870 as a potential security risk since the web plugin can be used in a local network or even publicly accessible server environment. This PR proposes adding a new parameter called `readonly` for the web plugin configuration. If this parameter is set to true, then the DELETE and PATCH methods will result in errors and are disabled.

### Technical Changes

- **`beetsplug/web.py`** — Added `readonly` config key with a default value of `true`; added request guards at the DELETE and PATCH route handlers to check `app.config['READONLY']` and return an HTTP 405 or 403 error if read-only mode is active; added logic to read the beets config value and push it into the Flask app configuration at startup
- **`docs/plugins/web.rst`** — Added documentation for the new `readonly` configuration option, including its default value and how to disable it
- **`docs/changelog.rst`** — Added changelog entry noting the behavioral change and the migration path for existing users
- **`test/test_ui.py` or equivalent test file** — Added test cases covering DELETE and PATCH requests under both `readonly: true` and `readonly: false` configurations

### Implementation Approach

This configuration involves two configuration namespaces. The first namespace is that of the beets plugin itself, which uses YAML files to store configurations, while the second namespace is the Flask application itself 	(`self.config['readonly'].get(bool)`) and writes it into Flask's `app.config` under the key `READONLY`. The individual route handlers for DELETE (`/item/<id>`) and PATCH (`/item/<id>`) then read from `app.config['READONLY']` before proceeding. In case of a `True` flag, they raise an error straightaway without performing any database change operation. Business logic is thus kept separate from middleware code while the two-stage configuration propagation mechanism helps Flask’s test client inject the same configuration value during tests as well.

### Potential Impact

This change will impact all beets web plugin users, who were using HTTP DELETE/PATCH actions without explicitly setting `readonly: false`. This is a breaking change because it changes the default behavior. The REST API surface that is exposed to the outside world will be smaller by default. This makes things more secure by default in case of accidental public exposure. The read side (GET actions) remains unaffected.

### Design Rationale

This approach was chosen because it introduces minimal disruption to the existing system while solving the problem effectively. By extending configuration (PR 3877) and using a plugin-based mechanism (PR 3883), the changes remain modular and backward-compatible. This avoids invasive modifications to core logic and aligns with beets' extensible architecture.

**Engineering Insight:** This change follows a secure-by-default design principle. Instead of relying on users to configure security, it restricts unsafe operations unless explicitly enabled.
---

## PR 2: [#3883 — Experimental "bare-ASCII" matching query (bareasc plugin)](https://github.com/beetbox/beets/pull/3883)

### PR Summary

When users had music libraries containing characters which weren't from the standard ASCII set, they could not perform queries to search for music items in them as it involved only ASCII inputs. For instance, when a user used "Bjork", it would not give any results for "Björk" since beets matched the queries on an exact string match or substring basis. It came into the notice during issue #3882. In this pull request, there is an introduction of a new plugin "bareasc". In this plugin, when the user searches by including "#" at the beginning, then both the term and the candidate item string get converted to ASCII versions.

### Technical Changes

- **`beetsplug/bareasc.py`** — New file; defines the `BareASCIIQuery` class (subclass of `SubstringQuery` or `StringQuery`) that overrides the match logic to apply `unicodedata.normalize('NFD', ...)` followed by ASCII encoding with `errors='ignore'` to strip diacritics; registers `#` as the query prefix via the plugin's `queries()` method
- **`docs/plugins/bareasc.rst`** — New documentation file describing the plugin, its purpose, and usage examples
- **`docs/plugins/index.rst`** — Added `bareasc` to the plugin list
- **`docs/changelog.rst`** — Added changelog entry for the new plugin
- **`test/test_query.py` or `test/test_bareasc.py`** — Added test cases verifying that `#bjork` matches items with "Björk", that exact ASCII-only strings still match normally, and that the `#` prefix is correctly recognized

### Implementation Approach

The underlying trick uses the `unicodedata` module of Python’s standard libraries. The Normalization Form NFD (Canonical Decomposition) separates compound characters into base characters and individual combining characters. So for example "ö" would turn into "o" and a combining umlaut symbol. Then by encoding it into ASCII with `errors='ignore'`, all the non-ASCII combining characters are ignored and you get the ASCII base characters only. The above process is carried out symmetrically on both the input query string and the tested value field. The plug-in taps into the query processing mechanism in beets through its query interface and registers its own query type which is recognized via the "#" prefix.

### Potential Impact

It is a simple addition that does not affect any current functionality unless the bareasc plug-in is explicitly invoked. After doing so, the `#` symbol becomes a search operator, and any previously defined queries starting with `#` will now behave differently. The modification applies to the search interface layer and does not interfere with metadata storage, importing processes, and file handling mechanisms. It greatly helps users with non-English music libraries while using ASCII keyboards.

### Design Rationale

This approach was chosen because it introduces minimal disruption to the existing system while solving the problem effectively. By extending configuration (PR 3877) and using a plugin-based mechanism (PR 3883), the changes remain modular and backward-compatible. This avoids invasive modifications to core logic and aligns with beets' extensible architecture.

**Engineering Insight:** This demonstrates extensibility through plugins, allowing feature expansion without modifying the core query engine.
