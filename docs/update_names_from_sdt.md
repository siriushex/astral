# Update Stream Names From SDT (service_name)

This tool updates `stream.config.name` using the SDT service name discovered via `astral --analyze`.

Goals:
- Safe by default: dry-run unless `--apply` is provided.
- Low load: small default parallelism, per-input timeouts, and a stream rate limit.
- Best-effort parsing: supports multiple analyze output variants.

## What It Does
For each selected stream:
1. Reads `config.input[]` in priority order.
2. Runs `astral --analyze` on each input until it finds SDT `service_name`.
3. Chooses the name:
   - If `pnr`/`set_pnr` is present (or `#pnr=` in input URL fragment), prefers the matching SID.
   - Otherwise uses the first SDT service name seen.
4. Updates `stream.config.name` via `PUT /api/v1/streams/<id>` (full config payload).

## Requirements
- Python 3
- Access to Astral API (`/api/v1`) as an admin user (default: `admin/admin`)
- Local `astral` binary available:
  - defaults to `./astral` (repo root) or `astral` from `PATH`
  - override with `--astral-bin` (alias: `--astra-bin`)

## Examples

Dry-run (default):
```bash
python3 tools/update_stream_names_from_sdt.py --api http://127.0.0.1:9060
```

Apply changes (writes names) with a config backup export:
```bash
python3 tools/update_stream_names_from_sdt.py \
  --api http://127.0.0.1:9060 \
  --backup \
  --apply
```

Apply with a custom backup file path:
```bash
python3 tools/update_stream_names_from_sdt.py \
  --api http://127.0.0.1:9060 \
  --backup /tmp/astral-export.json \
  --apply
```

Only streams matching regex by id or name:
```bash
python3 tools/update_stream_names_from_sdt.py \
  --api http://127.0.0.1:9060 \
  --match "(ntv|viasat|bridge)"
```

Target explicit stream ids (repeatable):
```bash
python3 tools/update_stream_names_from_sdt.py \
  --api http://127.0.0.1:9060 \
  --id a019 --id Megahit
```

Force update even if current name looks human:
```bash
python3 tools/update_stream_names_from_sdt.py --api http://127.0.0.1:9060 --force --apply
```

## Resource Limits (Important)
Defaults are tuned for low load:
- `--parallel 2` (max concurrent streams)
- `--timeout-sec 10` (per input)
- `--rate-per-min 30` (streams per minute)

If you have a large config, keep `--parallel` small.

## Testing / Parser Debug
You can test parsing without a live stream by providing static analyzer output:
```bash
python3 tools/update_stream_names_from_sdt.py \
  --api http://127.0.0.1:9060 \
  --mock-analyze-file /path/to/analyze_output.txt
```

