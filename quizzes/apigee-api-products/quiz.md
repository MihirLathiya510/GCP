# Quiz — Apigee: Publishing APIs as Products

**Score: 100%**

---

**Q1. Which of the following statements are benefits of using a VerifyAPIKey policy in an API proxy? (Select two)**

- The VerifyAPIKey policy enforces the rule that an API key should be stored in a header.
- ✅ **Any custom attributes associated with the developer, app, and API products will be populated as variables and can be used to control the behavior of the API.** ← CORRECT
- The caller is forced to present the consumer key and consumer secret to gain access to the API.
- API requests for a specific app will be automatically rate-limited.
- ✅ **Only apps that have been registered to use the API will be allowed access.** ← CORRECT

> The key ref can point to any flow variable (header, query param, etc.) — not limited to headers.
> VerifyAPIKey checks only the consumer key, not the secret (OAuth uses both).
> Rate limiting requires a separate Quota or SpikeArrest policy.

---

**Q2. Which type of developer is configured in the Publish section of the Apigee console?**

- Portal developer
- API developer
- Backend developer
- ✅ **App developer** ← CORRECT

> The Publish section manages external developers who register apps to consume your APIs.

---

**Q3. Which of the following statements about API products are true? (Select two)**

- APIs bundled in one API product cannot be bundled in another API product.
- API products are APIs that are sold on the open market.
- ✅ **API products may be used to control access or service levels for APIs.** ← CORRECT
- ✅ **API products should be designed based on the needs of app developers.** ← CORRECT
- Apps should only be associated with a single API product.

> The same API operation can appear in multiple products.
> Apps can be associated with multiple products.
> Products are access-control bundles, not necessarily sold publicly.

---

**Q4. Which status code range indicates an error caused by an issue with the client's request?**

- ✅ **4XX** ← CORRECT
- 5XX
- 1XX
- 2XX

> 4XX = client errors (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, etc.)
> 5XX = server errors, 2XX = success, 1XX = informational.
