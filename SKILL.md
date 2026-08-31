---
name: himalaya
description: Manage configured email accounts with the Himalaya CLI. Use when the user asks to check, search, read, draft, send, reply to, forward, move, copy, or delete email; to list mailboxes or accounts; or refers to inbox, unread messages, or mail folders. Always obtain explicit confirmation before sending, moving, copying, or deleting mail.
---

# Himalaya email

Use the user's existing Himalaya configuration. Never create or edit mail configuration, passwords, tokens, or account settings.

## Preflight

Run `himalaya account list` before the first mail operation in a session. If it fails, report the error and stop. Do not ask for credentials in chat or logs.

Use `-a <account>` whenever more than one account is configured or the sender account is material.

Before sending, make sure the `From` address is an identity accepted by the selected account. A working SMTP login does not imply that the server permits arbitrary aliases. Use `himalaya account check -a <account> -b smtp --json` when authentication or sender ownership is uncertain.

## Read and search

Read-only actions need no confirmation:

```bash
himalaya mailbox list -a <account>
himalaya envelope list -a <account> -m Inbox -s 20
himalaya envelope search -a <account> -m Inbox "from sender@example.com and after 2025-01-01"
```

Prefer envelope operations for delivery checks because they do not mark a message as read. Do not use `message read` merely to verify arrival.

Summarize message IDs with any results so the user can identify the intended message precisely.

## Drafting and sending

Before sending, establish the account / `From`, recipients, subject, and exact body. Present the final draft and wait for a clear confirmation to send. A request to draft or compose is not permission to send.

Use the selected account and a matching `From` header. Verify its SMTP backend during preflight, but when saving a sent copy do not force `-b smtp` on `message send`: the default `auto` mode must combine the account's SMTP outgoing backend with its IMAP/JMAP storage backend. Do not hand-write raw MIME when a header contains non-ASCII text or the message contains HTML/multipart content. Generate the message with Python's standard `email` package so headers are RFC 2047 encoded and the body has valid MIME boundaries and transfer encoding:

```bash
python3 - <<'PY' | himalaya message send -a sender@example.com --save 'Sent' --json
from email.headerregistry import Address
from email.message import EmailMessage
from email.policy import SMTP
import sys

msg = EmailMessage(policy=SMTP)
msg["From"] = Address(display_name="Sender Name", username="sender", domain="example.com")
msg["To"] = Address(display_name="Recipient Name", username="recipient", domain="example.net")
msg["Subject"] = "Тема с корректной кодировкой"
msg.set_content("Plain-text body\n", charset="utf-8")
msg.add_alternative("<p>HTML body</p>", subtype="html", charset="utf-8")
sys.stdout.buffer.write(msg.as_bytes())
PY
```

For every approved send, save the exact submitted MIME message to the account's sent mailbox in the same command with `--save <MAILBOX>`. Before the first send for an account, resolve the actual sent mailbox with `himalaya mailbox list -a <account>`; raw backend mailbox names are valid when no `sent` alias exists. For `danil@brandymint.ru`, use `--save 'Отправленные'`.

Require the combined success result (`Message successfully saved and sent` or its backend equivalent), then verify the saved envelope by recipient, subject, and timestamp and report its mailbox and message ID. If SMTP succeeds but saving the copy fails, do not resend: report the ambiguous partial failure and preserve the generated MIME so it can be added to the sent mailbox separately after explicit confirmation. Do not add the sender in `Cc` merely to create a sent record. Use self-`Cc` only as an explicitly approved fallback when the account cannot save to a sent mailbox; include that `Cc` in the final recipient list shown for approval.

For replies and forwards, confirm the exact source message ID, target recipients, and the final content before sending. Do not expose Bcc recipients in a summary.

Treat `Message successfully sent` as SMTP acceptance, not proof of delivery. When the recipient inbox is available through a configured account, verify the new envelope by subject, sender, recipient, and timestamp without opening it. Report the message ID. If the envelope has blank or malformed headers, fix MIME generation and send a uniquely labelled test only when the user has authorized iterative testing.

## Organizing mail

Copying, moving, and deleting are state-changing. First show the exact message IDs, source mailbox, target mailbox or delete action, and ask for explicit confirmation. Default to one message at a time; do not perform a bulk action unless the user explicitly supplies the IDs or range.

After confirmation, use the Himalaya command syntax supported by the installed version. Report the affected IDs and destination. If the command fails, show the error and do not retry with a destructive alternative.

## Safety

- Do not send, move, copy, or delete based on an inferred intent.
- Do not modify flags or mark messages as read unless the user asks.
- Keep attachments local and mention their filename and recipient before sending.
- Redact credentials, tokens, and sensitive email contents from summaries unless the user explicitly asks for them.
- After an ambiguous send error, check the recipient inbox before retrying so a successful SMTP submission followed by a mailbox-copy failure does not create a duplicate.
