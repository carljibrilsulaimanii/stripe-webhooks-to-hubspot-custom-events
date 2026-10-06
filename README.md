<!--
  README.md -- Stripe webhooks into HubSpot workflow webhook events
  Author:  Jibril Sulaiman
  Date:    2026-10-06
  What:    Click-by-click guide for sending any Stripe event (payment_intent.succeeded,
           checkout.session.completed, ...) straight into a HubSpot workflow through the
           "Webhook event is received" trigger, with no middleware.
  Why:     HubSpot's Stripe sync is live-only and mirrors PaymentIntents only, so test
           payments, $0 checkouts and every other Stripe event never reach a workflow.
           The webhook trigger can take them, but a Stripe payload breaks HubSpot's
           50-property limit, and the error is easy to misread.
-->

# Stripe webhooks → HubSpot workflow webhook events

Send any Stripe event straight into a HubSpot workflow, with no middleware: no Zapier,
no relay server, no code between the two. Stripe posts the event to a URL HubSpot gives
you, HubSpot turns the JSON into **webhook event** properties, and a workflow runs on
each one. Branches and custom code actions can then read fields like
`data.object.amount_total` directly.

The setup takes about 30 minutes. The hard part isn't connecting the two; it's getting
a Stripe payload under HubSpot's limits so HubSpot will create the event at all. This
guide covers both of HubSpot's setup wizards, the traps in each, and how to tell the
real error from the one on screen.

**Worked example:** the
[hubspot-stripe-zero-dollar-checkout-sync](https://github.com/carljibrilsulaimanii/hubspot-stripe-zero-dollar-checkout-sync) is built on the
`checkout.session.completed` event set up here. It has the custom code that runs on the
event; this repo covers the trigger.

## Why it exists

HubSpot's native Stripe sync covers less than it looks:

1. **It mirrors PaymentIntents, and only live ones.** Test-mode payments never arrive,
   so you can't test a payment workflow without spending real money.
2. **A $0 checkout never arrives.** A 100%-off promotion code or a $0 price creates no
   PaymentIntent and no charge, so the sync has nothing to copy.
3. **Every other Stripe event is missing.** Checkout sessions, refunds, disputes,
   subscription changes: none of them can start a workflow.

HubSpot can receive them anyway. A workflow trigger called **Webhook event is
received** gives you a URL that accepts a POST from any app, Stripe included. HubSpot
flattens the JSON into one property per path (`data.object.id` becomes the property
`data_object_id`), and the workflow can branch on those properties and pass them to
custom code. No middleware is needed.

What makes it hard:

- **A webhook event can have at most 50 properties.** A Stripe payload flattens to far
  more: a `checkout.session.completed` started at 66 in production, and a
  `payment_intent.succeeded` was over the limit too. You have to delete properties
  before HubSpot will create the event.
- **In the older wizard, going over the limit shows only a bare error:** *"Request for
  https://app.hubspot.com/api/portal-event-ingest/v1/webhook-definitions failed with
  status 400."* The reason is only visible in the browser's developer tools. In
  production the first guess (a special character in the event name) was wrong and cost
  time.
- **Two different wizards** create webhook events, with different screens and
  different rules (Step 1).

## How it works

```text
 Stripe event destination ──POST──► HubSpot webhook URL
   (checkout.session.completed,       │ flattens JSON: data.object.amount_total
    payment_intent.succeeded, ...)    │                → data_object_amount_total
                                      ▼
                         webhook event (≤ 50 properties you kept)
                                      ▼
                     workflow: branch on properties → custom code
```

## What's in this repo

| Path | What it is |
|---|---|
| [`README.md`](README.md) | This guide. The whole build is done in Stripe's and HubSpot's screens; there's no code to install. |

The custom code that runs on these events lives in the repos that use them (see
[Related repos](#related-repos-stripe-beyond-hubspot-commerce)).

## Table of contents

- [1. Requirements](#1-requirements)
- [2. Setup, step by step](#2-setup-step-by-step)
  - [Step 1: Pick connected or unconnected](#step-1-pick-connected-or-unconnected)
  - [Step 2: Start the workflow and copy its webhook URL](#step-2-start-the-workflow-and-copy-its-webhook-url)
  - [Step 3: Create the Stripe event destination](#step-3-create-the-stripe-event-destination)
  - [Step 4: Send one real test event](#step-4-send-one-real-test-event)
  - [Step 5: Cut the properties under 50](#step-5-cut-the-properties-under-50)
  - [Step 6: Link or match, then create the event](#step-6-link-or-match-then-create-the-event)
  - [Step 7: Use the properties in the workflow](#step-7-use-the-properties-in-the-workflow)
  - [Step 8: Test end to end](#step-8-test-end-to-end)
- [Which properties to keep](#which-properties-to-keep)
- [Troubleshooting](#troubleshooting)
- [Limits](#limits)
- [Security](#security)
- [Related repos: Stripe beyond HubSpot Commerce](#related-repos-stripe-beyond-hubspot-commerce)

## 1. Requirements

| Need | Why |
|---|---|
| HubSpot with workflows that offer the **Webhook event is received** trigger (Operations Hub / Data Hub Professional or above at the time of writing; check your plan) | The trigger this is built on |
| Custom code actions (same tier) if you want code to run on the event | Optional; branches alone work without it |
| Stripe dashboard access that can create event destinations (Developers / Workbench) | The sending side |
| A way to make one real event of the type you want (a sandbox, or a real checkout) | HubSpot needs a sample before it can build the properties |

## 2. Setup, step by step

### Step 1: Pick connected or unconnected

About 2 minutes. Decide this first, because **you can't change it after the event is
created** (HubSpot says so: *"After the event is created, its CRM link setting can't be
changed."*).

| | Unconnected | Connected (linked to contacts) |
|---|---|---|
| Who it enrolls | Nothing. The workflow runs on the event itself. | The contact whose property matches a field in the payload |
| Payloads with no matching contact | Still run | **Never enroll.** The match runs before any action, so no step can create the contact first. |
| Can use it in | Workflows only (*"These events can only be used in workflows."*) | Workflows, plus CRM data, personalization and reporting |
| Use it when | The event is about a payment or a session, not a person you already have | Every event belongs to a contact who already exists |

For Stripe, **unconnected is usually right**: a buyer at checkout often isn't a
contact yet. Both examples in production ended up unconnected.

You'll meet one of two wizards, depending on where you start the workflow:

| Wizard | Looks like | Its rule |
|---|---|---|
| **"Create new webhook event"** pop-up | One dialog: **Webhook event name**, **Description**, **Connect webhook**, then **Next: Event properties** → **Next: Link to object** | Offers **No, keep this event unconnected to CRM records** |
| **"Create a webhook event"** full page | Four steps across the top: **Name · Connect · Map · Match** (*"Step 3 of 4"*) | The last step, **Match your enrollment property**, is required: the workflow is contact-based |

> ⚠️ **If you need unconnected and you're in the four-step wizard, start somewhere
> else.** In production the first build went through the four-step wizard, had to match
> on email, and was rebuilt as an unconnected event from the pop-up. The workflow
> builder you start from decides which wizard you get (wording may differ).

✅ **Check:** you know which kind you need and which wizard you're in.

### Step 2: Start the workflow and copy its webhook URL

About 3 minutes.

**2a.** Create a workflow from scratch (name it after the Stripe event, for example
`Stripe checkout.session.completed`).

**2b.** For the trigger, choose **Webhook event is received**. Under the event picker
(*"Choose event or create a new one"*), click **Create new webhook event**.

**2c. Name it.** In **Webhook event name**, use the Stripe event and its purpose, for
example `Stripe Checkout Session Completed`. **Description** is optional (*"Add a
description to help Breeze understand this webhook event"*).

**2d. Copy the URL.** Under **Connect webhook** (*"Paste this URL into the webhook
settings page of the app you want to integrate with..."*), click **Copy**. It looks
like `https://api.hubapi.com/automation/v4/webhook-triggers/<portal>/<id>`.

> ⚠️ **Treat the URL like a password.** Anyone who has it can post fake events into
> your workflow (see [Security](#security)). Don't screenshot it or paste it in a chat.

Leave this dialog open. It's waiting for a test event.

✅ **Check:** the URL is on your clipboard and the HubSpot dialog is still open.

### Step 3: Create the Stripe event destination

About 5 minutes.

**3a.** In Stripe, open **Workbench → Webhooks** (in a sandbox, the banner reads *"You
are testing in a sandbox. No real transactions will be processed."*). Click **Create an
event destination**. The steps down the left are **Select events · Choose destination
type · Configure your destination**.

**3b. Select events.**
1. **Event destination scope:** **Your account** (*"Events from resources in your
   account"*), not **Connected accounts**.
2. **API version:** Stripe defaults to your account's version. Note it; the payload
   shape follows it (production accounts showed `2017-08-15`).
3. **Events:** in **Find event by name or description**, type the event, for example
   `checkout.session.completed` or `payment_intent.succeeded`, and tick it. **Selected
   events** should show **1**.
4. Click **Continue**.

> ⚠️ **One event per destination.** Each HubSpot webhook event is built from one
> payload shape. Send a different Stripe event to it and most properties come in empty.
> Make a separate destination and webhook event for each Stripe event type.

**3c. Choose destination type:** **Webhook endpoint**. Click **Continue**.

**3d. Configure your destination.** Paste the HubSpot URL from Step 2d as the endpoint
URL. Give it a name you'll recognize (for example `Comp Ticket Sync - HubSpot`) and
create it.

**3e.** The destination page shows the name with **Active**, the URL, and
**Destination details**: **API version** and **Listening to** *1 event*. The **...**
menu next to **Edit destination** has **Disable**, **Roll secret** and **Delete**.

> ⚠️ **Ignore the signing secret.** Stripe signs every delivery, but HubSpot's webhook
> trigger can't check a signature. Rolling it changes nothing on the HubSpot side.

✅ **Check:** the destination is **Active** and listening to exactly 1 event.

### Step 4: Send one real test event

About 5 minutes. HubSpot builds the property list from the first event it receives, so
it needs a real one.

**4a. In a sandbox or test mode:** trigger it with the Stripe CLI, or with **Stripe
Shell** (in the **Developers** bar at the bottom of the dashboard):

```text
stripe trigger payment_intent.succeeded
```

**4b. In live mode:** `stripe trigger` refuses: *"stripe trigger is disabled in live
mode. Switch to testmode to run stripe trigger."* Make one real event instead. For
`checkout.session.completed`, complete a checkout yourself; a 100%-off promotion code
makes it free.

> ⚠️ **Make the sample look like the events you care about.** HubSpot only offers the
> fields present in the sample. A paid checkout and a free one carry different fields
> (`payment_intent` is set on one and `null` on the other). If you'll branch on a
> field, it must be in the sample.

**4c.** Back in HubSpot:
- **Pop-up wizard:** a green bar reads *"Event received at <date, time>"*, with the
  payload below it (`id`, `type`, `object`, `created`, `livemode`, `api_version`,
  `data` → `object` → ...). Click **Next: Event properties**.
- **Four-step wizard:** **Review your test event** (*"Please review the test event and
  confirm it is how you want."*), with **Expand all** and **Retry a new test event**.
  Click **Next**.

> ⚠️ **The sample shows real customer data.** A live checkout's sample includes the
> buyer's email, address and your success URL with its UTMs. Don't screenshot it.

✅ **Check:** HubSpot shows the payload, and `type` is the event you selected.

### Step 5: Cut the properties under 50

About 10 minutes. This is the step that fails.

**5a. Pop-up wizard: "Edit event properties".** A table with **PROPERTY NAME**,
**INTERNAL NAME**, **SAMPLE VALUE** and **FIELD TYPE**, a trash icon on each row, and
**+ Add custom property** at the top. Every flattened path is a row:
`data.object.id` → internal name `data_object_id`.

**5b. Four-step wizard: "Map the data for your webhook's properties".** One card per
property, each with **Third-party property label**, **HubSpot property label**
(pre-filled with the path, like `['data']['object']['id']`) and **Data type**. Over
the limit, a red banner reads *"The limit of 50 properties has been reached. Reduce the
number of properties returned by the webhook to continue."*

**5c. Delete everything you won't use.** Keep the short list in
[Which properties to keep](#which-properties-to-keep) plus anything your branches and
code need. Delete the rest with the trash icon. The limit is on the event definition,
not on what Stripe sends: Stripe keeps posting the full payload, and HubSpot fills only
the properties you kept.

**5d. Fix the type of every row you keep.**

| Field | Set it to | Why |
|---|---|---|
| Amounts (`amount`, `amount_total`, `amount_subtotal`) | **Number** | A branch compares Number values. Left as text, `amount_total` branched wrong in production and paid checkouts went down the $0 path. |
| `livemode`, `has_more` and other true/false fields | **Boolean** | |
| `created` and other Unix times | **Number** | They're seconds, not dates |
| `api_version` | **Delete it** | The pop-up typed `2017-08-15` as **Date**, which blocked **Next** with *"Fix any property errors before continuing."* You don't need it. |
| Everything else | **String** | |

> ⚠️ **In the four-step wizard, Data type starts empty on every card.** Each one you
> keep needs a type, or you'll see *"There are errors or missing values in the
> properties below. Please fix them before continuing."*

> ⚠️ **Labels are capped at 50 characters too.** Deep paths fail on their own:
> `['data']['object']['discounts'][0]['promotion_code']` gives *"A property label can't
> be more than 50 characters."* Shorten the HubSpot label, or delete the row.

> ⚠️ **"Fix any property errors" passing doesn't mean you're under 50.** It's a
> separate check. In the pop-up wizard you only find out at the last step (Step 6).

✅ **Check:** fewer than 50 rows, every row has a type, no red or yellow banner.

### Step 6: Link or match, then create the event

About 2 minutes.

**6a. Pop-up wizard.** **Next: Link to object** opens *"Should this event be connected
to a CRM object?"*. Choose **No, keep this event unconnected to CRM records** (or
**Yes, link this event to CRM records** if you chose connected in Step 1). Click
**Create new event**.

**6b. Four-step wizard.** **Match your enrollment property**: *"Choose a property from
your incoming third-party webhook that's an exact match for one of the unique HubSpot
properties."* Set **Third-party property label** to the buyer's email and **HubSpot
property label** to **Email**.

> ⚠️ **Match on `customer_details.email`, not `customer_email`.** On a Checkout
> Session, `['data']['object']['customer_details']['email']` is what the buyer typed.
> `customer_email` is only the prefilled value, and it was `null` on the real
> checkout used in production.

**6c. If Create fails with the bare 400** (*"Request for
https://app.hubspot.com/api/portal-event-ingest/v1/webhook-definitions failed with
status 400."*), find the real reason:
1. Open the browser's developer tools (F12) → **Network**.
2. Click **Create new event** again.
3. Click the red `webhook-definitions` request → **Response**.
4. If it says `WebhookIngestDefinitionValidationError.EVENT_PROPERTIES_LIMIT` /
   *"Webhook Definition should not have more than 50 properties"*, go back to Step 5
   and delete more rows.

✅ **Check:** the dialog closes and the workflow trigger shows your event by name. In
the event picker it reads *"This event isn't linked to a HubSpot object."* (unconnected)
or *"This event is linked to HubSpot Contacts."* (connected).

### Step 7: Use the properties in the workflow

About 5 minutes.

**7a. Branch on them.** Add a branch and filter on the event's properties, for example
`data.object.amount_total` **is equal to** `0`, or `data.object.livemode` is true.

> ⚠️ **Check which side "None met" lands on.** In production the free path was wired to
> the branch's *"None met"* side, so every paid checkout fell through to it. Put the
> path you want behind an explicit condition and send *"None met"* to the end.

**7b. Pass them to custom code.** In a **Custom code** action, under **Property to
include in code**, add a row per input. The value picker lists them under **Trigger
data** → your event name *(trigger)*, as paths like `data.object.id`. Delete any empty
**Key** rows, or saving fails with *"Property selection is required"*.

**7c. Read as little as you can from the payload.** The webhook URL takes posts from
anyone (see [Security](#security)). The safe pattern is to pass only the object's id
(`data.object.id`) and have the code fetch the object from Stripe's API, then work from
that. It also gets you fields Stripe leaves out of the event, like a checkout session's
line items.

**7d.** Turn the workflow on.

✅ **Check:** the workflow is on, and its trigger names your webhook event.

### Step 8: Test end to end

About 5 minutes.

**8a.** Make another real event (Step 4).

**8b.** In the workflow, open **Run History**. The trigger reads **Webhook event is
received** with a green check, and each action shows **Done**, its **Outputs** and
*"Successfully executed"*.

**8c. Replay a delivery without a new purchase.** In Stripe, open the event (Workbench
→ **Events**), find **Deliveries to webhook endpoints** and click **Resend** (wording
may differ). Use it after you fix a workflow. If your code writes records, make it look
records up first, so a resend updates instead of creating a duplicate.

**8d.** On the Stripe destination's **Event deliveries** tab, every delivery should be
a 2xx. A failed delivery here means HubSpot rejected it, not that your workflow failed.

✅ **Check:** one new event, one workflow run, and the outcome you expected.

## Which properties to keep

A starting list. Add what your branches and code read; delete the rest.

**`checkout.session.completed`**

| Path | Why |
|---|---|
| `id` | Stripe's event id, to trace one delivery back |
| `data.object.id` | The session id: the one field custom code needs to fetch the session |
| `data.object.amount_total` | Branch on it (**Number**) |
| `data.object.payment_intent` | Empty on a $0 checkout, set on a paid one: a second test that doesn't depend on amount |
| `data.object.status` | Act only on `complete` |
| `data.object.mode` | `payment` vs `subscription` |
| `data.object.livemode` | Catch a test destination pointed at a live workflow (**Boolean**) |
| `data.object.customer_details.email` | The buyer's typed email; required if you match on Email |
| `data.object.created` | Readable run history |

Don't filter on `payment_status`. It read `paid` on a 100%-off checkout in production.

Usually safe to delete: everything under `custom_text`, `payment_method_options`,
`automatic_tax`, `total_details`, `consent`, `phone_number_collection`,
`shipping_address_collection`, plus the long tail of fields that are `null` in your
sample.

**`payment_intent.succeeded`**

| Path | Why |
|---|---|
| `id`, `type` | Trace a delivery; confirm the event type |
| `data.object.id` | The PaymentIntent id |
| `data.object.amount`, `data.object.amount_received`, `data.object.currency` | Amount (**Number**) and currency |
| `data.object.status` | `succeeded` |
| `data.object.livemode` | **Boolean** |
| `data.object.created` | |
| `data.object.customer`, `data.object.latest_charge`, `data.object.receipt_email` | Joins to the customer and charge |

Delete the `data.object.charges.*`, `shipping.*` and `payment_method_options.*`
families unless you need them.

## Troubleshooting

| You got | Cause | Fix |
|---|---|---|
| *"...webhook-definitions failed with status 400."* on Create | Over 50 properties (pop-up wizard) | Read the real reason in DevTools (Step 6c), then delete rows (Step 5) |
| *"The limit of 50 properties has been reached..."* | Over 50 properties (four-step wizard) | Step 5 |
| *"Fix any property errors before continuing."*; **Next** greyed out | A row with a bad type, usually `api_version` as **Date** | Delete or retype it (Step 5d) |
| *"There are errors or missing values in the properties below..."* | A kept card with no **Data type** | Step 5d |
| *"A property label can't be more than 50 characters."* | A deep path as the label | Shorten it or delete the row (Step 5) |
| *"stripe trigger is disabled in live mode..."* | Stripe Shell / CLI in live mode | Make a real event, or work in a sandbox (Step 4) |
| *"Property selection is required"* in Custom code | An empty input row | Delete it (Step 7b) |
| Paid events run the free/$0 path | Amount typed as text, or the path wired to *"None met"* | Retype to **Number** (Step 5d); fix the branch (Step 7a) |
| Events arrive but nothing enrolls | Connected event, payload email isn't a contact | Rebuild as unconnected (Step 1); the setting can't be changed |
| Properties arrive empty | A different event type sent to the same webhook event | One Stripe event per destination (Step 3b) |
| You need a field you deleted | It's not in the definition | Add it back: the payload still carries it. In the pop-up, **+ Add custom property** (wording may differ), or rebuild the event. |

## Limits

- **50 properties per webhook event**, and **50 characters per property label**.
- **The CRM link (connected or not) is fixed** once the event is created.
- **Stripe's payload follows the destination's API version.** Older versions name some
  fields differently; build from a sample with the version you'll run.
- **HubSpot can't verify Stripe's signature** (see Security).
- **A webhook event is forward-only.** Events from before the destination existed never
  arrive. Backfill from Stripe's API.

## Security

- **The webhook URL is an unauthenticated write endpoint.** Anyone with it can post a
  made-up "Stripe" event into your workflow, and HubSpot can't check Stripe's
  signature. So:
  - Don't share, paste or screenshot the URL.
  - Never trust money or identity fields from the payload for anything that matters.
    Pass the object id to custom code and re-read the object from Stripe's API (Step
    7c). A forged post then has to name a real object that passes your checks.
  - Keep `livemode` and check it.
- **The test sample in the wizard and the run history contain customer data.** Treat
  screenshots of them as personal data.
- Use a **restricted** Stripe key in custom code, with read access to only what the
  code fetches.

## Related repos: Stripe beyond HubSpot Commerce

This repo is one of a set of guides for taking Stripe payments without HubSpot
Commerce, and for getting the Stripe data that HubSpot's native Stripe
integration leaves out into HubSpot. Each one stands alone.

| Repo | What it adds |
|---|---|
| [hubspot-order-form-stripe-checkout-link-integration](https://github.com/carljibrilsulaimanii/hubspot-order-form-stripe-checkout-link-integration) | A HubSpot order form that hands buyers to a Stripe Payment Link, and writes the UTMs back onto the payment record |
| **stripe-webhooks-to-hubspot-custom-events** (this repo) | Any Stripe event into a HubSpot workflow through the "Webhook event is received" trigger, no middleware |
| [hubspot-capi-server-side-lead-and-purchase-conversions-meta-google](https://github.com/carljibrilsulaimanii/hubspot-capi-server-side-lead-and-purchase-conversions-meta-google) | Stripe purchases sent server-side from HubSpot workflows to Meta and Google |
| [hubspot-stripe-zero-dollar-checkout-sync](https://github.com/carljibrilsulaimanii/hubspot-stripe-zero-dollar-checkout-sync) | Free and 100%-off Stripe Checkout orders, which create no payment, written into a HubSpot custom object, plus a backfill |
| **Product names on payment records** (coming) | Which product each Stripe payment was for, and routing buyers by product |
| **Stripe test mode mirror** (coming) | Test payments in the same HubSpot object as live ones, so workflows can be tested without real charges |

---

Built by [Jibril Sulaiman](https://github.com/carljibrilsulaimanii).
