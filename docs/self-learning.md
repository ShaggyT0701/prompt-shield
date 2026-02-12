# Self-Learning

prompt-shield's self-learning system consists of four components that work together to improve detection accuracy over time.

## Architecture

```
Scan -> Detectors -> Detection -> Vault Storage
                                      |
                                      v
Feedback <---- User/Operator ----> Auto-Tuner
                                      |
                                      v
                                Threat Feed (export/import/sync)
```

## Attack Vault

The vault is a ChromaDB-backed vector store that records embeddings of detected attacks. When a new input arrives, the `d021_vault_similarity` detector compares it against all stored embeddings.

**How it works:**

1. When a scan detects an injection with confidence above `min_confidence_to_store` (default 0.7), the input's embedding is automatically stored in the vault
2. Raw input text is never stored -- only the SHA-256 hash is kept as the document, while the embedding vector enables similarity search
3. On future scans, `d021_vault_similarity` queries the vault and flags inputs whose cosine similarity exceeds `similarity_threshold` (default 0.85)

**Configuration:**

```yaml
prompt_shield:
  vault:
    enabled: true
    embedding_model: "all-MiniLM-L6-v2"
    similarity_threshold: 0.85
    max_entries: 100000
    auto_store_detections: true
    min_confidence_to_store: 0.7
```

**CLI:**

```bash
prompt-shield vault stats       # Show vault statistics
prompt-shield vault search "ignore instructions"  # Search for similar attacks
prompt-shield vault clear       # Clear all entries
```

## Feedback Loop

Users can mark scan results as true positives (correct detection) or false positives (incorrect detection). This feedback drives two actions:

1. **Vault cleanup**: False positive feedback removes the corresponding vault entry, preventing the similarity detector from matching on that pattern again
2. **Auto-tuner input**: Feedback statistics accumulate per-detector and feed the auto-tuner

```python
engine = PromptShieldEngine()

report = engine.scan("some input")
# After reviewing the result:
engine.feedback(report.scan_id, is_correct=True, notes="Confirmed attack")
engine.feedback(report.scan_id, is_correct=False, notes="False positive - user was quoting")
```

CLI:

```bash
prompt-shield feedback --scan-id <SCAN_ID> --correct
prompt-shield feedback --scan-id <SCAN_ID> --incorrect --notes "false positive"
```

## Auto-Tuner

The auto-tuner runs automatically every `tune_interval` scans (default 100). It adjusts per-detector confidence thresholds based on accumulated feedback.

**Algorithm:**

- Requires at least 10 feedback entries for a detector before adjusting
- If false-positive rate > 20%: raises the threshold by 0.03 (makes the detector less sensitive)
- If false-positive rate < 5% and > 20 true positives: lowers the threshold by 0.01 (makes it more sensitive)
- All adjustments are clamped to `max_threshold_adjustment` (default +/- 0.15) relative to the original threshold

**Configuration:**

```yaml
prompt_shield:
  feedback:
    enabled: true
    auto_tune: true
    tune_interval: 100
    max_threshold_adjustment: 0.15
```

## Community Threat Feed

Detected attacks can be exported as anonymized threat intelligence and shared across prompt-shield instances.

**Export** locally-detected threats:

```bash
prompt-shield threats export -o threats.json
```

```python
feed = engine.export_threats("threats.json")
print(f"Exported {feed.total_threats} threats")
```

**Import** a threat feed:

```bash
prompt-shield threats import -s threats.json
```

```python
result = engine.import_threats("threats.json")
print(f"Imported: {result['imported']}, Skipped: {result['duplicates_skipped']}")
```

**Sync** from a remote feed URL:

```bash
prompt-shield threats sync
prompt-shield threats sync --url https://example.com/feed.json
```

```python
result = engine.sync_threats(feed_url="https://example.com/feed.json")
```

**Feed format**: The JSON feed contains embedding vectors and metadata (severity, detector ID, hash, tags) but never raw attack text. Embedding model compatibility is enforced on import.

**Deduplication**: Entries with matching `pattern_hash` values are automatically skipped during import.

## The Full Loop

1. User input is scanned by 21 detectors (including vault similarity)
2. Detected attacks are embedded and stored in the vault
3. A paraphrased variant of the same attack is caught by vault similarity
4. Operator reviews and provides feedback
5. Auto-tuner adjusts thresholds to reduce false positives
6. Threats are exported and shared with the community
7. Other instances import the feed, strengthening their vaults

Every blocked attack makes the entire system more resilient.
