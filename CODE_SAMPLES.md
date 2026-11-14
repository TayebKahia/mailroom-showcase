# Code samples

Excerpts from the private Mailroom repository (version 1.0.0), copied as they are, apart from two references to internal design records removed from docstrings; `# …` marks code left out between two parts of one file. They show the three places where Mailroom's safety is enforced by code rather than by the model: what an Operator sees, what is written once, and what can never be changed afterwards.

## 1. The ProposalChecker: code decides what a Proposal can do

The model returns a draft and its planned Actions. Before an Operator sees anything, `check` traces each Citation to this Run's own tool results, drops Actions the thread does not support, and runs the Pricing Guard: every amount in the reply must be a Catalog price valid on the event date, or such a price times a quantity the customer stated. Anything else blocks approval until a person fixes it or confirms it as their own quote. The recipient and subject are set here by code; the model has no field for them.

```python
def check(draft: DraftProposal, run_context: RunContext, catalog: Catalog) -> CheckedProposal:
    """The Proposal an Operator sees, with its issues and any post-draft Escalation."""
    facts = _ThreadFacts.of(run_context)
    citations, uncited = _citations(draft.cited_excerpts, run_context.retrieved)
    actions, mismatches = _matching_actions(draft.actions, facts, catalog, run_context)
    issues = [
        *_email_warnings(run_context),
        *uncited,
        *_pricing_guard(draft.draft_text, facts, catalog),
        *mismatches,
    ]
    escalate: EscalationReason | None = None
    # In eval/README.md's order: a special price before a gap in the Knowledge Base.
    if _asks_or_offers_a_special_price(draft.draft_text, run_context.unanswered_texts):
        escalate = "pricing_request"
    elif not citations and _the_documents_left_a_gap(run_context):
        escalate = "kb_not_found"
    return CheckedProposal(
        proposal_id=run_context.proposal_id,
        conversation_id=run_context.conversation_id,
        run_id=run_context.run_id,
        recipient=run_context.customer,
        subject=_reply_subject(run_context.thread_subject),
        draft_text=draft.draft_text,
        actions=actions,
        citations=citations,
        issues=issues,
        escalate=escalate,
    )

# …

def _pricing_guard(text: str, facts: _ThreadFacts, catalog: Catalog) -> list[CheckIssue]:
    """Every amount in the Draft must be a Catalog price valid on the event date (the
    Message's date when it is unknown), or such a price times a quantity the Customer stated
    for its unit (guests, hours, days). Any other amount, or one in another currency, blocks.
    A sum of several prices blocks too: the Operator confirms it as their own quote."""
    allowed = _allowed_amounts(facts, catalog)
    currency = catalog.currency
    issues: list[CheckIssue] = []
    for amount in find_amounts(text):
        if amount.currency in (None, currency) and amount.value in allowed:
            continue
        written = text[slice(*amount.span)]
        issues.append(
            CheckIssue(
                code="unlisted_amount",
                blocking=True,
                detail=(
                    f"{written} is not a Catalog price in {currency}, nor one times a "
                    "quantity the Customer stated"
                ),
                span=amount.span,
            )
        )
    return issues


def _allowed_amounts(facts: _ThreadFacts, catalog: Catalog) -> frozenset[Decimal]:
    on = facts.priced_on
    allowed: set[Decimal] = set()
    for item in catalog.items:
        if item.valid_from <= on and (item.valid_to is None or on <= item.valid_to):
            allowed.add(item.price)
            allowed.update(item.price * quantity for quantity in facts.quantities[item.unit])
    return frozenset(allowed)
```

The tests read like a price list. Each case is one sentence a model might write, and the amounts that must be blocked:

```python
@pytest.mark.parametrize(
    ("draft_text", "customer_text", "guest_count", "blocked"),
    [
        # …
        # Amounts in words.
        ("Dinner is sixty-two pounds per head.", WEDDING, 80, []),
        ("Hire is one thousand nine hundred and fifty pounds.", WEDDING, 80, []),
        ("Hire is two hundred pounds.", WEDDING, 80, ["two hundred pounds"]),
        # Not on the Catalog, or not this Catalog's currency.
        ("Evening hire is £1,800 per event.", WEDDING, 80, ["£1,800"]),
        ("For 80 guests the dinner comes to £4,961.", WEDDING, 80, ["£4,961"]),
        ("The dinner is $62 per head.", WEDDING, 80, ["$62"]),
        ("Evening hire is 1,950 euros.", WEDDING, 80, ["1,950 euros"]),
        ("Evening hire is EUR 1,950.", WEDDING, 80, ["EUR 1,950"]),
        # A Catalog price times a quantity nobody stated.
        ("£62 per head, so for 81 guests it is £5,022.", WEDDING, 80, ["£5,022"]),
        ("£150 per hour, so three hours would be £450.", TWO_HOURS, None, ["£450"]),
        # Numbers that are not amounts are not checked.
        ("We can seat 80 guests from 18:00 to 01:00 on 12 June 2027 (BM-24817).", WEDDING, 80, []),
    ],
)
def test_the_pricing_guard_blocks_every_amount_not_from_the_catalog(
    draft_text: str, customer_text: str, guest_count: int | None, blocked: list[str]
) -> None:
    fields = _fields(guest_count=Unknown() if guest_count is None else guest_count)

    checked = check(_draft(text=draft_text), _context(customer_text, fields=fields), CATALOG)

    issues = [issue for issue in checked.issues if issue.code == "unlisted_amount"]
    assert [draft_text[slice(*issue.span)] for issue in issues if issue.span] == blocked
    assert all(issue.blocking for issue in issues)
    assert len(issues) == len(blocked)
    assert checked.blocked == bool(blocked)
```

## 2. The Executor: every approved Action runs once, across crashes

An Action can crash between "the email was sent" and "we recorded that it was sent". Each Action carries an Action Id that is written into what it creates (a Gmail header, a Calendar extended property, a HubSpot contact property). After a restart, `reconcile` looks each unfinished Action up by that Id: found means done, not found means retry once, and a failed lookup goes to a person instead of a blind retry.

```python
    def execute_approved(self, conn: psycopg.Connection, action_id: ActionId) -> ActionResult:
        """Execute one Action if it is ready. An Action not ready, or already started, is not
        touched: its recorded state is returned. Raises what the write raised when it was
        neither done nor refused: the Action stays `executing` for `reconcile()`."""
        action = self._start(conn, action_id)
        if action is None:
            return _recorded(conn, action_id)
        return self._write(conn, action)

    def reconcile(self, conn: psycopg.Connection) -> list[ActionResult]:
        """Settle every Action left `executing` by a crash, at worker start. Found
        by its Action Id: executed. Not found: retried once (a hold or reply already retried
        is escalated instead). The lookup failing: `failed` and escalated `execution_failed`
        for a person to check, except a lead, whose reply has gone: it joins the CRM
        retries. An Action that cannot be settled stays `executing` and the rest go on. A
        disconnected mailbox stops it, raising `MailboxDisconnected`: nothing can be looked
        up, so the worker reconciles again once it is connected. One worker reconciles at a
        time (Compose runs one)."""
        rows = conn.execute(
            "SELECT action_id FROM actions WHERE state = 'executing' ORDER BY updated_at"
        ).fetchall()
        results: list[ActionResult] = []
        for (action_id,) in rows:
            try:
                results.append(self._reconcile(conn, _load(conn, action_id)))
            except MailboxDisconnected:
                raise
            except Exception as error:
                logger.warning(
                    "action not reconciled, left executing",
                    extra={"action_id": str(action_id), "error": _type(error)},
                )
        return results

    def _reconcile(self, conn: psycopg.Connection, action: _Action) -> ActionResult:
        try:
            found = self._find(action)
        except MailboxDisconnected:
            raise
        except Exception as error:
            logger.warning(
                "action lookup failed",
                extra={"action_id": str(action.action_id), "error": _type(error)},
            )
            if action.kind == "upsert_crm_lead":
                return self._lead_failed(conn, action, _reason(error))
            return self._escalated(
                conn,
                action,
                f"Not known whether it was {_DONE[action.kind]}: after a restart it could not"
                f" be looked up ({_reason(error)}). Check by hand before doing it again.",
            )
        if found is not None:
            return self._executed(conn, action, found)
        if action.kind == "upsert_crm_lead" and action.attempts > len(CRM_RETRY_DELAYS):
            return self._lead_failed(conn, action, "not found after a restart")
        if action.kind != "upsert_crm_lead" and action.attempts > 1:
            return self._escalated(
                conn,
                action,
                f"Not {_DONE[action.kind]}: not found after a restart, and already retried once"
                " after an earlier one. Check by hand before doing it again.",
            )
        retried = self._retrying(conn, action)
        if retried is None:
            return _recorded(conn, action.action_id)
        return self._write(conn, retried, after_restart=True)

    def _find(self, action: _Action) -> str | None:
        """The provider's id of what this Action wrote, looked up by its Action Id."""
        if action.kind == "send_reply":
            sent = self._inbox.find_sent(action.thread_id, action.action_id)
            return sent.provider_ref if sent else None
        if action.kind == "create_calendar_hold":
            event = self._calendar.find_hold(action.action_id)
            return event.provider_ref if event else None
        lead = self._crm.find_lead(action.action_id)
        return lead.provider_ref if lead else None
```

The test crashes the process at every Action in turn, before and after the provider write, and counts what reached the providers:

```python
@pytest.mark.parametrize("after_write", [True, False], ids=["after-write", "before-write"])
def test_a_crash_at_every_action_still_writes_one_email_one_hold_and_one_lead(
    fresh_database_url: str, tmp_path: Path, after_write: bool
) -> None:
    """Each adapter crashes the process once, after (or before) its provider write and
    before `executed` is recorded. Each restart is a new connection and new adapters on the
    same folders; `reconcile()` finds the write by its Action Id, or retries it once."""
    with psycopg.connect(fresh_database_url, autocommit=True) as conn:
        conversation, action_ids = _approved(conn, FakeInbox(tmp_path / "inbox"))

    crashing = [
        Executor(
            CrashingInbox(tmp_path / "inbox", after_write=after_write),
            BUSINESS,
            calendar=CrashingCalendar(tmp_path / "calendar.jsonl", after_write=after_write),
            crm=CrashingCRM(tmp_path / "crm.jsonl", after_write=after_write),
            clock=Clock(),
        )
    ] * 3
    reconciled: list[list[Any]] = []
    for executor in crashing:  # crashes at the hold, then the reply, then the lead
        with psycopg.connect(fresh_database_url, autocommit=True) as conn:
            reconciled.append([r.state for r in Adapters(tmp_path).executor().reconcile(conn)])
            with pytest.raises(Crash):
                executor.execute_ready(conn)
    with psycopg.connect(fresh_database_url, autocommit=True) as conn:
        restarted = Adapters(tmp_path).executor()
        reconciled.append([r.state for r in restarted.reconcile(conn)])
        assert restarted.execute_ready(conn) == []
        after = _reload(conn, conversation)

    adapters = Adapters(tmp_path)
    hold_id, reply_id, lead_id = action_ids
    assert [s.action_id for s in adapters.inbox.sent_replies()] == [reply_id]
    assert list(adapters.calendar.holds) == [hold_id]
    assert list(adapters.crm.leads) == [lead_id]
    # Counted in the files too: a second write with the same Action Id would hide in a dict.
    assert len((tmp_path / "calendar.jsonl").read_text().splitlines()) == 1
    assert len((tmp_path / "crm.jsonl").read_text().splitlines()) == 1
    assert reconciled == [[], ["executed"], ["executed"], ["executed"]]
    assert [(a.state, a.attempts) for a in after.actions] == [
        ("executed", 1 if after_write else 2)
    ] * 3
    assert after.status == "replied"
    assert [run.state for run in after.runs] == ["executed"]
```

## 3. The audit chain: history the app cannot change

Every state change, Proposal, Approval and Action result is appended to one hash chain inside the transaction that makes the change. The app's database role may only insert and read this table, and `mailroom audit verify` recomputes the chain and names the first row that was edited, deleted or moved.

```python
def body_hash(body: str | None) -> bytes | None:
    return None if body is None else hashlib.sha256(body.encode("utf-8")).digest()


def chain_hash(
    prev_hash: bytes, fields: Mapping[str, JsonValue], body_digest: bytes | None
) -> bytes:
    """A row's hash. Every part has a fixed length or ends itself (a JSON object), so no two
    different rows hash the same input."""
    return hashlib.sha256(prev_hash + canonical_json(dict(fields)) + (body_digest or b"")).digest()


def append(conn: psycopg.Connection, event: AuditEvent) -> None:
    """Append `event`. Call it inside the transaction that makes the change it records, at
    READ COMMITTED (the default; anything else is refused): the previous hash must be read
    after the lock is taken."""
    idle = conn.info.transaction_status == TransactionStatus.IDLE
    with conn.transaction() if idle else nullcontext():
        if event.conversation_id is not None:
            # Conversation first, then the chain: the order every writer keeps.
            conn.execute(
                "SELECT 1 FROM conversations WHERE id = %s FOR UPDATE", (event.conversation_id,)
            )
        conn.execute("SELECT pg_advisory_xact_lock(%s)", (_CHAIN_LOCK_KEY,))
        # clock_timestamp, not now(): read under the lock, times follow the chain's order.
        row = conn.execute(
            "SELECT clock_timestamp(), current_setting('transaction_isolation'),"
            " (SELECT hash FROM audit_events ORDER BY sequence DESC LIMIT 1)"
        ).fetchone()
        assert row is not None
        recorded_at, isolation, prev_hash = row
        if isolation != "read committed":
            # An older snapshot would miss the last append and fork the chain.
            raise RuntimeError(f"Audit Trail appends need READ COMMITTED, not {isolation}")
        prev_hash = prev_hash or GENESIS
        digest = body_hash(event.body)
        fields = _fields(
            {
                "recorded_at": recorded_at,
                "conversation_id": event.conversation_id,
                "run_id": event.run_id,
                "kind": event.kind,
                "actor": event.actor,
                "operator_email": event.operator_email,
                "metadata": event.metadata,
            }
        )
        conn.execute(
            "INSERT INTO audit_events (recorded_at, conversation_id, run_id, kind, actor,"
            " operator_email, metadata, body, body_hash, prev_hash, hash)"
            " VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)",
            (
                recorded_at,
                event.conversation_id,
                event.run_id,
                event.kind,
                event.actor,
                event.operator_email,
                Jsonb(event.metadata),
                event.body,
                digest,
                prev_hash,
                chain_hash(prev_hash, fields, digest),
            ),
        )

# …

def verify(conn: psycopg.Connection) -> VerifyReport:
    """Recompute the whole chain, oldest row first, in one snapshot: appends committing
    meanwhile are not seen, and cannot make a valid chain look broken."""
    events = 0
    expected_prev = GENESIS
    head_sequence: int | None = None
    with _snapshot(conn), conn.cursor("audit_verify", row_factory=dict_row) as cursor:
        cursor.itersize = _VERIFY_BATCH
        cursor.execute(
            f"SELECT {_RECORD_COLUMNS}, body_hash, prev_hash, hash"
            " FROM audit_events ORDER BY sequence"
        )
        for row in cursor:
            sequence: int = row["sequence"]
            problem = _problem(row, expected_prev)
            if problem is not None:
                return VerifyReport(events, head_sequence, expected_prev, sequence, problem)
            events += 1
            head_sequence = sequence
            expected_prev = bytes(row["hash"])
    return VerifyReport(events, head_sequence, None if head_sequence is None else expected_prev)

# …

def _problem(row: Mapping[str, Any], expected_prev: bytes) -> str | None:
    """How the row fails to fit the chain after a row whose hash is `expected_prev`."""
    if bytes(row["prev_hash"]) != expected_prev:
        return "it does not follow the event before it: an event was deleted, inserted or moved"
    stored_digest = None if row["body_hash"] is None else bytes(row["body_hash"])
    if row["body"] is not None and body_hash(row["body"]) != stored_digest:
        return "its text was changed"
    if chain_hash(expected_prev, _fields(row), stored_digest) != bytes(row["hash"]):
        return "its record was changed"
    return None
```

```python
def test_an_edited_row_is_found_at_its_sequence(
    fresh_database_url: str, as_owner: Owner, column: str, value: object
) -> None:
    sequences = _append_five(fresh_database_url)
    target = sequences[2]
    _tamper(
        as_owner(fresh_database_url),
        f"UPDATE audit_events SET {column} = %s WHERE sequence = %s",
        value,
        target,
    )

    report = _verify(fresh_database_url)

    assert not report.ok
    assert report.broken_at == target


def test_a_deleted_row_is_found_at_the_row_after_it(
    fresh_database_url: str, as_owner: Owner
) -> None:
    sequences = _append_five(fresh_database_url)
    _tamper(
        as_owner(fresh_database_url), "DELETE FROM audit_events WHERE sequence = %s", sequences[2]
    )

    report = _verify(fresh_database_url)

    assert report.broken_at == sequences[3]
    assert report.problem is not None and "deleted" in report.problem
```
