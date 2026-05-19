# solo-saas-stack

The exact infrastructure that runs a UK SaaS solo, on free tiers, for under £40/month.

This is a notes-as-portfolio repo. No frameworks to install, no boilerplate to fork. Just the receipts — what runs, what it costs, why I chose it, what I would do differently with more money.

## The whole stack

| Layer | Choice | Cost | Why |
|---|---|---|---|
| Compute | Oracle Cloud Always Free, ARM 24GB / 4 cores | £0/mo | Permanently free, beats any small AWS or DigitalOcean instance I have paid for. The ARM bit matters — most images and Python wheels work without fuss; the surprises were minor. |
| Frontend hosting | Cloudflare Pages | £0/mo | Free tier handles every spike of organic search traffic. Build + deploy via `wrangler pages deploy` in 5 seconds. |
| Dynamic backend | FastAPI on Oracle, exposed via Cloudflare Tunnel | £0/mo | The tunnel pattern means I never open ports on Oracle's firewall. Cloudflare terminates TLS, tunnel forwards to `127.0.0.1:8000`. The static + dynamic split is on the same domain. |
| Email | Resend free tier | £0/mo | 3,000 transactional sends per month, 100 per day. Currently using ~8% of cap. The SDK is honest, the webhooks are sane. |
| LLM inference | Anthropic Claude API | ~£14/mo | Pay per token. Sonnet 4.5 for the conversational layer, Haiku for the cheap classification work. Single biggest line item and the one I actively want to grow. |
| Payments | Stripe | Free until I take money | Then percentage cut. No platform fee, no monthly minimum. |
| Database | JSON files on disk | £0/mo | I am not joking. The day I outgrow that I will switch to SQLite. The day I outgrow SQLite I will switch to Postgres. Probably three years out. |
| Domain | Cloudflare Registrar | £8/yr | At-cost domain pricing. No upsells, no rip-off renewals. |
| DNS / TLS | Cloudflare | £0/mo | Bundled with the domain. Anycast TLS, real DDoS coverage. |
| Push notifications | ntfy.sh | £0/mo | Phone alerts for hot leads. Self-hostable if needed, free tier works at solo scale. |
| Cron / scheduling | systemd timers + `at` jobs | £0/mo | Lives on the same Oracle box. No additional service to run. |
| Observability | `journalctl` + plain text log files + a daily digest cron | £0/mo | At solo scale, you can read your own logs. I added structured digest emails (top errors, send counts, reply queue size) to my inbox every evening. |
| Monitoring | A Cloudflare Healthcheck pinging `/` every minute | £0/mo | Email alert if 3 consecutive checks fail. |

**Total monthly burn: under £40.**

Less than a weekly Oyster card.

## Decision log

### Why Oracle Cloud Always Free over AWS / DigitalOcean / Fly

I went Oracle ARM Always Free because:
- 24GB RAM and 4 cores is not a hobbyist tier, it is a real machine
- Permanently free, not a 12-month promotional credit
- I have a stable IP, not a serverless cold-start lottery
- Ubuntu 22 LTS with full root, systemd, cron, the lot

What I would have given up by going elsewhere:
- AWS free tier: 750hrs/month of t3.micro (1GB RAM) for 12 months then paid. Not enough RAM for my LLM-adjacent workload, and a billing cliff after a year.
- DigitalOcean basic droplet: $4-6/month from day one. Not a deal-breaker but I would rather spend that on Claude tokens.
- Fly.io: great DX but you pay from day one for anything serious. Their free allowance is for tiny apps.
- Render / Railway: lock-in plus pricing that punishes traffic. Avoided.

The downsides I actually hit:
- Oracle's web console is the worst major-cloud UI by some distance. I avoid it. SSH does the work.
- The boot-volume default is 47GB, not negotiable. I have never come close to filling it.
- The free tier requires keeping the instance "active" — a stopped instance for 7+ days can be reclaimed. Mine has been up continuously, so this has never bitten.

### Why Cloudflare Tunnel for the dynamic backend

The simple version: I do not want to expose port 80/443 directly on Oracle. Cloudflare Tunnel runs as a daemon on Oracle, opens an outbound connection to Cloudflare's edge, and Cloudflare routes inbound HTTPS requests through that tunnel.

Benefits I get:
- TLS termination at the edge — never touch certificates myself
- DDoS protection — for free, at solo scale, more than sufficient
- WAF rules — I block known bad IP ranges, bot-net signatures, scraper user-agents
- Geographic routing — I can write rules like "GET on `/admin/` only from IPs that have ever logged in" if I want
- No open ports on Oracle's firewall — the box is genuinely unreachable except through the tunnel

The downside: tunnel daemon has to be running. I set it up as a systemd unit with restart-on-failure. It has been up for months without intervention.

### Why JSON files on disk for the database

This is the choice that most surprises people, so it deserves the most explanation.

At my current scale (small but real money flowing through), the cost-benefit on database choice is:

- A managed Postgres (Supabase, Neon, RDS) costs me at least $25/mo and adds a network hop on every read
- A local SQLite would mean writing migrations, managing schema, dealing with file-locking semantics during concurrent writes
- A NoSQL service (Mongo Atlas, DynamoDB) would be massive overkill and harder to debug

JSON files on disk give me:
- Zero cost
- Zero migrations — schema changes by adding fields, code keeps reading them
- Trivial to grep, jq, or copy for backups (`cp customers.jsonl customers.jsonl.bak`)
- Atomic writes via the `os.replace(tmp, target)` pattern — survives a crash mid-write
- Genuinely thread-safe at single-process FastAPI (uvicorn) workloads because I write everything through one helper function with a `threading.Lock`

When would I switch?
- When `len(customers) > 10,000` and the full-file scan for `find_by_email` becomes slow
- When I need joins across more than two collections
- When I have a paying customer who demands a backup strategy that JSON-file-copy can't satisfy

Until then, the JSON-file pattern is faster to iterate on than any database. I treat it as a feature, not a limitation.

### Why I pay for Claude over an open-source model

I did the math. At my volume (hundreds of LLM calls per day), Claude Sonnet 4.5 costs me ~£14/mo. To self-host a 70B-parameter open-source model with comparable quality, I would need at minimum a GPU instance ($150-300/mo) plus engineering time to maintain it.

So the decision is "spend £14 a month and treat inference as a solved problem" vs "spend £200+ a month and own the headache." The math is obvious.

The day Anthropic prices materially change, I would re-evaluate. The day open-source models hit Sonnet 4.5 quality at small-GPU runtime, I would re-evaluate. Neither has happened yet.

## What I would do differently with more money

If somebody handed me £10k/mo of cloud spend tomorrow, the things I would change:

1. **Move to multi-region.** Right now Oracle is single-region (UK South). For a service-business audience that is fine. For a global tool I would want presence in at least US East and Singapore.
2. **Postgres + managed backups.** Not because JSON files are failing me, but because I would want point-in-time recovery, replicas for read traffic, and the ability to grant a future engineering hire SQL access without showing them the entire `customers/` directory.
3. **Sentry or equivalent.** Right now I read logs. At higher scale I would want structured error aggregation with first-occurrence pinning.
4. **A staging environment.** Currently I push directly to Oracle. Works at one-person scale. Would not work with even one collaborator.

But notice what does NOT change:

- I would not move off Cloudflare. They are world-class at what they do and the free tier graduates cleanly to paid.
- I would not move off Resend. Same story.
- I would not move off Claude for the inference layer. Cost scales linearly, quality is best-in-class.

The choices made under cost constraints are mostly the choices I would make under abundance. That is a useful signal that the constraints were not actively hurting product decisions.

## Operating notes

### Things I do daily

- Read `journalctl -u my-service.service --since "1 day ago"` over coffee
- Check the daily digest email (top errors, send counts, reply queue size) over breakfast
- Reply to any customer DM that landed via the hot-lead notification

### Things I do weekly

- `apt update && apt upgrade` on Oracle (Sunday morning, when traffic is lowest)
- Backup the `customers/` directory to a separate Cloudflare R2 bucket (free tier)
- Review the previous week's cold-outreach reply categories and tune the templates

### Things I do monthly

- Review Anthropic spend per token-category to see if I should be more aggressive with the cheap Haiku tier
- Review Resend deliverability stats — bounce rate, complaint rate
- Companies House notifications for any customers operating as Ltd, in case directorship changes

### Things I deliberately do not do

- Build my own monitoring dashboard. The hot-lead phone notification + daily digest + journalctl covers 95% of what a custom dashboard would tell me. The last 5% is not worth the maintenance.
- Write tests for one-off cron scripts. They run once a day, fail loudly via email, get fixed the next morning. Tests for them would be optimising the wrong axis.
- Optimise for performance before I have a customer complaining about it. Premature optimisation is real and shows up in solo-founder time budgets.

## License

MIT. The repo is meant as reference, not a framework. Copy what you like.

## About

Built and maintained by Sal (Abdulsalam Oladapo). Solo, UK-based. The live product this stack runs is at https://consentleads.uk.

If you are an engineer building similar infrastructure and want to compare notes, I am at sal@consentleads.uk.
