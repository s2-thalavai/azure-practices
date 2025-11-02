## Problems & Solutions

### 1. Do we still need to virus-scan files uploaded by vendors in our React application, even when we have Azure API Management with WAF enabled?

### Why WAF is NOT Enough

Azure APIM with WAF (Web Application Firewall) protects mainly against:

- SQL injection

- XSS

- Malicious HTTP traffic patterns

- Bot attacks

- OWASP threats

**WAF does not inspect file contents for malware.
**

If a vendor uploads a PDF/Excel/ZIP containing malware, WAF will not detect or block it.
It only sees the request metadata — not deep file scanning.

## Why Virus Scanning Is Required

Vendor uploads can include:

- Trojan in PDFs

- Macro viruses in Excel/Word

- Malicious JavaScript in PDFs

- Embedded malware inside ZIP files

If your system stores or forwards these files without scanning, you expose your company and downstream users.

For enterprise procure-to-pay onboarding, compliance and security teams expect file scanning.

Regulatory expectations:

- ISO 27001

- SOC2

- PCI-DSS (if financial docs involved)

- Enterprise vendor security standards

## Architecture Example

```

React → APIM + WAF → Upload API
            │
            ▼
Quarantine Blob Storage
            │ (Blob created event)
            ▼
Azure Function → ClamAV / Azure Malware Scanner
            │
     Clean ─┴─ Infected
      │           │
Move to final   Delete file
container       Notify vendor


```

💡 Tip

> Consider size limit + allowed MIME types + extension whitelist as well.
That reduces attack surface.

Example allowed types: pdf, jpg, docx, xlsx, zip

## 🎯 Summary

| Requirement                 | Reason                                   |
| --------------------------- | ---------------------------------------- |
| APIM + WAF                  | Protects HTTP traffic only, not files    |
| Virus Scan                  | Prevents malware embedded in vendor docs |
| Quarantine → Scan → Approve | Best practice for onboarding systems     |

So yes, even with Azure APIM + WAF, you MUST virus scan uploaded documents.


## Reference architecture (event-driven, quarantine → scan → promote)

