# API Key Resolution

Resolve `$API_KEY` by checking these sources in order. Stop at the first non-empty value.

## 1. Shell environment

```bash
echo "$CUSTOMGPT_API_KEY"
```

If non-empty, use it.

## 2. `.env` file

Walk up from `$PWD` toward `/`, checking each directory for a `.env` file containing `CUSTOMGPT_API_KEY=`. Stop at the first match:

```bash
dir="$PWD"
while [ "$dir" != "/" ]; do
  if [ -f "$dir/.env" ]; then
    val=$(grep -m1 "^CUSTOMGPT_API_KEY=" "$dir/.env" | cut -d= -f2-)
    if [ -n "$val" ]; then echo "$val"; break; fi
  fi
  dir=$(dirname "$dir")
done
```

## 3. Config file

```bash
cat ~/.claude/customgpt-config.json
```

Extract the `apiKey` field value.

## If no key found

Ask the user:

> "Please provide your CustomGPT.ai API key. You can find it at https://app.customgpt.ai/profile#api-keys. If you don't have the account, you can get started at https://app.customgpt.ai/register"

Once provided, save for future sessions:

```bash
echo '{"apiKey":"KEY_HERE"}' > ~/.claude/customgpt-config.json
```

Confirm: "API key saved to `~/.claude/customgpt-config.json`."

## Auth error during an API call

If any API call returns HTTP 401, stop and tell the user:

> "API key invalid or expired. Please create new one at https://app.customgpt.ai/profile#api-keys, then save it with: `echo '{\"apiKey\":\"NEW_KEY\"}' > ~/.claude/customgpt-config.json`"
