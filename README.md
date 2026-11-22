# NYOTA starter-loan integration (deployment checklist)

Files: `app.py` is your original Flask service plus secure loan routes; `loan_routes.py` is the new API; `loan-result.html` is the result page; `eligibility.html` redirects to the result page without pretending approval.

## Critical prerequisites

1. Install `firebase-admin` along with your existing Flask, flask-cors and requests dependencies. Set `FIREBASE_SERVICE_ACCOUNT_JSON` to the Firebase service account JSON in Render (or use configured application default credentials). Do not expose it in HTML.
2. Configure **Firebase Authentication phone sign-in** in your website. After Firebase authenticates the user, provide `window.nyotaAuthToken = async () => firebase.auth().currentUser.getIdToken()` (compat SDK) or the equivalent modular SDK implementation. This function must be defined before `loan-result.html` requests the offer. Your current localStorage-only signup is **not authentication** and must be replaced for real payouts.
3. Use a **trusted underwriting / identity-verification service** to write `nyota_eligibility_decisions/<firebase-auth-uid>` in Firebase Realtime Database, e.g. `{"status":"approved","phone":"254712345678","starter_amount":50,"potential_limit":25000}`. The phone must match the verified Firebase Auth phone. **Never allow clients to write these approval records**. This package deliberately does not auto-approve based on self-declared income.
4. Configure HashBack wallet funds, test that the provider accepts a KES 50 payout and check actual provider fees. The starter-loan route calls HashBack directly; it does not use the Aviator balance and does not depend on `MIN_WITHDRAW`.
5. A HashBack `success:true` reply is treated as **accepted/pending**, not final M-Pesa delivery. To show “KES 50 sent”, a trusted reconciliation or authenticated provider callback must set `nyota_starter_loans/<uid>/status` to `CONFIRMED_DELIVERED` after verifying settlement. The supplied `/callback` endpoint merely logs payloads and must NOT be used to confirm payment until its signature and transaction fields are verified.
6. Publish the website and API on the same origin or configure restricted CORS for your own domain. Ensure all endpoints are HTTPS. Set up rate limiting, fraud monitoring, audit logging, reconciliation, and a customer support path.
7. Loan disclosures must include actual APR/fees, repayment schedule, lender identity, privacy terms, consent, and any applicable Kenyan lending compliance. The KES 25,000 figure is a potential limit only unless separately approved.

## API behavior

- `GET /api/loan/offer` verifies Firebase ID token and checks trusted approval.
- `POST /api/loan/initial-disbursement` atomically claims a one-time starter payout and calls HashBack for KES 50. A retry returns the original reference rather than paying again.
- `GET /api/loan/disbursement-status?reference=...` returns `COMPLETED` only after verified settlement is recorded; otherwise `PENDING` or `FAILED`.

**Important:** Unclear network outcomes are held for reconciliation, not automatically retried, to avoid double payouts. A definitive provider rejection is recorded as FAILED but the claim remains locked until a trusted operator/reconciliation process decides how to handle it. This is intentionally conservative.
