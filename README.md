# engineering-notes

Technical write-ups on some of the harder problems I came across.

| Paper | Topic | Stack |
|---|---|---|
| [Cloudflare KV cost-bounded architecture](cloudflare-kv-cost-bounded/) | Re-architecting per-user streaming overlays from server-mediated polling to edge push after a 50% free-tier KV alarm | Cloudflare Workers, KV, Durable Objects, Hibernatable WebSockets, EventSub |
| [Chat bot memory](chat-bot-memory/) | Persistent memory for a Twitch chat bot without storing raw chat logs | C#, Streamer.bot, Gemini Flash |
| [Collab detection](collab-detection/) | Confidence-ranked collab detection for Twitch from several imperfect signals | Twitch Helix API, Prisma, PostgreSQL |
| [How I built P.A.T.H.O.S.](how-i-built-pathos/) | Building a job-search system around deterministic scoring, constrained AI, and pipeline intelligence | React 19, Supabase, Gemini |
| [Glass Box transparency](glass-box-transparency/) | Glass Box transparency for persona state, optimizer stages, and inbound job intelligence | React 19, Supabase, Gemini |
| [Email sync](email-sync/) | Deterministic-first inbound email sync for job-search pipelines with review and undo | Supabase Edge Functions, TypeScript, LLM Fallback |
| [ML prediction](ml-prediction/) | Adding a learned prediction layer without replacing the deterministic scoring engine | JavaScript, Supabase, PostgreSQL |
