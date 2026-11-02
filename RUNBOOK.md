# Runbook: Durable money assistant

This runbook is for presenting the demo. The [README](README.md) explains what it is and how it's built.

## What it shows

A customer asks a Claude-powered banking assistant to send money. The assistant can read balances and price a transfer, but it can't move money. It starts a Temporal Workflow that builds a plan, waits for the customer to tap **Approve transfer**, and then withdraws and deposits, refunding the withdrawal if the deposit fails.

The demo makes three problems visible:

- **An agent that can move money can be talked into it, or can claim it did.** The approval gate lives in Workflow code, and the approval reaches the Workflow as an Update from a button tap. The model has no tool that approves. Beats 1 to 3 show it.
- **A transfer that fails between the debit and the credit leaves the customer's money nowhere.** The Workflow runs the transfer as a saga: when the deposit into a closed account fails, it refunds the withdrawal. Beat 5 shows it.
- **Downstream systems fail for a moment.** The deposit fails twice and Temporal retries it on a fixed interval, with idempotency keys so each effect lands once. Beat 3 shows it.

## Before you start

- **On tmprl-demo.cloud:** open https://durable-banking-skills.tmprl-demo.cloud. The worker, the agent API, and the UI run in one container that shares a SQLite bank. The workflow links on the transfer cards open Temporal Cloud in Namespace `tmprl-dem-cld-durable-banking-skills.a2dd6`, so sign in to Temporal Cloud with access to that Namespace before you present.
  - The bank is shared by everyone who opens the demo, and it only goes back to its starting balances when the container restarts. Check the **Demo accounts** rail before you start; see [Reset and tear down](#reset-and-tear-down) if 85-150 is low.
  - Each chat message calls the Claude API with the shared Anthropic key.
- **To run it locally:** install [uv](https://docs.astral.sh/uv/), Node 18 or later, and the [Temporal CLI](https://docs.temporal.io/cli). You need an Anthropic API key.
  1. Clone the repo, then from the root run `uv sync` and `cd web && npm install && cd ..`.
  2. `cp .env.example .env`, then set `ANTHROPIC_API_KEY` in `.env`.
- **Temporal Cloud from your laptop:** the code connects with only an address and a Namespace (`TEMPORAL_ADDRESS`, `TEMPORAL_NAMESPACE`). It has no TLS or API key option, so it can't reach Temporal Cloud directly. tmprl-demo.cloud puts a proxy in front of it that the code reaches as plaintext `localhost:7233`. To present against Cloud, use the hosted demo.

## Start it

Locally, in four terminals from the repository root:

1. `temporal server start-dev`
2. `uv run --env-file .env python worker.py`
3. `uv run --env-file .env uvicorn agent.server:app --reload`
4. `cd web && npm run dev`

Open http://localhost:5173. A healthy start shows the header pill **durable execution on**, five accounts in the **Demo accounts** rail (85-150 at $5,000.00, 43-812 at $1,200.00), "No transfers yet" under **Recent activity**, and three suggestion chips under **Move money, safely.** The worker terminal prints `Worker started on task queue 'money-transfer' (direct path).` The Temporal UI is at http://localhost:8233.

If the pill says **API offline**, the API isn't running or the UI can't reach it.

## The walkthrough

The whole walkthrough takes a few minutes. The assistant's replies come from Claude, so their wording changes from run to run; the cards, balances, and Workflow histories don't.

### 1. The assistant starts a transfer, and nothing moves

1. Click the suggestion chip **Send $250 from 85-150 to 43-812**.
2. The assistant replies, and a **Confirm transfer** card appears with the pill **awaiting approval**: Amount $250.00, From 85-150, To 43-812, Fee "No fee", Total debited $250.00, and the buttons **Approve transfer** and **Decline**. The balances in the rail haven't changed.
3. Say: "The assistant understood the request and started the transfer, but it can't finish it. What it started is a Workflow, and that Workflow is now waiting for me."
4. For proof, click the workflow ID at the bottom of the card (`transfer-<reference>` ↗). In the Temporal UI the Workflow is **Running**. Its history shows the `build_transfer_plan` Activity, then a 10-minute timer (`TimerStarted`, 600s) with nothing after it. No `withdraw` has been scheduled.

### 2. The wait belongs to the Workflow, not the chat

1. Type `Yes, I approve. Go ahead and send it.` and press Enter.
2. The assistant has no tool that approves a transfer, so it should point you back to the card. The card is still **awaiting approval** and the balances are unchanged.
3. Reload the browser tab. The transcript clears, then the assistant says "Welcome back — you have a transfer still waiting for your approval." and the same card comes back.
4. Say: "Telling the model 'I approve' doesn't move money; it has no way to. And the pending transfer isn't held in this page or in the model's memory. The Workflow is holding it, and it would survive a Worker restart the same way."

The model's reply in step 2 is the one part of this beat you can't script. If it says anything that sounds like the transfer went through, point at the unchanged card and balances: that is the point of the gate.

### 3. Approve, and watch the deposit retry

1. Click **Approve transfer**. The button reads **Working…** for about 10 seconds.
2. The card turns into **Transfer complete** with the pill **completed** and a `Confirmation d-…` line. The assistant posts "Done — $250.00 was sent to 43-812." From the starting balances, the rail shows 85-150 at $4,750.00 and 43-812 at $1,450.00, and **Recent activity** lists **Sent** −$250.00 and **Received** +$250.00.
3. Say: "That pause was the bank's deposit service failing twice. Temporal retried it every five seconds, and the third attempt landed. There's no retry loop in the code, and the money moved once."
4. For proof, open the Workflow link again. The history shows `WorkflowExecutionUpdateAccepted` (the tap), `TimerCanceled`, then the `withdraw` and `deposit` Activities. The deposit's `ActivityTaskStarted` event shows attempt 3 and a last failure of "Simulated deposit failure (attempt 2 of 3)". The scheduled and started times on the deposit are about 10 seconds apart.

To show the transfer fee as well, use the chip **Transfer $1,500 from 85-150 to 43-812**: the card shows a $7.50 fee and $1,507.50 total.

### 4. A bad request never reaches the approval step

1. Type `Send $100 from 85-150 to 55-200`.
2. The assistant explains that 55-200 is frozen. No card appears.
3. Say: "The plan failed validation inside the Workflow, so it never asked anyone to approve anything."
4. For proof, find the newest `transfer-…` Workflow in the Temporal UI. It **Completed** right after `build_transfer_plan`, with result status `invalid` and no timer. If the assistant only priced the transfer with its quote tool, there's no Workflow to show; the quote runs the same validation function.

Skip this beat when time is short.

### 5. The deposit fails for good, and the Workflow refunds the withdrawal

1. Type `Send $100 from 85-150 to 66-300`. 66-300 is a closed account that passes validation on purpose.
2. Click **Approve transfer**. This one comes back in a second or two.
3. The card turns into **Transfer failed — funds returned** with the pill **failed** and a `Refund r-…` line. The assistant says it couldn't deposit the funds because the destination account is closed, and that the withdrawal was rolled back. **Recent activity** lists **Sent** −$100.00 then **Refunded** +$100.00, and 85-150 is back where it was.
4. Say: "The bank took the money out of checking, then refused the deposit. Retrying a closed account won't help, so the Workflow didn't. It ran the compensation and put the money back."
5. For proof, open the Workflow link. The history shows `withdraw` completed, `deposit` failed on its first attempt with type `AccountClosedError`, then a `refund` Activity completed. The Workflow itself is **Completed**: the failure is a business outcome it handled, and its result has status `failed` and a `refund_txn_id`.

### 6. Decline

1. Start another transfer, for example the **Send $250 from 85-150 to 43-812** chip, then click **Decline**. You can also type `cancel that`; the assistant has a tool that can decline, but not approve.
2. The card turns into **Transfer declined**. Nothing appears under **Recent activity**.
3. Say: "Declining moves no money, so the assistant is allowed to do it on your word. Approving is the one thing it can't do."

## Show it in code

All links point at commit [`c0333fb`](https://github.com/temporal-sa/durable-agent-skills/commit/c0333fb86fdbeb2010265a921c29b96d0cc16931). Open a file locally with `code -g <path>:<line>`.

### Hold the transfer until a person approves it

**When to use it:** after beat 1 or 2, or when someone asks what stops the agent from sending money.

**Where it is:** [`skills/money_transfer/workflow.py` lines 157–183](https://github.com/temporal-sa/durable-agent-skills/blob/c0333fb86fdbeb2010265a921c29b96d0cc16931/skills/money_transfer/workflow.py#L157-L183) (`code -g skills/money_transfer/workflow.py:157`)

```python
# Step 2: wait for an explicit human decision.
self._status = "awaiting_approval"
try:
    await workflow.wait_condition(
        lambda: self._decision is not None, timeout=APPROVAL_TIMEOUT
    )
except asyncio.TimeoutError:
    self._status = "expired"
    # ...
    return self._result

decision = self._decision
assert decision is not None
if not decision.approved:
    self._status = "declined"
    # ...
    return self._result

# Step 3: execute the transfer as a saga.
return await self._execute(request, plan)
```

**What you need to understand:**

- `wait_condition` blocks the Workflow until a decision has arrived. With a `timeout`, it starts a durable timer; if the timer fires first, it raises `asyncio.TimeoutError` and the transfer ends `expired`. `APPROVAL_TIMEOUT` is 10 minutes ([line 69](https://github.com/temporal-sa/durable-agent-skills/blob/c0333fb86fdbeb2010265a921c29b96d0cc16931/skills/money_transfer/workflow.py#L69)).
- No Worker slot is held while it waits; the Temporal Service keeps the timer. The timer is in the Workflow's history, so a Worker restart, a deploy, or a closed browser tab doesn't lose the pending transfer or reset the clock.
- `_execute`, which moves the money, is only reachable after `decision.approved` is true. A prompt can't skip this line; the agent can only start the Workflow and relay a decision into it.
- A customer would set the timeout to their own approval window. Ten minutes suits a chat; an approval by a second person might wait days.

**Docs:** [Python: wait_condition](https://python.temporal.io/temporalio.workflow.html#wait_condition), [Durable timers](https://docs.temporal.io/workflow-execution/timers-delays)

### Accept a decision only while the transfer is waiting

**When to use it:** after beat 3, or when someone asks what happens if two people tap, or someone taps after it expired.

**Where it is:** [`skills/money_transfer/workflow.py` lines 115–131](https://github.com/temporal-sa/durable-agent-skills/blob/c0333fb86fdbeb2010265a921c29b96d0cc16931/skills/money_transfer/workflow.py#L115-L131) (`code -g skills/money_transfer/workflow.py:115`)

```python
# -- Update: the human decision, relayed by the confirmation step --------

@workflow.update
async def submit_decision(self, decision: ApprovalDecision) -> str:
    self._decision = decision
    return "recorded"

@submit_decision.validator
def _validate_decision(self, decision: ApprovalDecision) -> None:
    # Reject anything that would let a decision land at the wrong time. The
    # validator is read-only and runs before the update is admitted.
    if self._status != "awaiting_approval":
        raise ValueError(
            f"This transfer is '{self._status}', not awaiting approval."
        )
    if self._decision is not None:
        raise ValueError("A decision has already been recorded for this transfer.")
```

**What you need to understand:**

- The decision is an Update: the caller waits for the Workflow to accept or reject it, and gets a result back. A Signal would be fire-and-forget.
- The validator runs before the Update is accepted. Raising rejects it, and a rejected Update writes nothing to the Workflow's history. So a second decision never reaches the handler. In beat 3 the accepted one shows as `WorkflowExecutionUpdateAccepted` and `WorkflowExecutionUpdateCompleted`.
- After the Workflow has closed, for example because the 10-minute timer fired, there's nothing left to validate: the server refuses the Update with "workflow execution already completed". The client code below catches a validator rejection, but not that error, so the UI reports a tap on an expired card as a connection problem. See [When something goes wrong](#when-something-goes-wrong).

**Docs:** [Python: Update handlers and validators](https://docs.temporal.io/develop/python/workflows/message-passing#updates), [Update validators](https://docs.temporal.io/handling-messages#update-validators)

### The approval never passes through the model

**When to use it:** after beat 2, when someone asks how the tap gets to the Workflow and why the agent can't send the same call.

**Where it is:** [`agent/server.py` lines 137–143](https://github.com/temporal-sa/durable-agent-skills/blob/c0333fb86fdbeb2010265a921c29b96d0cc16931/agent/server.py#L137-L143) (`code -g agent/server.py:137`), which calls [`skills/money_transfer/client.py` lines 204–228](https://github.com/temporal-sa/durable-agent-skills/blob/c0333fb86fdbeb2010265a921c29b96d0cc16931/skills/money_transfer/client.py#L204-L228) (`code -g skills/money_transfer/client.py:204`)

```python
@app.post("/api/transfer/decision", response_model=DecisionOut)
async def decide(body: DecisionIn) -> DecisionOut:
    # ...
    outcome = await transfer.submit_decision(body.reference_id, body.approved)
```

```python
async def submit_decision(
    reference_id: str, approved: bool, reason: str = ""
) -> dict:
    """Relay a human's approve/decline into the workflow and return the outcome."""
    client = await get_client()
    # ...
    handle = client.get_workflow_handle(
        workflow_id_for(reference_id), result_type=TransferResult
    )
    try:
        await handle.execute_update(
            MoneyTransferWorkflow.submit_decision,
            ApprovalDecision(approved=approved, reason=reason),
        )
    except WorkflowUpdateFailedError:
        # The decision was rejected by the workflow's validator — the approval
        # window already expired, or a decision was already recorded. ...
        pass
    result: TransferResult = await handle.result()
```

**What you need to understand:**

- **Approve transfer** calls this HTTP endpoint directly. The model isn't in the path, so nothing it says can produce this call.
- The model's tools are in [`agent/skills.py` lines 70–145](https://github.com/temporal-sa/durable-agent-skills/blob/c0333fb86fdbeb2010265a921c29b96d0cc16931/agent/skills.py#L70-L145): `lookup_account`, `get_transfer_quote`, `initiate_transfer`, and `cancel_transfer`. `cancel_transfer` calls the same `submit_decision` with `approved=False` hard-coded ([lines 274–278](https://github.com/temporal-sa/durable-agent-skills/blob/c0333fb86fdbeb2010265a921c29b96d0cc16931/agent/skills.py#L274-L278)), which is why the assistant can decline on your word but can't approve.
- `handle.result()` waits for the Workflow to finish. That is why **Working…** stays up through the deposit retries in beat 3.
- This demo has no login; the session ID is a conversation key, not an identity. A real system would authenticate the person tapping and pass who they are in `decided_by`.

### Retry a flaky deposit, and stop retrying when it can't succeed

**When to use it:** after beat 3 for the retries, and after beat 5 for why the closed account wasn't retried.

**Where it is:** [`skills/money_transfer/workflow.py` lines 44–66](https://github.com/temporal-sa/durable-agent-skills/blob/c0333fb86fdbeb2010265a921c29b96d0cc16931/skills/money_transfer/workflow.py#L44-L66) (`code -g skills/money_transfer/workflow.py:44`)

```python
# Bank validation failures are deterministic facts about the request: retrying
# will not change the outcome, so fail fast and let the workflow handle it.
_NON_RETRYABLE_BANK_ERRORS = [
    "InvalidAccountError",
    "InsufficientFundsError",
    "AccountFrozenError",
    "AccountClosedError",
    "TransferLimitError",
]

_BANK_RETRY_POLICY = RetryPolicy(
    non_retryable_error_types=_NON_RETRYABLE_BANK_ERRORS,
)

# The deposit retries on a constant 5-second interval (rather than the default
# exponential backoff) so the demo's simulated deposit failures retry visibly —
# a 5-second wait between each attempt — before the deposit finally succeeds.
_DEPOSIT_RETRY_POLICY = RetryPolicy(
    initial_interval=timedelta(seconds=5),
    backoff_coefficient=1.0,
    maximum_interval=timedelta(seconds=5),
    non_retryable_error_types=_NON_RETRYABLE_BANK_ERRORS,
)
```

**What you need to understand:**

- Activities retry by default: first retry after 1 second, doubling each time up to 100 seconds, with no limit on attempts. The bank policy keeps those defaults and only names the errors not to retry.
- The deposit policy uses a fixed 5-second interval so the retries are easy to see in beat 3. That's a demo choice; a customer would usually keep exponential backoff.
- The bank raises plain Python exceptions such as `AccountClosedError`. The Python SDK turns an exception that isn't an `ApplicationError` into one whose type is the class name, which is what `non_retryable_error_types` matches. That's why beat 5's deposit failed on attempt 1 and went straight to the refund.
- Neither policy caps attempts. A deposit that kept failing with a retryable error would retry until something stopped it; a customer would set a `schedule_to_close_timeout` or `maximum_attempts` and let the saga refund.
- Retries are safe because each money movement carries an idempotency key, built in the Workflow from the transfer's reference (`withdraw:<ref>`, `deposit:<ref>`, `refund:<ref>`). The bank returns the original transaction when it sees a key again ([`banking/service.py` lines 341–347](https://github.com/temporal-sa/durable-agent-skills/blob/c0333fb86fdbeb2010265a921c29b96d0cc16931/banking/service.py#L341-L347)). A real payments API would take the same key.

**Docs:** [Retry Policies](https://docs.temporal.io/encyclopedia/retry-policies), [Python: Activity retries](https://docs.temporal.io/develop/python/best-practices/error-handling#configure-custom-retry-policies)

### Refund the withdrawal when the deposit fails

**When to use it:** after beat 5, or when someone asks what happens to money that left one account and never arrived in the other.

**Where it is:** [`skills/money_transfer/workflow.py` lines 188–265](https://github.com/temporal-sa/durable-agent-skills/blob/c0333fb86fdbeb2010265a921c29b96d0cc16931/skills/money_transfer/workflow.py#L188-L265) (`code -g skills/money_transfer/workflow.py:188`)

```python
async def _execute(
    self, request: TransferRequest, plan: TransferPlan
) -> TransferResult:
    self._status = "executing"
    compensations: list[Callable[[], Awaitable[object]]] = []

    try:
        # Register the refund *before* withdrawing. If the withdraw commits
        # but the activity is then lost (timeout, crash), the effect still
        # happened, so the compensation must already be on the stack. The
        # refund activity is a no-op when nothing was actually withdrawn.
        compensations.append(
            lambda: workflow.execute_activity(
                refund,
                LedgerInput(
                    # ...
                    idempotency_key=f"refund:{request.reference_id}",
                ),
                start_to_close_timeout=_LEDGER_TIMEOUT,
            )
        )
        withdraw_txn = await workflow.execute_activity(
            withdraw,
            # ...
        )

        deposit_txn = await workflow.execute_activity(
            deposit,
            # ...
        )
        # ...
    except ActivityError as err:
        if is_cancelled_exception(err):
            raise
        workflow.logger.error("Transfer failed, compensating: %s", err)
        self._status = "compensating"
        # ...
        async def run_compensations() -> None:
            nonlocal refund_txn
            for compensate in reversed(compensations):
                try:
                    refund_txn = await compensate()  # type: ignore[assignment]
                except Exception as comp_err:  # noqa: BLE001
                    workflow.logger.error("Compensation failed: %s", comp_err)

        # Shield so compensations still run if the workflow is cancelled.
        await asyncio.shield(asyncio.ensure_future(run_compensations()))
```

**What you need to understand:**

- The saga is ordinary code: a list of undo steps, a `try`, and an `except` that runs them in reverse. Temporal makes it reliable because the Workflow's progress is recorded. If the Worker dies after the withdrawal, the replacement Worker resumes at the deposit, not at the start, and the refund still runs if the deposit fails.
- The refund is registered before the withdrawal because a withdrawal can commit and then time out before reporting back. The bank's `refund` checks whether a withdrawal for this reference exists and does nothing if not ([`banking/service.py` lines 414–452](https://github.com/temporal-sa/durable-agent-skills/blob/c0333fb86fdbeb2010265a921c29b96d0cc16931/banking/service.py#L414-L452)).
- The refund returns `total_debit_cents`, amount plus fee, so a failed transfer costs the customer nothing.
- The refund Activity has no retry policy of its own, so it gets the default: it retries until the bank accepts it. A compensation that gives up would leave the money missing.
- The Workflow ends **Completed** with a result of `failed`. A refused deposit is an expected outcome, so the Workflow returns it rather than failing.

**Docs:** [Saga pattern](https://docs.temporal.io/design-patterns/saga-pattern)

### Optional: start the transfer through Nexus

**When to use it:** only if someone asks how another team or Namespace would call this skill. It's off by default, and the hosted demo doesn't turn it on.

**Where it is:** [`skills/money_transfer/nexus_impl.py` lines 40–53](https://github.com/temporal-sa/durable-agent-skills/blob/c0333fb86fdbeb2010265a921c29b96d0cc16931/skills/money_transfer/nexus_impl.py#L40-L53) (`code -g skills/money_transfer/nexus_impl.py:40`)

```python
@nexusrpc.handler.service_handler(service=MoneyTransferService)
class MoneyTransferServiceHandler:
    @nexus.workflow_run_operation
    async def start_transfer(
        self, ctx: nexus.WorkflowRunOperationContext, request: TransferRequest
    ) -> nexus.WorkflowHandle[TransferResult]:
        # ...
        return await ctx.start_workflow(
            MoneyTransferWorkflow.run,
            request,
            id=workflow_id_for(request.reference_id),
        )
```

**What you need to understand:**

- With `USE_NEXUS=true`, the agent starts a small caller Workflow (`nexus-transfer-<ref>`) that calls the `start_transfer` operation, and the handler starts the same `MoneyTransferWorkflow` at the same ID. The approval, the saga, and the UI don't change.
- The caller depends only on the `MoneyTransferService` contract in `nexus_service.py`, so the skill could live in another Namespace owned by another team. In this repo both sides use the `default` Namespace and the `money-transfer` Task Queue.
- To show it locally, follow the README's [Nexus setup](README.md#optional-run-the-skill-through-a-nexus-operation): create the endpoint once, set `USE_NEXUS=true`, and restart both the worker and the API.

**Docs:** [Nexus](https://docs.temporal.io/nexus)

### What's demo-only

When someone asks what they'd change for their own system, these are the parts to leave out:

- **The deposit fails on purpose.** [`skills/money_transfer/activities.py` lines 24–27](https://github.com/temporal-sa/durable-agent-skills/blob/c0333fb86fdbeb2010265a921c29b96d0cc16931/skills/money_transfer/activities.py#L24-L27) sets `DEPOSIT_SUCCEED_ON_ATTEMPT = 3`, and [lines 113–125](https://github.com/temporal-sa/durable-agent-skills/blob/c0333fb86fdbeb2010265a921c29b96d0cc16931/skills/money_transfer/activities.py#L113-L125) raise `SimulatedDepositFailure` before touching the bank. Set it to 1 to turn it off. The Workflow doesn't know about it.
- **The bank is a SQLite file** shared by the worker and the API ([`banking/service.py`](https://github.com/temporal-sa/durable-agent-skills/blob/c0333fb86fdbeb2010265a921c29b96d0cc16931/banking/service.py#L1-L15)). A customer's Activities would call their core banking or payments API with the same idempotency keys.
- **The closed account passes validation on purpose** ([`banking/service.py` lines 289–293](https://github.com/temporal-sa/durable-agent-skills/blob/c0333fb86fdbeb2010265a921c29b96d0cc16931/banking/service.py#L289-L293)), so beat 5 can show the refund. A real bank would usually reject it up front, and the saga would cover the failures that can't be predicted.
- **The conversation isn't durable.** The agent loop runs in the API process, outside Temporal ([`agent/agent.py` lines 1–8](https://github.com/temporal-sa/durable-agent-skills/blob/c0333fb86fdbeb2010265a921c29b96d0cc16931/agent/agent.py#L1-L8)), and sessions live in memory. Only the money movement is durable. A customer who wants the conversation to survive a restart would run the agent loop in a Workflow too.
- **There is no authentication**, and every visitor shares the same accounts.

## Reset and tear down

- **Between runs, locally:** run `uv run python scripts/seed_bank.py --reset` from the repository root. It wipes the ledger and puts every account back to its starting balance; the worker and API can stay running. Reload the browser tab to get an empty transcript with the suggestion chips. The API still remembers the earlier conversation for this browser until you restart it or clear the site's local storage.
- **Between runs, on tmprl-demo.cloud:** you can't reset the bank, and other people may have used it. If 85-150 is getting low, move money back: `Send $500 from 43-812 to 85-150`, then approve.
- **Open transfers:** a transfer left on **awaiting approval** expires on its own after 10 minutes, and its Workflow completes. The card on screen still says **awaiting approval**. Nothing else keeps running between transfers, so nothing keeps consuming Actions on Temporal Cloud once the open cards are approved, declined, or expired.
- **To stop locally:** decline or approve any open card, then stop the UI, the API, the worker, and the dev server with `Ctrl-C`.

## When something goes wrong

| What you see | Likely cause | Fix |
|---|---|---|
| The header pill says **API offline**, or the rail says "Start the API to load balances." | The API isn't running, or the UI can't reach it | Start `uv run --env-file .env uvicorn agent.server:app --reload`; the Vite dev server proxies `/api` to port 8000 |
| Every message gets "I couldn't reach the service just now. Check that the API is running, then try again." while balances load | The chat call failed, most often because `ANTHROPIC_API_KEY` isn't set for the API | Check the API terminal; set the key in `.env` and restart the API with `--env-file .env` |
| The chat hangs for about a minute, then the assistant says the transfer couldn't be started | The worker isn't running, so the Workflow never builds its plan | Start `uv run --env-file .env python worker.py`, then ask again |
| **Working…** stays up much longer than 10 seconds after **Approve transfer** | The worker stopped after the card appeared | Start the worker; the Workflow continues where it was and the card settles |
| The assistant says the transfer is done, but there's no card | The model answered without calling `initiate_transfer` | Nothing moved; the rail proves it. Ask it again to send the money; that's a good moment to point out the balances didn't change |
| The happy path comes back invalid, "short of the … needed" | 85-150 has been drained by earlier runs | Reset the bank locally, or move money back from 43-812 on the hosted demo |
| Tapping a card gives "I couldn't reach the service to record that decision…" while everything is running | The card is more than 10 minutes old. The transfer expired and its Workflow closed, and the API reports the refused Update as an error. The card doesn't update itself when it expires | Start a new transfer |
| The workflow link opens the wrong Temporal UI or Namespace | `VITE_TEMPORAL_UI` and `VITE_TEMPORAL_NAMESPACE` are baked into the UI at build time | Set them in `web/.env` and restart `npm run dev`, or rebuild the image |
| With `USE_NEXUS=true`: `Workflow class StartTransferWorkflow is not registered on this worker` | The API has the flag but the worker doesn't | Restart the worker with `--env-file .env` |

## Keeping this runbook current

The code links point at a fixed commit. When a linked file changes, update its link, line numbers, and snippet together. Before an event, list the linked files that have changed:

```sh
grep -oE 'blob/[0-9a-f]{40}/[^#)]+' RUNBOOK.md | sort -u | while IFS=/ read -r _ sha file; do
  git diff --quiet "$sha" HEAD -- "$file" || echo "changed since ${sha:0:7}: $file"
done
```
