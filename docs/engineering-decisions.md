# Engineering decisions

These are the design choices that shaped the charging gateway and the renewal engine, and the trade-offs behind each one.

## 1. One charging API, pluggable billing back ends

Every operator charges differently: IN over UCIP/AIR, an SDP over SOAP, or a custom HTTP API. Every payment provider differs again.

- The CGW exposes **one request format** to every client. Behind it, a **billing-interface factory** picks the implementation per billing node: a socket connector, a webservice client, or a dummy.
- **Why:** onboarding a new operator or market means configuring a new node and, at most, writing one adapter. The rating, limits, logging, and every client stay untouched.
- **Trade-off:** the common API covers what all back ends have in common. Operator-specific features travel as extra parameters instead of first-class fields.

## 2. Rating split by event type

Separate rate engines handle **specific events** (one-off charges) and **session start, interim, and stop**. A factory picks one from the charging type and the wallet option.

- **Why:** session charging needs state (what was reserved at start, what was used at each interim) that one-off charges don't. Keeping them apart keeps each engine simple and independently testable.

## 3. Priority-ordered rate contexts

One request can match several rate plans, such as a promotion, a time-of-day price, or the default package price.

- All matching contexts are loaded and **sorted by priority**. The first one that applies sets the rate ID, the billing nodes, and the rating info.
- **Why:** pricing changes (a promotion for a weekend, a new time slot) become configuration rather than code, and the precedence is explicit.

## 4. Price limits as capture and revert

Each subscriber has **monthly and yearly spending caps**, which reset on calendar boundaries and can optionally carry forward unused monthly allowance. Before charging, the CGW **captures** the amount against the cap. If the charge fails, it **reverts** the capture.

- **Why:** this works like a card authorization. The cap is checked and reserved before any money moves, and failed charges never use up the allowance. It protects subscribers from runaway charging and helps meet regulators' spending rules.

## 5. Throughput limits per application and time slot

TPS limits are configured **per application per time slot** and checked before rating.

- **Why:** operator charging systems have contracted capacity, and some hours are sensitive. One noisy application can't crowd out the others or overload the operator.

## 6. A compact TCP protocol for internal clients

Internal clients like the renewal engine talk to the CGW over **persistent TCP connections**, using a compact delimited request and a short space-separated response (session ID, result code, and so on).

- **Why:** renewals run in bulk, and the gateway handled about 500 TPS at peak. Reusing long-lived connections and keeping parsing trivial kept per-request overhead low. External partners go through the webservices instead.

## 7. Renewal fairness and pacing

- **Several picker threads** each fill their own queue from the database in bounded batches. A **round-robin global queue** hands work to requester threads.
- Each requester has a **per-second TPS cap**, and requesters run only inside **allowed time windows**.
- **Why:** at about 200,000 renewals a day, volume spikes at known times, such as the start of a billing cycle. Pacing keeps the charging rate inside the operator's limits. Round-robin over several queues stops one slow slice of the subscriber base from blocking the rest.

## 8. Renewal as an explicit state machine

Each charge result maps to one state: **Registered**, **InGracePeriod**, **RenewalFailed**, or **Deregistered**.

- The next state comes from the subscription's next renewal date plus the package's **grace period** and **retry window**, compared with the next retry time.
- **Downgrade before failing:** if a package allows it, a failed charge is retried at once on a cheaper package, which keeps the subscriber.
- An optional **eligibility check** runs before charging, so requests that are bound to fail never reach the operator.
- **Why:** charging rules differ between operators and markets. Expressing them as per-package configuration (grace days, retry period, downgrade target) let one codebase serve all of them.

## 9. Crash-safe in-flight tracking

Before charging, each requester thread writes the subscription ID to its own **in-flight file**, and clears it afterwards.

- **Why:** if the process dies mid-charge, the IDs that were in flight are known exactly. They can be reconciled against the CDRs instead of being charged again blindly, which matters when the action is taking money.
- Shutdown is **graceful**: removing a command file tells the threads to finish their current request and stop.

## 10. Fakes and dummy billing nodes

Both services can run against **fake** back ends: a dummy billing interface in the CGW, and a fake socket and webservice in the renewal engine.

- **Why:** you can't test charging against a live operator balance. Fakes let the full flow, including retries, downgrades, and state transitions, run in development and staging with scripted result codes.

## 11. Keeping a long-lived platform secure

- The platform was moved from Log4j 1.x to **Log4j2**, and later pinned to patched releases after **Log4Shell**. The build also excludes the old logging artifacts, so they can't come back through transitive dependencies.
- The database driver was upgraded to the **MySQL 8** connector to follow the database upgrade.
