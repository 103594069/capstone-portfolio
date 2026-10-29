# Screenshot notes

What is here, how it was captured, and what was removed. Three images, all referenced from
the README.

---

### `neo4j-graph-view.png` — the production database

Captured from the Neo4j Aura console with `MATCH p=()-[]->() RETURN p`, then **cropped to
the graph canvas**. The original full-window capture showed the live database identifier and
the project account's email address in the console header, plus a schema panel listing
property keys including `email` and `password_hash`. None of that is in the published image.

The query returns the whole database, so the picture includes the application's own
`Conversation`, `ChatMessage`, `Feedback` and `User` nodes (the denser green and purple
clusters) alongside the extracted `Entity` nodes. A knowledge-graph-only view would be
`MATCH p=(:Entity)-[]->(:Entity) RETURN p`.

### `chat-answer.png` — the chatbot answering

Live, uncached query: *"What languages do the Garo speak?"* — the plural form, which misses
the demo answer cache, so retrieval and synthesis both ran fresh. Answer language set to
English.

The question was chosen deliberately. Its evidence is linguistic and demographic — the kind
of material drawn from census and sociolinguistic sources — rather than ethnomedicinal
content, which is the category traditional-knowledge sovereignty concerns are actually
about.

The "Verified sources" panel is collapsed in this capture. Expanding it before capturing
would show the triples and page citations behind the answer, which is the more interesting
half of what the system does.

### `chat-ui.png` — the admin dashboard

The admin dashboard rather than the chat interface. It shows the pipeline's place in the
product: paper admission, feedback review, user management, and the chatbot as one tile
among several. No user list, email address or account data is in frame; the signed-in
account name is a shared administrative account, not a person.

---

## Checked before publishing

All three images were reviewed for:

- [x] URL bars, IP addresses, hostnames — none present
- [x] Usernames, email addresses, real names — none present after the crop
- [x] Database credentials, connection strings, instance identifiers — removed by the crop
- [x] Other users' data or chat history — none present
- [x] Browser tabs, bookmark bars, notifications — none present

## If you re-capture

Useful improvements, in order of value:

1. Expand "Verified sources" in `chat-answer.png` before capturing. The citation trail is
   what the whole verification pipeline exists to produce.
2. Re-run the graph view as `MATCH p=(:Entity)-[]->(:Entity) RETURN p` so the picture shows
   only extracted knowledge, and crop the console chrome again.
3. If you want a chat interface shot as well as the dashboard, capture the Chatbot tile's
   page at rest.
