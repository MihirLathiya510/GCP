# Quiz — Apigee: Route Rules and the Debug Tool

**Score: 100%**

---

**Q1. Which of the following is not configured for an environment group?**

- Environments
- Name
- Hostnames
- ✅ **Base path** ← CORRECT

> Base path is configured in a proxy endpoint, not an environment group.

---

**Q2. Which parts of a REST API request together typically represent the operation being performed? (Select two)**

- The base path
- ✅ **The HTTP verb** ← CORRECT
- ✅ **The path suffix** ← CORRECT
- The headers
- The message body

---

**Q3. Which part of a proxy determines the target endpoint that will be used?**

- HTTPTargetConnection
- ✅ **Route rule** ← CORRECT
- Post flow
- Target server

> Route rules determine which target endpoint is used, if any.

---

**Q4. Which of the following combinations of proxy and target endpoints is not legal for an API proxy?**

- More than one proxy endpoint and one target endpoint
- One proxy endpoint and more than one target endpoint
- One proxy endpoint and zero target endpoints
- ✅ **Zero proxy endpoints and one target endpoint** ← CORRECT

> The proxy endpoint is the entry point for an API proxy — at least one is required.
