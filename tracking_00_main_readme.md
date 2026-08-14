# GTM Engagement Tracking Engine

**Author:** Adilsha S Rehman
**Stack:** Microsoft Power Automate (HTTP webhooks) · Excel Online · HTML injection into outbound mail

An experiment in building email open and click tracking from scratch, inside Power Automate, rather than relying on a marketing platform.

---

## Why I built it

Standard sequencing tools report open and click rates that are visibly inflated. Corporate security scanners — Mimecast, Proofpoint, and similar — automatically fetch images and follow links to check for malware, and every one of those registers as engagement. If a rep prioritises follow-ups based on those numbers, they are chasing firewalls.

I wanted to see whether the tracking layer itself could be built in the tooling I already had, and whether a few defensive checks could separate machine traffic from human traffic well enough to be useful.

This was a final-weeks project. It works, but it never ran at scale, and several of the thresholds below are reasoned rather than tuned against real data.

---

## Architecture

Two separate flows, deliberately not combined.

**Flow 1 — Open tracking.** A transparent 1×1 pixel embedded in the email body. When the image loads, it hits a Power Automate HTTP webhook carrying the lead ID and campaign stage as URL parameters. The flow returns the image immediately, then processes in the background.

**Flow 2 — Click tracking.** Links in the email point at a webhook rather than the destination. The flow intercepts, checks, logs, and issues an HTTP 302 redirect to the real URL.

They are split because the click path is user-facing and the open path is not. A click has someone waiting on it, so that flow has to stay fast; open tracking can take as long as it needs. Combining them would have meant the slowest branch setting the pace for both.

| Flow | Documentation |
|---|---|
| 1 | [Open Tracking — Listener Architecture](./01_Listener_Flow_Architecture.md) |
| 2 | [Click Tracking — Redirect Architecture](./02_Click_and_Device_Flow_Architecture.md) |

---

## The filtering approach

Every request is treated as automated until it passes checks.

**Temporal.** A scanner typically fetches within milliseconds of delivery. Flow 1 requires more than 60 seconds between send time and pixel load. Flow 2 injects a 1–2 second delay before responding, on the theory that impatient scanners time out and humans do not.

**State matching.** The URL parameters carry the campaign stage. If that does not match the prospect's current stage in the ledger, the request is treated as noise rather than a genuine open of the current message.

**Header analysis.** The `User-Agent` string is parsed to identify headless clients and to record device context — desktop or mobile — for the requests that pass.

These reduce false positives. They do not eliminate them, and the section below explains why.

---

## Limitations

**Scale is untested.** Built in my final weeks in the role. The logic runs correctly; it has not been exercised against production volume, and I have no data on Power Automate's HTTP trigger behaviour under concurrent load.

**Open tracking has a ceiling I did not account for.** Apple's Mail Privacy Protection pre-fetches and caches every image in every message through a proxy, regardless of whether the recipient opens it. Gmail has proxied images since 2013. Neither presents as a scanner — the timing is not instant and the User-Agent looks ordinary — so my filters do not catch them. For any recipient base with meaningful Apple Mail usage, a share of "verified" opens are the proxy rather than a person. Click tracking is the more trustworthy of the two signals for this reason.

**The redirect delay is a real cost.** The 1–2 second pause in Flow 2 that defeats impatient scanners is also perceptible to a human clicking the link. For cold outreach, where data quality matters more than polish, I would take that trade. For a nurture sequence to warm leads I would drop the delay and accept dirtier metrics rather than add friction.

**The redirect endpoint accepts an arbitrary destination.** Flow 2 takes the target URL as a parameter and forwards to it. Anyone who obtains the webhook URL could craft a link that redirects to a destination of their choosing. That is an open-redirect vulnerability, and the fix is an allowlist of permitted domains checked before the 302 is issued.

**Thresholds are guesses.** Sixty seconds and one-to-two seconds are reasonable-sounding numbers, not values derived from observing real scanner behaviour. Tuning them would need a labelled dataset I never had.

**No consent or lawful-basis layer.** IP addresses are personal data under GDPR, and pixel tracking in marketing email falls under ePrivacy rules. This was built as a technical exercise on cold outreach; deploying it against EU recipients would need a consent mechanism and legal review before anything else.

**Forwarded links carry the original ID.** If a recipient forwards the email, engagement from the new reader is attributed to the original prospect.

---

## What I took from it

The interesting part was not the tracking — it was realising how much of the reported engagement in standard tools is machine traffic, and how hard it is to filter reliably once you try. Every heuristic I added caught some cases and missed others, and the biggest source of false opens turned out to be one my filters were structurally incapable of catching.

It also made the case for splitting flows by latency requirement rather than by feature, which is a habit that carried into later work.
