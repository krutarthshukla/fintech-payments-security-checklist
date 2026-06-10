# Fintech / Payments Security Testing Checklist

A working checklist of things to test on payment gateways, PSPs, banking and wallet apps, lending/BNPL platforms, and merchant payment integrations. It focuses on the money-movement and sensitive-data problems that generic web and API checklists tend to miss. Tick items off as you go, and push the severity up whenever real money moves or regulated data leaks.

## Before you start

- [ ] Get written authorization and rules of engagement, and confirm whether you're on production or a sandbox.
- [ ] Collect test credentials: API keys (both test and live), test cards, test bank accounts, test VPAs/IBANs.
- [ ] Create at least two accounts of every role you'll test: two customers, two merchants, a sub-merchant, an admin/ops user.
- [ ] Walk each money flow end to end through a proxy before testing it: checkout, transfer, payout, refund, settlement, KYC.
- [ ] Note where real money actually moves, and never run double-spend or over-withdrawal tests on real funds without sign-off.

## Authentication, API keys and sessions

- [ ] Hunt for API keys and secrets in URLs, query strings, frontend JS, mobile app bundles and git history.
- [ ] Confirm a test or restricted key can't run live captures, payouts, or read another merchant's data.
- [ ] Confirm rotating or revoking a key invalidates it immediately.
- [ ] If requests are signed, tamper with a signed field, remove the signature, and send an empty or invalid one. If the signing formula looks guessable, try to brute-force the secret and look for default secrets in source and backups.
- [ ] For token-based auth, check the token's scope is enforced on the server (a read-only token must not move money) and try the usual JWT tricks: none algorithm, key confusion, tampered claims.
- [ ] Test session handling on dashboards: fixation, idle and absolute timeout, concurrent sessions, server-side logout.
- [ ] Try to skip, replay or downgrade step-up auth (OTP, transaction PIN, card step-up, biometric, push approval). Reuse an old OTP for a new transaction, or use one user's challenge to approve another user's action.
- [ ] Confirm the step-up is tied to the exact amount and payee, so changing either after approval invalidates it.
- [ ] Confirm login and OTP endpoints are rate-limited, don't reveal whether a user exists, and never leak the OTP in the response, headers, logs or push payload.
- [ ] Try to escalate from customer to merchant to admin by changing a parameter, a role value, or by browsing straight to admin endpoints.

## Authorization and tenant isolation

- [ ] Treat every ID in every request as an access-control test: payment, order, transaction, refund, payout, customer, account/wallet, mandate, settlement, invoice, dispute, statement, KYC document, beneficiary.
- [ ] Replay each read request as a different user or merchant and see if you get someone else's data.
- [ ] Replay each write request as another tenant: refund, capture, cancel, payout, or edit another tenant's payment or beneficiary.
- [ ] Check whether IDs are sequential or guessable, and whether you can walk through other records by incrementing them. Random IDs still need an ownership check.
- [ ] On create and update requests, add fields you shouldn't control (merchant_id, customer_id, account_id, status, fee, is_settled, role, kyc_verified, credit_limit) and see if the server accepts them.
- [ ] Read the raw API responses for fields the UI hides: full card number, other people's personal data, internal flags, ledger internals.
- [ ] Run admin or ops-only actions as a lower-privileged user (force-capture, manual settle, balance adjustment, refund override, fee waiver), including by changing the HTTP method or hitting the admin route directly.
- [ ] Check marketplace isolation: a sub-merchant must not act on the platform account or on sibling sub-merchants.
- [ ] Confirm exports and statements (CSV/PDF) are scoped to the caller's own data.
- [ ] On GraphQL or batch endpoints, try nested traversal and aliasing to reach objects or brute-force IDs the per-request check would otherwise block.
- [ ] Try a transfer where the source and destination are the same account.
- [ ] Use a sandbox or test-mode object ID (and a test key) against the live environment, and the reverse, to see if the two modes are properly isolated.

## Amount, currency and field tampering

- [ ] Lower the amount, set it to zero, negative, a tiny decimal (0.001), a huge number, scientific notation (1e2), or send it as a string instead of a number.
- [ ] Confirm the server recalculates the payable amount from the cart and ignores any client-supplied price or total. Try adding a price field to an add-to-cart or order request.
- [ ] Pay in a weaker currency for the same numeric amount, mismatch the order and payment currency, or exploit cents-vs-whole-units confusion.
- [ ] Tamper with fee, tax, shipping, surcharge and convenience-fee fields; try a negative discount or negative shipping; stack coupons and reuse a coupon past its limit.
- [ ] On cross-currency payments, tamper with the exchange rate or FX markup the client sends, and confirm the server sets it from its own source.
- [ ] Abuse quantity: duplicate the parameter, send a decimal or negative quantity, or remove items until the quantity goes negative.
- [ ] In multi-line payouts or transfers, slip a negative line into a positive total and see if each line still executes.
- [ ] Change the payee account, VPA or IBAN after the payment has been authorized or OTP-approved.
- [ ] Change the merchant's settlement/payout bank account to one you control, and see whether it goes through without re-verification.
- [ ] Try to beat per-transaction and per-day limits by splitting the amount, switching currency, or resetting the counter with a new session or device.
- [ ] For volatile assets, hold a buy/sell request and release it only after the price moves in your favor, and check the server re-prices at execution.
- [ ] Try to pay an amount smaller than the processing fee so the merchant loses money on the transaction.
- [ ] Tamper with rewards, cashback and points: redeem more than earned, set a negative redemption, inflate the accrual rate.
- [ ] On loans, EMI and BNPL, tamper with principal, tenure, interest rate and installment count.

## Payment state machine

- [ ] Map the legal states (created, authorized, captured, settled, refunded, voided, failed, disputed) and then try every illegal transition between them.
- [ ] Capture without an authorization, or capture more than was authorized.
- [ ] Capture the same authorization more than once, or split one authorization into captures that together exceed it.
- [ ] Capture against an authorization or hold that has already expired.
- [ ] Refund a payment that was never captured, already refunded, or failed; refund more than was captured; stack partial refunds past the total.
- [ ] Refund to a different card or bank account than the one that paid, or as wallet credit you can then cash out.
- [ ] Void or cancel after capture or settlement; re-settle an already-settled transaction.
- [ ] Mark an order paid by hitting the internal status endpoint directly, or by browsing to the success/callback page without paying.
- [ ] Replay a successful authorization or capture to charge again.
- [ ] Force the payment to fail but still claim the goods, and the reverse (server thinks failed, client thinks success).
- [ ] Change the cart, address or discount after the amount is locked, and add items after payment to see if they're treated as paid.
- [ ] On disputes and chargebacks, try to get a refund and keep the goods, or recover twice (refund plus chargeback).

## Idempotency, races and double-spend

- [ ] Resend the same request with the same idempotency key (expect exactly one effect), then with a different key for the same logical action (watch for a double charge or credit).
- [ ] Try omitting the idempotency key, reusing another request's key, and predicting keys.
- [ ] Fire many identical requests at the same instant against: wallet spend, refunds, coupon and gift-card redemption, withdrawals and payouts, loan disbursal, OTP/token verification, and one-time bonus credits.
- [ ] Try to push a balance negative with simultaneous debits.
- [ ] Confirm held or reserved funds are released exactly once, never twice and never left stuck.
- [ ] After forcing a race, pull the statements and confirm the ledger still balances.

## Webhooks, callbacks, redirects and SDKs

- [ ] Forge an incoming "payment successful" notification and see if the order ships. Confirm the receiver verifies the signature over the raw body with the right secret before trusting it.
- [ ] Confirm the timestamp is part of what's signed, not checked separately; otherwise replay an old event with the timestamp bumped to now.
- [ ] Replay a genuine signed notification and look for duplicate fulfilment; confirm there's a short validity window and event de-duplication.
- [ ] Send events out of order or drop one (a capture before its authorization, a refund before the capture) and see how the receiver reconciles state.
- [ ] See whether any host can deliver a notification, or whether the source is restricted.
- [ ] Tamper with status, amount, transaction-id and signature in the return/redirect URL and see if the order is marked paid from it.
- [ ] Test the return URL for open redirect, and the webhook/callback URL registration for server-side request forgery by pointing it at internal addresses.
- [ ] On embedded card fields or iframes, try to read the card number or CVV from the parent page, via cross-window messaging with a spoofed origin, or by clickjacking the iframe.
- [ ] Inspect client SDKs for leaked live keys, internal endpoints or secrets.
- [ ] Check the payment page for unexpected third-party scripts and weak script-integrity controls.

## Sensitive data and card handling

- [ ] Confirm the full card number is masked everywhere (UI, API responses, logs, errors, analytics) and never returned after tokenization.
- [ ] Confirm the CVV is never stored anywhere: at rest, in logs, in cache, in queues, or in a response.
- [ ] Confirm no full track data, PIN blocks or chip data are stored.
- [ ] Confirm a token can't be reversed back to a card number and can't be reused at another merchant.
- [ ] Check bank account numbers, IBANs, government IDs and KYC documents are access-controlled and masked, and not sitting on a guessable storage URL.
- [ ] Look for card data, keys, tokens or personal data in logs, error pages and stack traces, and confirm verbose errors are off in production.
- [ ] Look for exposed debug and admin endpoints, config files, API docs, schema introspection, source maps and backups.
- [ ] Confirm TLS is enforced with no downgrade, and that no sensitive data travels in the URL or query string.
- [ ] Confirm sensitive responses aren't cached and don't survive in browser history.
- [ ] Check data-residency rules are honored where they apply.

## Rail-specific tests

Cards and 3-D Secure:
- [ ] Skip the card step-up by replaying a frictionless result or dropping the challenge value.
- [ ] Tamper with the step-up callback values, and try to downgrade to a weaker version or to no step-up at all.
- [ ] Abuse card-on-file: charge a saved card without a valid mandate.
- [ ] Bypass test-card detection to push test cards into the live flow.

UPI:
- [ ] Abuse collect requests to trick a user into approving a payment, and check payer/payee names can't be spoofed.
- [ ] Check a VPA lookup doesn't leak the registered name to anyone who asks.
- [ ] Test mandate/autopay creation and edits for authorization and amount caps.
- [ ] Tamper with the amount and payee in the UPI deep link or intent.

Bank transfer (ACH/SEPA):
- [ ] Change the beneficiary after authorization, and brute-force or race the micro-deposit verification.
- [ ] Check returns and reversals can't be abused to keep money after a clawback.

Open banking:
- [ ] Confirm strong customer authentication is enforced and bound to the amount and payee, and that exemptions can't be forced.
- [ ] Confirm a read-only consent can't initiate a payment, and that consent can't be reused after expiry or revocation.
- [ ] Test access across consents and providers: one provider reading another's data, or a guessable consent ID.
- [ ] Confirm the third party's certificate is actually bound to its identity, not just terminating TLS.

Wallets:
- [ ] Race top-ups and withdrawals, and check inter-wallet transfer authorization, negative balances and expired-balance reuse.

Lending and BNPL:
- [ ] Race credit-limit checks to disburse past the limit.
- [ ] Tamper with repayment amounts and foreclosure/skip-EMI logic, and confirm interest is recalculated on the server.

## KYC, onboarding and account takeover

- [ ] Check document upload validates type, size and real content, and that a low-KYC account can't move high-value money.
- [ ] Try to reach an active, money-moving account while skipping verification steps.
- [ ] Try to read another user's uploaded KYC documents.
- [ ] Change the email or phone without re-authentication, then move money out: test the whole takeover-to-payout chain.
- [ ] Check there's a cooldown or limit when adding a new payee.
- [ ] Try to bypass device binding or new-device verification.

## Mobile app (payment-specific)

- [ ] Bypass certificate pinning, then run all the API tests above over the app's traffic.
- [ ] Check a local biometric or PIN unlock isn't trusted on its own to authorize a transaction.
- [ ] Tamper with amount, payee and redirect values in the app's deep links.
- [ ] Look for the card number or OTP being copied to the clipboard, shown in notifications, or visible in the app-switcher snapshot.
- [ ] Decompile the app and look for hard-coded keys, signing secrets or credentials.

## Severity notes

- Unauthorized money movement to the attacker is critical, and needs a working proof of concept plus a real balance or ledger change.
- Reproducible double-charge, double-spend or negative balance is critical to high.
- Reading or modifying another tenant's transactions or KYC is high, and rises with the number of records exposed.
- Stored CVV, leaked full card numbers or track data is high and a compliance failure.
- Webhook or return-URL forgery that yields free goods, accepted amount or currency tampering, and step-up bypasses are high.
- Read-only data leaks are medium to high depending on the sensitivity of the data.
- Don't rate anything above medium without an actual exploit artifact.

## References

- OWASP Web Security Testing Guide — Testing for the Payment Functionality
- OWASP API Security Top 10 (2023)
- OWASP Application Security Verification Standard (ASVS)
- OWASP Mobile Application Security (MASVS / MASTG)
- PCI DSS v4.0.1
- PortSwigger Web Security Academy — Race Conditions
