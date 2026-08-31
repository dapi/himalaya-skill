# Himalaya agent skill

An agent skill for safely reading, composing, sending, and organizing email with the [Himalaya CLI](https://github.com/pimalaya/himalaya).

The skill preserves explicit confirmation gates for mail mutations and generates standards-compliant MIME for non-ASCII headers and multipart messages.

## Install

```sh
npx skills add dapi/himalaya-skill --skill himalaya -g -a codex -a claude-code -a kimi-code-cli -y
```
