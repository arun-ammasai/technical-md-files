# Runbook: Adding a Custom Recognizer to POC 1 (Presidio Analyzer)

**Context:** POC 1 (`presidio-analyzer`, port 5002) loads its recognizer list
from a YAML file controlled by the `RECOGNIZER_REGISTRY_CONF_FILE` env var.
By default it points at a bundled file inside the image. To add custom
recognizers without losing the built-ins, you mount your own YAML (seeded
from the image's default) and point that env var at it.

## Step 1 — Seed your config file from the image's own defaults

Only do this once per file — skip this step if `poc1/recognizers-config.yaml`
already exists and already has your previous custom entries in it.

```bash
docker cp presidio-analyzer:/app/presidio_analyzer/conf/default_recognizers.yaml /Users/aammasa/presidio-local/poc1/recognizers-config.yaml
```

This preserves every built-in recognizer (email, phone, credit card, SSN,
etc.) as the base.

## Step 2 — Append your custom recognizer(s) to the end of the file

Open `/Users/aammasa/presidio-local/poc1/recognizers-config.yaml`, go to the
**very end**, and add your entry *inside* the existing `recognizers:` list
(same indentation as the entries already there — 2-space `- name:`, 4-space
nested fields). Do **not** select-all/replace the file; append only.

Template for one recognizer:

```yaml
  - name: <SomeIdRecognizer>
    supported_language: "en"
    supported_entity: "<ENTITY_TYPE>"
    patterns:
      - name: "<description>"
        regex: "<escaped-regex>"
        score: 0.9
    context: [<keyword1>, <keyword2>]
```

Notes:

- No `type:` field needed — an entry with no `type` key defaults to a custom
  pattern recognizer.
- Regex special characters (`\b`, `\d`) must be double-escaped (`\\b`,
  `\\d`) since this is a YAML double-quoted string.
- `context` words boost confidence when they appear near the match — keep
  them lowercase.

## Step 3 — Validate the file structure before touching Docker

This specific check catches the most common mistake (accidentally replacing
the whole file so it becomes a bare list instead of a mapping):

```bash
python3 -c "
import yaml
doc = yaml.safe_load(open('/Users/aammasa/presidio-local/poc1/recognizers-config.yaml'))
assert isinstance(doc, dict), f'top level is {type(doc)}, expected dict'
assert 'recognizers' in doc
names = [r['name'] for r in doc['recognizers']]
print('OK -', len(doc['recognizers']), 'recognizers, including:', '<SomeIdRecognizer>' in names)
"
```

## Step 4 — Wire the file into `docker-compose.yml` (one-time setup)

Skip this step if it's already wired from a previous recognizer addition.
Under the `presidio-analyzer` service:

```yaml
  presidio-analyzer:
    image: ghcr.io/data-privacy-stack/presidio-analyzer:latest
    container_name: presidio-analyzer
    ports:
      - "127.0.0.1:5002:3000"
    environment:
      RECOGNIZER_REGISTRY_CONF_FILE: /app/poc1/recognizers-config.yaml
    volumes:
      - ./poc1/recognizers-config.yaml:/app/poc1/recognizers-config.yaml:ro
    restart: unless-stopped
```

## Step 5 — Validate the compose file

```bash
cd /Users/aammasa/presidio-local
docker compose config -q && echo "compose OK"
```

## Step 6 — Recreate only the POC 1 container

Never recreate the whole stack for a POC 1-only change — leave POC 2, POC 3,
and the Anonymizer alone:

```bash
docker compose up -d --force-recreate presidio-analyzer
```

## Step 7 — Confirm it actually came up healthy

```bash
docker ps -a --filter name=presidio-analyzer --format '{{.Names}}\t{{.Status}}'
```

Must say `Up ... (healthy)`. If it says `Restarting`, check logs
immediately:

```bash
docker logs presidio-analyzer 2>&1 | tail -40
```

A line like `The configuration in file ... should be a valid YAML, got
<class 'list'>` means Step 2 broke the file structure — go back and fix it.

## Step 8 — Confirm the recognizer loaded (not just "container healthy")

```bash
docker logs presidio-analyzer 2>&1 | grep -i "Loaded recognizer: <SomeIdRecognizer>"
```

## Step 9 — Functional verification

Quick curl check:

```bash
curl -s -X POST "http://localhost:5002/analyze" \
  -H "Content-Type: application/json" \
  -d '{"text":"<sample text containing your new identifier>","language":"en"}'
```

Look for your `entity_type` in the response.

Or run the full end-to-end example (detect + anonymize):

```bash
cd /Users/aammasa/agentic-platform-workspace/pii-redaction-prototype
python3 examples/poc1_detect_and_anonymize.py
```

## Troubleshooting quick-reference

| Symptom | Likely cause |
|---|---|
| `Connection refused` on port 5002 | Container crash-looping — check `docker logs` |
| `got <class 'list'>` in logs | YAML file's top-level is a bare list, not a mapping — file was replaced instead of appended to |
| Entity not detected but container healthy | Recognizer not in the file, regex doesn't match the text, or `score` is below the `score_threshold` sent in the `/analyze` request |
| Built-ins (email, phone, etc.) stop working | The custom file replaced the default registry instead of being seeded from it — redo Step 1 |
