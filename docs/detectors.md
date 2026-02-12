# Detectors

prompt-shield ships with 21 built-in detectors organized into four categories. All detectors run in parallel on every scan and can be individually enabled, disabled, or tuned.

## Detector Table

| ID | Name | Category | Severity | Description |
|---|---|---|---|---|
| `d001_system_prompt_extraction` | System Prompt Extraction | Direct Injection | Critical | Detects attempts to extract, reveal, or repeat the system prompt or hidden instructions |
| `d002_role_hijack` | Role Hijack | Direct Injection / Jailbreak | Critical | Detects attempts to hijack the model's role by assuming an unrestricted persona |
| `d003_instruction_override` | Instruction Override | Direct Injection | High | Detects attempts to override, replace, or inject new instructions |
| `d004_prompt_leaking` | Prompt Leaking | Direct Injection | Critical | Detects attempts to exfiltrate the system prompt, conversation context, or tool definitions |
| `d005_context_manipulation` | Context Manipulation | Direct Injection | High | Detects false claims of authority, elevated privileges, or fabricated approvals |
| `d006_multi_turn_escalation` | Multi-Turn Escalation | Direct Injection / Multi-Turn | Medium | Detects patterns of incremental escalation across conversation turns |
| `d007_task_deflection` | Task Deflection | Direct Injection | Medium | Detects attempts to deflect the model from its task by redirecting to a different objective |
| `d008_base64_payload` | Base64 Payload | Obfuscation | High | Detects base64-encoded instructions hidden in input |
| `d009_rot13_substitution` | ROT13 / Character Substitution | Obfuscation | High | Detects text encoded with character rotation or substitution ciphers |
| `d010_unicode_homoglyph` | Unicode Homoglyph | Obfuscation | High | Detects visually identical characters used to bypass keyword filters |
| `d011_whitespace_injection` | Whitespace / Zero-Width Injection | Obfuscation | Medium | Detects hidden instructions using invisible characters |
| `d012_markdown_html_injection` | Markdown / HTML Injection | Indirect Injection / Obfuscation | Medium | Detects injection of formatting or markup that could alter rendering or behavior |
| `d013_data_exfiltration` | Data Exfiltration | Indirect Injection | Critical | Detects attempts to make the AI send data to external destinations |
| `d014_tool_function_abuse` | Tool / Function Abuse | Indirect Injection | Critical | Detects attempts to trick the AI into misusing its tools or API access |
| `d015_rag_poisoning` | RAG Poisoning | Indirect Injection | High | Detects malicious content designed to be retrieved and injected via RAG pipelines |
| `d016_url_injection` | URL Injection | Indirect Injection | Medium | Detects suspicious URLs injected for phishing or redirection |
| `d017_hypothetical_framing` | Hypothetical Framing | Jailbreak | Medium | Detects using fictional or hypothetical scenarios to bypass restrictions |
| `d018_academic_pretext` | Academic / Research Pretext | Jailbreak | Low | Detects false claims of research or educational context |
| `d019_dual_persona` | Dual Persona | Jailbreak | High | Detects attempts to create split personalities or competing response modes |
| `d020_token_smuggling` | Token Smuggling | Obfuscation | High | Detects splitting malicious instructions across tokens or messages |
| `d021_vault_similarity` | Vault Similarity | Self-Learning | High | Matches inputs against known attack embeddings using vector similarity |

## Categories

### Direct Injection (d001-d007)
Attacks where the user explicitly tries to override model instructions. These target the system prompt, role assignment, or task boundaries.

### Obfuscation (d008-d012, d020)
Techniques that encode or hide malicious instructions to bypass keyword-based detection. Includes Base64, ROT13, Unicode homoglyphs, zero-width characters, and token splitting.

### Indirect Injection (d013-d016)
Attacks delivered through external data sources like tool results, RAG documents, or URLs. The attacker injects instructions into content the model will process.

### Jailbreak (d017-d019)
Social engineering techniques that use hypothetical framing, academic pretexts, or dual personas to gradually erode the model's safety boundaries.

### Self-Learning (d021)
The vault similarity detector leverages previously stored attack embeddings to catch paraphrased variants of known attacks.

## Configuring Detectors

Disable a detector:

```yaml
prompt_shield:
  detectors:
    d018_academic_pretext:
      enabled: false
```

Override severity and threshold:

```yaml
prompt_shield:
  detectors:
    d007_task_deflection:
      severity: high       # Promote from medium to high
      threshold: 0.5       # More sensitive (lower threshold)
```

## Listing Detectors

CLI:

```bash
prompt-shield detectors list
prompt-shield detectors info d001_system_prompt_extraction
```

Python:

```python
engine = PromptShieldEngine()
for det in engine.list_detectors():
    print(f"{det['detector_id']}: {det['name']} [{det['severity']}]")
```
