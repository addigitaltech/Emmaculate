# School fees and payments

## How fees work
1. **Fee types** (Fees & payments > Fee types): Tuition, Uniform, Bus ...
2. **Assign fees to classes** (Fees & payments > Assign fees to classes): pick the fee type, the amount for each student, the session (and term), then the classes and, if you like, one arm. Every active student of those classes is charged. Students who already have that fee for that session and term are skipped, so pressing the button twice never charges anyone twice. Single students can still be charged one by one under *Fees owed*.
3. **Parents and students see the debt** in their portal: total still owing, each fee with what was charged, what is paid and the balance, and the payment history with printable receipts.

## Full payment or installments (Settings > Portal & website switches)
- **Installments allowed:** a parent can pay a fee in one go or in parts. Every part must be at least the *smallest installment* you set (or the balance, if that is less) and never more than the balance. A fee shows as *Part paid* until it is cleared.
- **Full payment only:** all of a student's unpaid fees are added together and paid in one payment.

## What happens when someone pays
1. The parent chooses Paystack or Flutterwave and pays on the provider's secure page.
2. The website never trusts the page the parent returns to: the server asks the provider whether the payment really happened and checks the amount, currency and reference. Only then is the fee marked paid.
3. A receipt is created. The parent (or the student's parents, when a student paid) gets an **email with the receipt attached**; the school gets an **email alert**; and, when SMS is set up, the school gets a **text message** (and parents too if you switch that on).
4. Each payment is announced once only, even if the provider calls twice. If an email or text fails, the payment is still recorded.
5. Receipts can be printed again at any time from the portal.

If a provider request times out or returns an inconclusive result, the transaction is retained as **Processing** and blocks another payment attempt for that student. Do not try again with a different key: the provider might have accepted the original request before the response was lost. Finance must check the provider dashboard against the displayed/local transaction reference and reconcile the attempt before a retry is authorized. A return page or a payment initiation response does not itself confirm settlement.

## Refunds and reconciliation

Refunds require an authorized finance role and an enabled provider configuration. The original payment is re-verified before a refund is requested. Paystack refund webhook events are used to update asynchronous outcomes. Flutterwave may accept a refund request before final disbursement; a merely accepted/pending response is not recorded as a completed refund. Check the provider dashboard and follow the school's documented finance reconciliation procedure before treating an outstanding Flutterwave refund as final. Never repeat an unresolved refund or manually mark a transaction as refunded solely because a request was accepted. Automated Flutterwave refund-status callbacks/reconciliation are not yet implemented.

## Payment history and debts (Fees & payments > Payment history & debts)
Pick **dates** or a **whole session** (and optionally a class or arm, or search a student). The page shows totals, every payment and everyone who still owes. **Print**, **Download PDF** or **Download spreadsheet**.

## Setting up notifications
- **Email:** make sure email works (docs/EMAIL_SETUP.md). In Portal switches, list the bursar's email(s); Finance Admin users are always included.
- **SMS (optional):** create an account at termii.com, then add to the `.env` file (or Render > Environment):
  ```
  SMS_DRIVER=termii
  TERMII_API_KEY=...
  TERMII_SENDER_ID=YourSchool
  ```
  Then list the phone numbers to text in Portal switches (Nigerian numbers such as 08031234567). Never put the key in chat, code or GitHub.

## Safety built in
Only a plain naira amount is accepted, the fee and balance come from the database (never from the browser), one payment at a time per student, amounts are capped, names sent to the provider are cleaned, provider addresses are checked, webhooks are signature-checked, and every change is in the activity log. Use provider sandbox/test keys in development and staging; tests use faked provider responses and make no live charges.
