# GTM Engagement Tracking Engine

## Overview
A custom, serverless email engagement tracking system built natively within Microsoft Power Automate. This architecture bypasses standard third-party marketing plugins (which are often plagued by false-positive bot clicks) to provide a pristine, highly defensive analytics engine for outbound sales campaigns. 

By utilizing HTTP webhooks, the system acts as a "Man-in-the-Middle" routing engine. It intercepts engagement data, runs temporal and state-matching security checks to filter out automated corporate firewalls, logs verified human activity into a central database, and seamlessly routes the user without latency.

## Why Custom Infrastructure?
Standard email sequencing tools often report inflated open and click rates because corporate security scanners (like Mimecast or Proofpoint) automatically "click" links and download pixels to check for malware. 

This engine solves that data integrity problem through **Defensive Data Validation**:
*   **Temporal Delay Checks:** Forcing interactions to pass time-delay thresholds to weed out instant machine pre-fetches.
*   **State-Matching:** Ensuring the URL parameters match the exact CRM stage of the prospect.
*   **Header Analysis:** Parsing `User-Agent` strings to drop headless bots and log exact device context (Desktop vs. Mobile).

## Repository Structure

This architecture is deliberately split into two separate flows. Combining them would create a heavy, slow web of logic that degrades the end-user experience. By splitting them, we ensure the "Click" routing remains lightning-fast, while the "Open" tracking processes quietly in the background.

*   [**Flow 1: The Listener Architecture (Open Tracking)**](./01_Listener_Flow_Architecture.md)
    *   *Trigger:* Invisible 1x1 HTML image pixel.
    *   *Response:* `200 OK` (Silent processing).
    *   *Function:* Captures genuine screen time and filters out preview panes.
*   [**Flow 2: Click and Device Architecture (Link Tracking)**](./02_Click_and_Device_Flow_Architecture.md)
    *   *Trigger:* Encoded hyperlink (webpages or hosted files).
    *   *Response:* `HTTP 302 Redirect` (Instant routing).
    *   *Function:* Captures high-intent clicks and device context before forwarding to the final destination.

## Integration & Tech Stack
*   **Routing Engine:** Microsoft Power Automate (HTTP Webhooks & REST API responses)
*   **Database Engine:** Microsoft Excel (Online) / Any structured CRM store
*   **Injection Method:** HTML tagging (`<img src="...">` and `<a href="...">`) injected dynamically into outbound mailing flows.

## Campaign Impact
Implementing this architecture yields **Zero False Positives**. When a lead is marked as "Verified" or "Clicked," it guarantees actual human engagement. This allows for hyper-targeted sales follow-ups, pristine A/B testing data, and the ability to optimize landing pages based on verified mobile vs. desktop usage trends.
