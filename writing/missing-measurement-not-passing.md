# A missing measurement is not a passing one

*August 2026*

I defined the availability and latency objectives for a 58-repo service fleet as multi-window burn-rate alerts: a 1% error budget where a fast burn — 14.4× the budget, breaching on the 1h *and* the 5m window — pages, and a slow burn — 6×, on 6h *and* 30m — opens a ticket. Both windows must breach, so a brief spike does not wake anyone. A sanitized copy of the rules, thresholds and reasoning intact, is public in [lgtm-stack-experiment](https://github.com/hjaraujof/lgtm-stack-experiment/blob/main/config/mimir/rules/demo/slo-burn.yaml), so nothing below has to be taken on my word.

The multipliers are the boring part. They come out of the Google SRE workbook and they were correct on the first attempt. Every hour I actually spent went into the denominator.

## The denominator I would have written

The obvious availability SLI is the fraction of inbound server requests answered 5xx. Numerator: server spans with a 5xx status code. Denominator: server spans.

That version reports a permanent, authoritative-looking zero on your highest-traffic services.

Some frameworks emit `SPAN_KIND_SERVER` spans from their own tracing rather than from an HTTP instrumentation. Those spans set neither `http.response.status_code` nor `http.request.method`, and they leave span status `UNSET` even on a failed request. A server-rendered front end is the common case. Divide by all server spans and every request such a service serves lands in the denominator while nothing can ever land in the numerator, so the ratio is zero — not zero because the service is healthy, zero because the measurement cannot see it. It renders on a dashboard identically to a service with no errors.

So the denominator is restricted to spans that carry a status code at all:

```
sum by (service_name) (rate(traces_span_metrics_calls_total{
  span_kind="SPAN_KIND_SERVER", http_response_status_code=~"5.."}[1h]))
/ clamp_min(sum by (service_name) (rate(traces_span_metrics_calls_total{
  span_kind="SPAN_KIND_SERVER", http_response_status_code!=""}[1h])), 1e-9)
```

A service that emits no status codes now produces no series rather than a zero. That is the honest outcome, and it fails silently, which is why it needs its own alert:

```
(increase(... span_kind="SPAN_KIND_SERVER" ...[30m]) > 300)
  unless (increase(... http_response_status_code!="" ...[30m]) > 0)
```

More than 300 inbound requests in thirty minutes, and not one of them classifiable. It is an `unless` on a presence check rather than a ratio, so it needs no denominator and no sample gate — the question is whether any measurable request exists, not what fraction of them failed.

When I first deployed that pair it fired immediately, on the two busiest services in production. Neither had any server-side failure signal at all, and the availability dashboard had been reporting both as flawless for as long as it had existed.

## Why not span status

Three reasons, in increasing order of subtlety, and I only understood the third one after the rules were already running.

Span status over all span kinds measures instrumentation depth. One failing request marks every decorated method span it touches, so a service with deep internal instrumentation reads as catastrophically broken while its server responses are clean. A service can show a server error ratio of 0 and an internal error ratio of 0.08, and an unfiltered rule pages on the second number.

Filtering to server spans fixes that and is still not enough, because under OpenTelemetry semantic conventions a 4xx does not set server span status to `Error`. A service can answer the large majority of its traffic 401 for days and read as a perfectly healthy zero.

And status code is immune to the benign-error floor by construction. A caught-and-handled internal error sets `STATUS_CODE_ERROR` on an internal span and never produces a 5xx response, so it cannot enter a status-code numerator. Every ratio built on span status carries that floor underneath it and has no way to subtract it.

The 401 case is the one I care most about. A prolonged auth outage is a production incident that a span-status stack cannot see: common frameworks do not log 4xx by default, a caller that returns null on a 401 fails quietly, and every rule keyed on span status is untouched by a 4xx. It also must not burn the availability budget, because a service that authenticates a caller and says no is working correctly. So it gets its own SLI — 401 and 403 only, not all 4xx, because a 404 and a 422 are caller-driven and folding them in would put a permanent floor under every service in the fleet.

## The gate is the part I would carry to another fleet

Both ratio alerts need a minimum-traffic gate, because a short window with a handful of requests yields a noisy ratio and a flapping alert.

The first thing I got wrong was gating on rate. A gate of 0.2 requests per second does not stabilize a ratio: a bursty low-QPS service that straddles it still flaps, because one cold-start request collapses a near-empty window. The root cause is sample count, not rate, so the gate has to be an absolute request count inside the window.

The second thing I got wrong was more interesting. The latency alert gates at more than 300 requests per 30 minutes. That number is not a round choice: at n=300 and p of about 0.95 the ratio's standard error is roughly 1.3 percentage points, against a threshold band of 5 points — a band-to-error ratio of about 3.8. Then I wrote the auth-failure alert and reached for the same 300.

Copying it would have been actively wrong. The auth alert separates a healthy service, sitting at exactly zero, from one at 10 points or more. That is a much wider band, so it reaches the same confidence at a much lower sample count: at n=150 and p of 0.10 the standard error is 2.4 points against a 10-point band, a ratio of about 4.1. And a modestly-trafficked auth service can sit near 150 requests per 30 minutes — so a 300 gate would have excluded the exact service the alert exists to catch. Derive the gate from the band it has to resolve, not from the last alert you wrote.

The window and the gate are also coupled, which I learned by breaking it. When the latency SLI gained its `span_kind` filter, the existing 300-request gate over a 5-minute window silenced the alert entirely: with only server spans counted, no service in the fleet cleared 300 requests in five minutes. The gate exists for sample count, so the fix is to widen the window to 30 minutes rather than lower the bar to a number the noise can clear.

That `span_kind` filter is load-bearing for the same reason the denominator is. A service with deep internal instrumentation can emit several hundred internal spans per inbound request, so an unfiltered latency ratio is almost entirely internal-span noise — which masks a real regression on a well-instrumented service and fires on one serving every inbound request under the threshold. Under an unfiltered rule, adding instrumentation moves the objective with no user-visible change whatsoever.

## What is still open

The coverage tripwire is the most important rule in the file and the easiest one to silence. It fires for exactly the services you can least afford to fly blind, it will keep firing until someone fixes their instrumentation, and a silence has no technical consequence at all. I wrote the summary line to say that whoever silences it is choosing to fly those services blind, which is the only enforcement a rule file can carry.

Its obvious fix is often the wrong one. Re-enabling a generic incoming-HTTP instrumentation to recover the status code can reinstate a duplicate server span that loses `http.route`, takes the routed span name with it, and logs an "operation on ended Span" error once per request. The fix belongs on the framework span that survives, which is more work than a configuration flag and is still not done on both services.

And the 1% error budget is a placeholder. It is applied uniformly to every service and validated against no measured baseline, which is stated in the header of the file itself. The structure and the multipliers are the part I will defend; the absolute budget is what you tune per service once you have real baselines. Until that calibration happens I can tell you the objectives are defensible and the gates are derived, and I cannot tell you an attainment figure — anyone quoting one off these rules would be quoting a number nobody has earned yet.
