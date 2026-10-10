---
name: openapi-test-runner
description: Scaffold Cucumber.js step definitions and project files to run the .feature files produced by openapi-owasp-gherkin. Step 5a in the openapi-security-test pipeline.
---

# OpenAPI Test Runner

Scaffold a runnable Cucumber.js test suite from the generated `.feature` files.

## Workflow

1. **Inventory feature files** — list all `.feature` files in `security_test_cases/`.
2. **Read auth setup** — check `openapi-dependencies.md` for the session setup sequence (register → login → save token).
3. **Read test context** — check `openapi-test-context.md` for base URL, register/login paths, and any known credentials or resource IDs to wire into the world.
4. **Read the raw spec** for base URL and endpoint list if context files are absent.
5. **Generate** the following files (see structure below).
6. **Verify** `package.json` exists; if not, create one or update the existing one.

## Output File Structure

```
security_test_cases/
  support/
    world.js          ← shared state (baseUrl, token, last response)
    hooks.js          ← Before/After hooks (register+login before each scenario)
    steps/
      common.steps.js ← shared steps used across all feature files
      auth.steps.js   ← login/register/logout steps
      <domain>.steps.js per resource domain derived from the spec
cucumber.js           ← Cucumber config
package.json          ← add @cucumber/cucumber + axios devDependencies
```

## Step Definition Patterns

### world.js
```js
const { setWorldConstructor, World } = require('@cucumber/cucumber');
const axios = require('axios');

class ApiWorld extends World {
  constructor(options) {
    super(options);
    this.baseUrl = process.env.BASE_URL; // set BASE_URL to servers[0].url from the spec
    if (!this.baseUrl) throw new Error('BASE_URL env var is required — set it to servers[0].url from the spec');
    this.token = null;
    this.response = null;
    this.users = {}; // { username: token }
    // Set these to the actual register/login paths from openapi-dependencies.md
    this.registerPath = process.env.REGISTER_PATH || '/register';
    this.loginPath = process.env.LOGIN_PATH || '/login';
  }

  async request(method, path, { body, token, headers } = {}) {
    const h = { 'Content-Type': 'application/json', ...headers };
    if (token) h['Authorization'] = `Bearer ${token}`;
    this.response = await axios({
      method, url: `${this.baseUrl}${path}`,
      data: body, headers: h, validateStatus: () => true,
    });
    return this.response;
  }
}

setWorldConstructor(ApiWorld);
```

### hooks.js
```js
const { Before, After } = require('@cucumber/cucumber');

Before(async function() {
  const user = `test_${Date.now()}`;
  const pass = 'TestPass1!';
  await this.request('POST', this.registerPath, { body: { username: user, password: pass } });
  const r = await this.request('POST', this.loginPath, { body: { username: user, password: pass } });
  this.token = r.data.token;
  this.currentUser = user;
});

After(async function() {
  // optional cleanup — delete test user if the API exposes a delete endpoint
  this.token = null;
});
```

### common.steps.js — key step patterns
```js
const assert = require('assert');

Given('I am authenticated as {string} with password {string}', async function(user, pass) {
  // Use register and login paths from openapi-dependencies.md
  await this.request('POST', this.registerPath, { body: { username: user, password: pass } });
  const r = await this.request('POST', this.loginPath, { body: { username: user, password: pass } });
  this.users[user] = r.data.token;
  this.token = r.data.token;
});

When('I send a {word} request to {string}', async function(method, path) {
  await this.request(method, path, { token: this.token });
});

When('I send a {word} request to {string} with body:', async function(method, path, body) {
  await this.request(method, path, { body: JSON.parse(body), token: this.token });
});

Then('the response status code should be {int}', function(code) {
  assert.strictEqual(this.response.status, code);
});

Then('the response body should contain {string}', function(key) {
  assert.ok(key in this.response.data, `Key "${key}" not in response`);
});
```

## cucumber.js Config

```js
module.exports = {
  default: {
    paths: ['security_test_cases/**/*.feature'],
    require: ['security_test_cases/support/**/*.js'],
    format: ['progress-bar', 'html:reports/cucumber-report.html', 'json:reports/cucumber-report.json'],
    parallel: 2,
  }
};
```

## package.json additions

Add to `devDependencies`:
```json
"@cucumber/cucumber": "^10.0.0",
"axios": "^1.6.0",
"assert": "^2.1.0"
```

Add to `scripts`:
```json
"test:security": "cucumber-js"
```

Generate one additional `test:<category>` script per `.feature` file found in `security_test_cases/` that bypasses the config and runs a single file directly (e.g. `"test:api1": "cucumber-js security_test_cases/api1_bola_idor.feature --no-profile"`).

## Notes

- One steps file per resource domain — don't put everything in one file.
- Scan all .feature files for unique step phrases before writing definitions; every step must have a match.
- Vulnerability-assertion steps (not HTTP outcomes): implement as pending (`return 'pending'`).
- Next step: `npm install` then `npm run test:security`, then `openapi-finding-reporting`.
