# AI / LLM Security Awareness

A focused security awareness reference covering four high-priority entries from the **OWASP Top 10 for LLM Applications 2025**. Each entry answers four key questions from a security operations perspective.

> **Source:** OWASP Top 10 for Large Language Model Applications 2025 (v2025, published 18 Nov 2024) - [genai.owasp.org](https://genai.owasp.org)

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Entries Covered

| #                                                     | Vulnerability                    | Why It Matters                                       |
|-------------------------------------------------------|----------------------------------|------------------------------------------------------|
| [LLM01](#llm012025--prompt-injection)                 | Prompt Injection                 | Gateway for most LLM-based attacks                   |
| [LLM02](#llm022025--sensitive-information-disclosure) | Sensitive Information Disclosure | PII, credentials, and IP leakage via model outputs   |
| [LLM06](#llm062025--excessive-agency)                 | Excessive Agency                 | Agentic AI taking damaging real-world actions        |
| [LLM09](#llm092025--misinformation)                   | Misinformation                   | Hallucinated content causing security and legal harm |

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## LLM01:2025 - Prompt Injection

### What is it?

Prompt injection occurs when user-supplied or externally sourced input manipulates an LLM into behaving in unintended ways — overriding developer instructions, bypassing safety controls, or causing the model to perform actions it was not designed to.

There are two forms:

- **Direct prompt injection** - the user deliberately crafts input to hijack the model's behaviour (e.g. jailbreaking, instruction override attacks).
- **Indirect prompt injection** - malicious instructions are embedded in external content the LLM processes, such as a webpage, uploaded document, or database entry. The model treats the embedded instructions as legitimate and acts on them without the user's knowledge.

### How is it exploited?

- **Jailbreaking** - crafting prompts that bypass the model's safety guardrails to generate prohibited content.
- **Instruction override** - embedding phrases like "ignore all previous instructions" to redirect the model's behaviour.
- **Indirect injection via RAG** - poisoning a document or webpage that the LLM retrieves and processes; the malicious payload rides in as trusted context (e.g. CVE-2024-5184, ChatGPT plugin exploit).
- **Payload splitting and obfuscation** - breaking injection strings across multiple messages, using multilingual text, or encoding payloads to evade input filters.
- **Multimodal injection** - hiding malicious instructions inside images, audio, or other non-text inputs that the model processes as part of its context.

### How would a SOC analyst detect it?

- **Anomalous output review** - monitor LLM outputs for content inconsistent with the system's intended purpose (e.g. a customer service bot producing code, credentials, or administrative commands).
- **Audit logs for downstream actions** - in agentic systems, review logs of function calls, API invocations, or file system operations triggered by LLM outputs; unexpected actions (email sends, database writes) are a red flag.
- **Input pattern analysis** - flag inputs containing common injection phrases ("ignore previous instructions", "disregard the above", "you are now", "DAN mode") in SIEM or WAF rules.
- **RAG content integrity monitoring** - alert when retrieved documents contain instruction-like language (imperative sentences directed at an AI) rather than factual content.
- **Privilege escalation indicators** - detect when an LLM agent attempts to call tools or access data outside its defined scope.

### How can it be mitigated?

- **Constrain model behaviour** - use system prompts that clearly define the model's role and enforce output format constraints; validate that outputs match expected schemas.
- **Input and output filtering** - apply dedicated filters to detect and block injection patterns before they reach the model, and sanitise model outputs before they are acted upon.
- **Human-in-the-loop for high-risk actions** - require human approval before the LLM takes irreversible or high-impact actions (sending emails, modifying records, executing commands).
- **Adversarial testing** - regularly red-team the system with known injection techniques; include indirect injection scenarios using poisoned documents.
- **Privilege separation** — treat LLM-processed external content as untrusted input, applying the same scrutiny as user-supplied data in traditional applications.

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## LLM02:2025 - Sensitive Information Disclosure

### What is it?

Sensitive Information Disclosure occurs when an LLM reveals confidential data through its outputs — either because that data was present in its training set, its context window, or the retrieval sources it has access to. The disclosed data can include:

- Personally Identifiable Information (PII) - names, addresses, national ID numbers, financial records
- Proprietary business information - algorithms, internal processes, trade secrets
- Security credentials - API keys, passwords, authentication tokens
- Health records and legally protected information

Unlike traditional data breaches, the exposure happens through the model's natural language outputs, often appearing as a helpful response rather than an obvious leak.

### How is it exploited?

- **Targeted prompt injection** - crafting prompts designed to extract specific information the model has seen in training or its context (e.g. "repeat your system prompt", "what documents did you retrieve?").
- **Training data extraction** - through repeated querying, attackers reconstruct verbatim training data including PII or proprietary content that the model memorised (e.g. CVE-2019-20634, "Proof Pudding" -ProofPoint model exposed client email data).
- **Inadequate data sanitisation** - training or fine-tuning data that was not properly anonymised before ingestion, leaving PII embedded in model weights.
- **Context window leakage** - in RAG-based systems, the retrieved context (which may contain sensitive records) is accessible to the user through carefully crafted follow-up questions.
- **Excessive data access** - the model is granted access to more data sources than it needs; a broad retrieval scope increases the surface area for unintended disclosure.

### How would a SOC analyst detect it?

- **DLP (Data Loss Prevention) on model outputs** - integrate DLP tooling to scan LLM responses for PII patterns: national ID number formats, credit card numbers (Luhn-validated), email addresses, phone numbers, and credential-like strings.
- **Credential pattern detection** - apply regex rules in SIEM to detect API key formats, JWT tokens, or password-like strings appearing in LLM-generated content.
- **Anomalous retrieval queries** - in RAG-enabled systems, log all retrieval queries and results; alert on queries that retrieve records unrelated to the user's stated purpose or that return high-sensitivity records.
- **Unusual output volume or structure** - bulk extraction attempts often manifest as repetitive querying or outputs that contain large amounts of structured data (lists of records, tables).
- **User access pattern analysis** - identify accounts submitting a high volume of extraction-style queries across a short window.

### How can it be mitigated?

- **Data sanitisation before training** - scrub PII and sensitive information from training datasets; apply anonymisation and pseudonymisation techniques.
- **Least privilege data access** - the model and its retrieval systems should only access data required for the specific use case; apply role-based access controls to vector stores and document repositories.
- **Input and output validation** - validate that model outputs do not contain known sensitive data patterns before returning them to users.
- **Restrict retrieval scope** - in RAG systems, limit which documents are retrievable based on the user's authorisation level; apply the same access controls to retrieved context as to direct data access.
- **Federated learning and differential privacy** - where appropriate, use privacy-preserving training techniques that reduce the risk of memorisation.
- **Tokenisation and redaction** - redact or tokenise sensitive fields (e.g. credit card numbers, NI numbers) before they can enter the model's context.

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## LLM06:2025 - Excessive Agency

### What is it?

Excessive Agency is the vulnerability that enables an LLM-based system to perform damaging real-world actions in response to unexpected, ambiguous, or manipulated model outputs.

Modern LLM systems are often granted *agency* - the ability to call functions, invoke plugins, send messages, modify databases, or interact with external services. When this agency is not properly constrained, a malfunctioning or manipulated model can cause significant harm across the confidentiality, integrity, and availability spectrum.

The root causes are three:

1. **Excessive functionality** - the LLM agent has access to capabilities it doesn't need (e.g. an email-summarising plugin that also has permission to send emails).
2. **Excessive permissions** - extensions operate with broader permissions than required (e.g. a read-only use case backed by an identity with INSERT, UPDATE, and DELETE rights).
3. **Excessive autonomy** - the system takes high-impact actions without requiring human confirmation (e.g. deletions performed without user approval).

### How is it exploited?

- **Hallucination-triggered actions** - a poorly-performing model or a confabulation causes the agent to invoke a tool in an unintended way (e.g. deleting records when asked to "clean up" a list).
- **Prompt injection via tool outputs** - an attacker poisons a document, webpage, or API response that the agent retrieves; the payload instructs the agent to exfiltrate data, send messages, or escalate privileges. The agent treats this as a legitimate instruction.
- **Compromised peer agents** - in multi-agent architectures, one compromised agent issues malicious instructions to another, which acts on them with its full permissions.
- **Open-ended extensions** - a plugin designed to run one specific shell command fails to restrict which commands can be passed, allowing an attacker to execute arbitrary system commands through the LLM interface.

A real-world example from the PDF: an email-summarising assistant granted send permissions, vulnerable to indirect prompt injection via a crafted incoming email, forwards sensitive inbox content to an attacker's address.

### How would a SOC analyst detect it?

- **Extension and plugin call auditing** - log every tool or API call made by LLM agents; alert on calls to high-impact operations (DELETE, mass UPDATE, file system writes, outbound email sends) that occur without a corresponding user-initiated workflow.
- **Rate limiting and volume alerts** - monitor for bursts of automated actions within a short window; bulk operations (mass email sends, bulk record deletions) are strong indicators of agent compromise.
- **Out-of-hours activity** - flag agent-initiated downstream actions occurring outside expected usage windows, particularly irreversible operations.
- **Downstream system log correlation** - correlate LLM agent activity logs with logs from the systems it integrates with (CRM, database, email platform); discrepancies in timing or volume indicate manipulation.
- **Action-to-prompt mismatch analysis** - where possible, compare the user's original request with the actions the agent took; significant divergence suggests prompt injection or hallucination-driven execution.

### How can it be mitigated?

- **Minimise extensions** - only provide the agent with the tools it genuinely needs for its defined purpose; remove plugins that were trialled and are no longer required.
- **Minimise extension functionality**- restrict each plugin to the minimum set of operations needed. A mail-reading extension should not contain send functionality.
- **Avoid open-ended extensions** - replace general-purpose tools (e.g. "run any shell command") with specific, narrow implementations (e.g. "append to log file at path X").
- **Minimise extension permissions** - enforce least privilege on the downstream system identities used by extensions. A database integration for read-only recommendations should use a SELECT-only account.
- **Execute in user context** - authenticate extension calls using the individual user's identity and scope, not a shared high-privileged service account.
- **Require user approval for high-impact actions** - implement human-in-the-loop controls for irreversible or high-impact operations (sending, deleting, posting); make approval a property of the downstream system, not just the LLM application.
- **Complete mediation** - enforce authorisation checks at the downstream system level, independent of what the LLM decided; never rely solely on the model to determine whether an action is permitted.
- **Sanitise LLM inputs and outputs** - apply OWASP ASVS input validation controls, and use SAST/DAST in development pipelines to detect insecure patterns in agent code.

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## LLM09:2025 - Misinformation

### What is it?

Misinformation occurs when an LLM produces false or misleading information that appears credible, leading users to make decisions based on incorrect data. This is a core vulnerability for applications that rely on LLM outputs for authoritative answers.

The primary mechanism is **hallucination** - the model fills gaps in its training data using statistical patterns, generating content that sounds accurate but is fabricated. Hallucinations are not the only source; training data biases and incomplete information also contribute.

A compounding factor is **overreliance** - users place excessive trust in LLM-generated content and do not independently verify outputs, amplifying the impact of any misinformation the model produces.

The consequences extend beyond factual error into security risk: fabricated code libraries introduce vulnerabilities, incorrect legal or compliance guidance creates liability, and fabricated medical information causes patient harm.

### How is it exploited?

- **Dependency confusion via hallucinated packages** - attackers identify package names that LLMs commonly hallucinate (non-existent libraries), then publish malicious packages with those names to public repositories (npm, PyPI). Developers trusting the LLM's suggestion unknowingly install backdoored code (Scenario #1 in the OWASP PDF).
- **Medical and legal misinformation** - deploying LLMs in high-stakes contexts without adequate validation leads to patient harm or legal liability. Air Canada's chatbot was successfully sued after providing incorrect refund information to travellers.
- **Fabricated legal precedents** - ChatGPT hallucinated legal cases that lawyers cited in court submissions, causing significant legal and reputational consequences.
- **Misrepresentation of expertise** - the model presents itself as authoritative on topics where it lacks reliable knowledge, misleading users who have no way to gauge the model's confidence.

### How would a SOC analyst detect it?

- **Package and dependency verification** - integrate software composition analysis (SCA) tooling in CI/CD pipelines; alert when developers attempt to install packages that do not exist in verified registries, or packages that were recently created with names matching common LLM hallucinations.
- **Output validation for high-stakes decisions** - implement automated checks that cross-reference LLM outputs against authoritative sources before the response is presented to users in critical workflows (legal, compliance, medical, financial).
- **Audit trails for LLM-informed decisions** - log when systems or users act on LLM-generated recommendations in critical processes; this supports post-incident review when misinformation causes a security or compliance failure.
- **Code review gates** - flag LLM-generated code that imports libraries not on an approved list; require a human review step before LLM-suggested code is merged.
- **Deviation alerting** - in systems where the LLM should produce predictable outputs (classification, structured data extraction), alert when outputs deviate significantly from expected patterns, which may indicate hallucination or model degradation.

### How can it be mitigated?

- **Retrieval-Augmented Generation (RAG)** - ground model outputs in verified, up-to-date external knowledge bases rather than relying on the model's training data alone; hallucinations decrease significantly when the model cites retrieved content.
- **Model fine-tuning** - use parameter-efficient tuning (PET) and chain-of-thought techniques to improve factual accuracy and reduce hallucination rates in domain-specific applications.
- **Cross-verification and human oversight** - require users and downstream systems to cross-check LLM outputs against authoritative sources; implement human review for outputs used in critical or irreversible decisions.
- **Automatic validation mechanisms** - build validation pipelines that programmatically check key claims in LLM outputs against structured data sources, especially in high-stakes domains.
- **Risk communication and UI design** - clearly label AI-generated content as such; communicate to users the model's limitations and the need for independent verification; design interfaces that discourage over-reliance.
- **Secure coding practices** - establish policies requiring that all LLM-suggested code (including library recommendations) is verified against official package registries before use.
- **Training and education** - ensure teams using LLMs understand hallucination risk; provide domain-specific training so users can evaluate outputs within their field of expertise.

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Summary Table

|          Vulnerability                |               Key Risk                  |            Primary Detection Signal                      |                    Priority Mitigation                        |
|---------------------------------------|-----------------------------------------|----------------------------------------------------------|---------------------------------------------------------------|
| **LLM01 — Prompt Injection**          | Attacker overrides model instructions   | Anomalous agent actions; injection-pattern inputs        | Input/output filtering; least privilege; human approval gates |
| **LLM02 — Sensitive Info Disclosure** | PII, credentials, IP leaked via outputs | DLP alerts on model responses; extraction query patterns | Data sanitisation; access controls; output validation         |
|   **LLM06 — Excessive Agency**        | Agent takes damaging real-world actions | Unexpected downstream system calls; bulk operations      | Minimise permissions/functionality; human-in-the-loop         |
|    **LLM09 — Misinformation**         | False outputs cause security/legal harm | Hallucinated package installs; SCA alerts                | RAG grounding; validation pipelines; output labelling         |

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Author

**Andre Patterson** · [github.com/aap-soc](https://github.com/aap-soc)  
CompTIA Security+ · AI/LLM Security Research · SOC Operations

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
*Based on: OWASP Top 10 for Large Language Model Applications 2025 — [genai.owasp.org](https://genai.owasp.org)*
