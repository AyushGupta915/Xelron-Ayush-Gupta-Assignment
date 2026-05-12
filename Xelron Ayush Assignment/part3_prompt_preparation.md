# Part 3: Prompt Preparation Document

## Selected PR: beetbox/beets — [#3877: Web readonly](https://github.com/beetbox/beets/pull/3877)

---

## 3.1.1 Repository Context

Beets is a music library manager for Linux, Windows, and Mac OS X that is implemented using Python. Its main functionality lies in managing music libraries by importing audio files, querying information about tracks and albums in MusicBrainz music database, renaming files and moving them according to custom templates, and saving the whole information in the SQLite database.

The main unique feature of beets is that it implements a modular plugin system. The core of the application includes the library model and command line interface, whereas almost all functionalities are provided by plugins. Plugins are located in the `beetsplug/` subdirectory and utilize the event system and special registration functions in order to work. One of such plugins is a web plugin, which launches a simple HTTP server based on Flask framework in order to provide the whole beets library as an HTTP-accessible RESTful service.

The target audience for beets is tech-savvy music fans – individuals who have high concerns regarding metadata and file management and know how to use a command-line tool. The web plugin application would be useful in cases where one wishes to view his or her library using the phone, media player, or customized frontend on the same computer.

The problem space that this project addresses lies in personal media management – issues related to tagging music files, dealing with the chaos that comes with music metadata from multiple sources, and managing a big music collection. In the case of web plugin, the problem space intersects with web security basics – once you expose your music library through HTTP, you have to consider the issue of access control.

---

## 3.1.2 Pull Request Description

Prior to this PR, the beets web plugin accepted DELETE and PATCH operations by any HTTP client capable of reaching the server, resulting in deletion or modification of library items within the database. This was not configurable. Users running the web plugin from a machine accessible to other users on their network, or even from the public Internet, had no configuration option for enforcing readonly access on the plugin.

This is addressed by issue #3870. The PR adds a new configuration value called `readonly` under `web:`. If `readonly=true` (the default value now), then DELETE and PATCH will respond with the proper HTTP error, denying access. If `readonly=false`, then the behavior is unchanged.

Default behavior change: earlier, the write was allowed implicitly. With the PR, the write action will be denied, unless it is specifically requested via `readonly: false`. This is an intentional breakage, approved by the author, as the safer default is the right choice for a plugin dealing with data exchange over the network. The current users, relying on the write action support, should enable it with `readonly: false` in their config. All read actions (GET requests) remain totally unaffected by the parameter value.

The code also shows how one should handle the plumbing to get a value stored in beets's YAML-like config into a Flask application's config dictionary - the former cannot be used directly since route handlers operate on `current_app.config` dict.
---

## 3.1.3 Acceptance Criteria

✓ When `readonly` is not specified in the config, it defaults to `true`, and DELETE requests to `/item/<id>` return a non-2xx HTTP error response.

✓ When `readonly` is not specified in the config, it defaults to `true`, and PATCH requests to `/item/<id>` return a non-2xx HTTP error response.

✓ When `readonly: false` is explicitly set in the `web:` section of the beets config, DELETE requests to `/item/<id>` succeed and remove the item from the library.

✓ When `readonly: false` is explicitly set in the `web:` config, PATCH requests to `/item/<id>` succeed and update the item's fields in the library.

✓ GET requests to `/item/<id>`, `/item/`, and `/album/<id>` succeed regardless of whether `readonly` is `true` or `false`.

✓ The `readonly` configuration option is documented in `docs/plugins/web.rst` with its default value and an example of how to disable it.

✓ A changelog entry is present in `docs/changelog.rst` describing this as a behavior-changing update and noting how existing users can restore previous behavior.

✓ The beets config value `self.config['readonly']` is correctly propagated into Flask's `app.config['READONLY']` at plugin startup time, so that Flask route handlers can access it through `current_app.config`.

---

## 3.1.4 Edge Cases

**Edge Case 1 — Misconfigured config value type:**
The user writes `readonly: yes` or `readonly: 1` instead of `readonly: true` in YAML. The implementation must use beets' config system to coerce this to a boolean (e.g., `self.config['readonly'].get(bool)`), not a raw string comparison. A string `"true"` would be truthy in Python but could silently misbehave if cast incorrectly.

**Edge Case 2 — The Flask test client and config injection:**
In the context of automated tests, however, the `app` instance in Flask is created without being associated with an actual beets configuration file. For tests that require the testing of whether `readonly: false`, there must be a mechanism by which `app.config['READONLY'] = False` is set before the request is made. Otherwise, if the transfer occurs only once during the startup process through some hook, then the tests for the permissive mode would be inaccurate.

**Edge Case 3 — Other HTTP methods (PUT, POST):**
The PR describes DELETE and PATCH endpoints. There could be cases where the initial web plugin accepts PUT and POST operations on some routes. This should be checked, as if the operation is a writing one, it should be under the readonly guard. Else, we will have partial protection, which could make users feel safe while they aren’t.

**Edge Case 4 — Error response format:**
For a DELETE/PATCH request that fails because of the read-only state, both the body and the status code need to match other API calls. The client will break if a plain text body is returned for an API that otherwise serves JSON data or if a vague status code such as 200 with an error message is used or well-defined status (405 Method Not Allowed or 403 Forbidden) with a JSON body is preferable.

---

## 3.1.5 Initial Prompt

You are implementing a pull request on the `beetbox/beets` repository. Beets is a Python music library manager with a plugin architecture. Plugins live in `beetsplug/`. The web plugin (`beetsplug/web.py`) runs a Flask HTTP server that exposes the library as a REST API.

**Your task** is to implement the changes from [PR #3877](https://github.com/beetbox/beets/pull/3877), which adds a `readonly` configuration option to the web plugin.

### What needs to be done

1. **Add the config option.** In the web plugin's `WebPlugin` class (inside `beetsplug/web.py`), locate where the plugin's configuration schema is defined (look for `config_defaults` or a `configspec` assignment). Add a new key `readonly` with a default value of `True`.

2. **Propagate config to Flask.** In the method that starts the Flask app (likely `commands()` or a startup hook), after creating the Flask `app` object, read the beets config value and push it into Flask's app config:
   ```python
   app.config['READONLY'] = self.config['readonly'].get(bool)
   ```

3. **Guard write endpoints.** In the Flask route handlers for DELETE and PATCH operations, add a check at the top of each handler:
   ```python
   if flask.current_app.config.get('READONLY', True):
       return flask.make_response(
           flask.jsonify({'error': 'read-only mode'}), 405
       )
   ```
   Apply this guard to every handler that performs a write operation (DELETE item, PATCH item — review carefully whether any PUT or POST routes also write to the database and guard those too).

4. **Write tests.** Add test cases that:
   - Confirm DELETE returns a non-2xx response when `app.config['READONLY'] = True`
   - Confirm DELETE succeeds (2xx) when `app.config['READONLY'] = False`
   - Confirm PATCH returns a non-2xx response when `app.config['READONLY'] = True`
   - Confirm PATCH succeeds when `app.config['READONLY'] = False`
   - Confirm GET requests return 200 in both configurations

5. **Update documentation.** In `docs/plugins/web.rst`, add documentation for the `readonly` option including: its purpose, its default value (`yes`), and an example config block showing how to set `readonly: no`.

6. **Add a changelog entry.** In `docs/changelog.rst`, add a note near the top under the current version describing this as a default-behavior change affecting DELETE and PATCH endpoints, and how users can opt back in.

### Acceptance criteria to verify

- Default behavior blocks DELETE and PATCH; GET still works
- Explicit `readonly: false` restores write access
- Flask config and beets config are correctly linked
- All write-route guards are consistent and return a well-formed JSON error with an appropriate HTTP status code

### Edge cases to handle

- Ensure the config value is parsed as a boolean, not a raw string
- Consider whether any other write-capable HTTP methods beyond DELETE and PATCH exist in the plugin
- Make sure the test setup allows injecting `app.config['READONLY']` directly so tests don't depend on reading an actual config file from disk

### Design Rationale

The chosen approach uses a configuration flag instead of implementing authentication because the web plugin is designed to remain lightweight and easy to use. Introducing authentication would significantly increase complexity and require user management. By enforcing a read-only mode by default, the system follows a secure-by-default principle while preserving flexibility for advanced users.

Additionally, this approach aligns with beets' plugin philosophy by making minimal, non-invasive changes without altering core architecture.

### Testing Strategy

- Use Flask test client to simulate HTTP requests
- Test both configurations (`READONLY = True` and `False`)
- Validate both status codes and JSON response structure
- Ensure no database mutation occurs in readonly mode
- Run full test suite to ensure no regression in unrelated functionality	

**Engineering Insight:** This change enforces a secure-by-default principle, reducing risk exposure without requiring user intervention, which is a common best practice in API design.
