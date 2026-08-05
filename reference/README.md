# Reference implementation

This directory will hold the open-source reference implementation of CyberProtocol:
an **issuer** (produces Seals) and a **verifier** (validates Seals against a resolver).

> **Status:** planned. The specification in [`../spec`](../spec) is the source of truth;
> code here will implement it. Contributions welcome, see [CONTRIBUTING](../CONTRIBUTING.md).

## Planned surface

```bash
# Verify an output's Seal
cyberprotocol verify ./response.json
# -> Seal valid | issuer did:cyber:agent:z6Mk... | EU AI Act: limited-risk
```

```js
import { verifySeal } from "@cyberprotocol/verify";

const result = await verifySeal(seal, {
  resolver: "https://resolver.cyberprotocol.io"
});
```

## Design goals

- **No proprietary dependency.** Verification runs with open tooling and public keys.
- **Interoperable.** Reuse W3C DID/VC and Data Integrity libraries rather than reimplementing crypto.
- **Small core.** Keep the verifier auditable; keep policy (compliance mapping) in data, not code.
