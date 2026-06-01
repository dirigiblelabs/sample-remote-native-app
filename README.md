# sample-remote-native-app

A minimal **Eclipse Dirigible** sample project demonstrating the `remote`
variant of the **native applications** feature
([eclipse-dirigible/dirigible#5949](https://github.com/eclipse-dirigible/dirigible/pull/5949)).
There is no application code in this repository — only the artefacts that tell
the Dirigible platform to reverse-proxy the public
[`httpbin.org`](https://httpbin.org) service under
`/services/native-apps-proxy/v1/http-bin/`, gated by a Dirigible role.

The sister project
[`sample-library-local-native-app`](https://github.com/dirigiblelabs/sample-library-local-native-app)
covers the `local` variant (a Node.js process Dirigible spawns and supervises).

---

## What is declared

| File                                | Purpose                                                                  |
| ----------------------------------- | ------------------------------------------------------------------------ |
| `project.json`                      | Identifies the project to Dirigible (`guid: sample-remote-native-app`).  |
| `roles.roles`                       | Declares the `http-bin` role (synchronized into `DIRIGIBLE_SECURITY_ROLES`). |
| `sample-remote-native-app.native-app` | The native-app artefact — kind `remote`, upstream `https://httpbin.org`, basePath `http-bin`, root exposed under the `http-bin` role. |

### The native-app artefact

```jsonc
{
  "name": "sample-remote-native-app",
  "basePath": "http-bin",
  "type": "remote",
  "config": {
    "url": "https://httpbin.org",
    "security": {
      "authentication": null,
      "exposedPaths": [
        { "path": "/", "scopes": ["http-bin"] }
      ]
    }
  }
}
```

- **`type: "remote"`** — Dirigible doesn't own the process; it just proxies.
- **`config.url`** — every request under the basePath is forwarded here.
- **`security.authentication: null`** — `httpbin.org` is anonymous; no outbound
  `Authorization` header is injected (this is the contrast with the basic-auth
  example in the sister `local` sample).
- **`security.exposedPaths: [{ "path": "/", "scopes": ["http-bin"] }]`** —
  `/` is a prefix that matches *everything* under the upstream, so the whole
  `httpbin.org` API is reachable. Each request additionally requires the
  caller to hold the **`http-bin`** Dirigible role. **Note:** native-app scope
  semantics are intentionally strict — the platform's super-roles
  `DEVELOPER` / `ADMINISTRATOR` do **not** grant implicit access. The role
  must be assigned explicitly.

### The role

```jsonc
[
  {
    "name": "http-bin",
    "description": "Permission to call the httpbin.org upstream exposed by sample-remote-native-app through the Dirigible native-app proxy at /services/native-apps-proxy/v1/http-bin/."
  }
]
```

---

## Try it

1. Clone this repository into a running Dirigible instance:
   - **Git perspective** → *Clone* →
     `https://github.com/dirigiblelabs/sample-remote-native-app.git`.
2. **Publish all** so the synchronizers pick the artefacts up.
3. Assign the **`http-bin`** role to the user that will call the proxy
   (**Security perspective** → *Users* → select user → *Assign Roles*).
4. Call any `httpbin.org` endpoint through Dirigible. Examples:

```
GET    http://localhost:8080/services/native-apps-proxy/v1/http-bin/get
POST   http://localhost:8080/services/native-apps-proxy/v1/http-bin/post
PUT    http://localhost:8080/services/native-apps-proxy/v1/http-bin/put
PATCH  http://localhost:8080/services/native-apps-proxy/v1/http-bin/patch
DELETE http://localhost:8080/services/native-apps-proxy/v1/http-bin/delete
GET    http://localhost:8080/services/native-apps-proxy/v1/http-bin/headers
GET    http://localhost:8080/services/native-apps-proxy/v1/http-bin/status/418
```

Each call is authenticated against Dirigible first; the platform then strips
the `/services/native-apps-proxy/v1/http-bin` prefix and forwards what remains
to `https://httpbin.org`. Without the `http-bin` role the proxy answers with
**403 Forbidden**.

---

## How removal works

Deleting any of the three files and republishing the project tells the
synchronizers to clean up:

- Removing `sample-remote-native-app.native-app` — the artefact row is deleted
  and the basePath is unregistered. Requests to the proxy return **404**.
- Removing `roles.roles` — the `http-bin` role row is deleted; user
  assignments referencing it are cascade-dropped via the database FK, so the
  role disappears from the assignment list of every user that held it.

---

## License

[Eclipse Public License 2.0](LICENSE).
