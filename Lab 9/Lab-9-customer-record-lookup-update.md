# Lab 9 - Create a customer record lookup & update agent using the Voice Live API and Azure Communication Services

**Objective:**

In this lab, you will run and explore a working AI phone agent that gives field sales reps a hands-free line into their CRM. A rep calls, is verified, hears a short briefing on a customer account, dictates what happened in a meeting, and gets the record updated — but only after the agent reads the change back, hears an explicit “yes, save it” and lets a short undo window run out. Requests the agent may not handle are transferred to Sales Operations in Microsoft Teams with the details attached. By the end of the lab, you will be able to:

- Explain how the project is organised and which file owns which decision.

- Read the most important parts of the code: verifying the caller, which records and fields a rep may see, the field allow-list, the policy refusals, the explicit save with an undo window, undo, and the handoff to Sales Operations.

- Run and test the complete agent locally in simulation mode.

- Create and configure Azure Communication Services and the Azure AI Voice Live API with keyless access.

- Call the agent from your own phone as a registered rep, update a record by voice, and transfer a call to Teams with the context attached.

- Clean up the resources afterwards.

**Description:**

Contoso’s field sales reps update the CRM late, or not at all, because typing on a phone after a meeting is painful. In this lab you work with Contoso’s CRM voice line. A rep calls from a registered mobile and enters a six-digit PIN (or, with Teams Phone, is identified by their Teams sign-in). The agent briefs them on an account they are allowed to see, takes a dictated update — a meeting note, deal changes, a follow-up task — and writes it to the CRM only after a read-back, “yes, save it” and a six-second undo window. “Wait” inside the window writes nothing; “undo that” afterwards reverts the change. Discounts, credit limits, payment terms, a new account owner, deleting or merging accounts are refused by policy, and the rep can be put through to Sales Operations.

The agent runs on a Node.js server on your machine. Azure Communication Services (ACS) provides the phone number and call control, the Azure AI Voice Live API (gpt-realtime) listens and speaks, and the server owns every decision: who is calling, what may be read out, which fields may change and within what limits, when a write lands, and where a transfer goes. The CRM is a local SQLite database seeded with fictional accounts.

The complete code is already in the lab files. You explore the key parts, run the agent in simulation mode without Azure, then create the Azure resources, buy a phone number, register your own phone as a rep, prepare Teams for transfers, take the agent live and test it with real calls.

**Prerequisites:** Windows with PowerShell 7, Node.js 22 or later, Visual Studio Code, the Azure CLI, the Dev Tunnels CLI and the MicrosoftTeams PowerShell module; an Azure subscription in the lab tenant where you can create resources and assign roles; a Teams user with a Teams Phone licence to act as Sales Operations; and a phone that can call a US number.

## Exercise 1: Explore the project

**Overview:**

In this exercise, you open the project, install its packages, run the automated tests, and look at the configuration files and the fictional CRM the agent works with.

1. Open a PowerShell window and execute the below commands to navigate to the project, install the packages and open VS Code.

   ```powershell
   cd "C:\LabFiles\voice-ai-template-gallery\docs\templates\customer-record-lookup-update\code"
   pwsh
   npm install --ignore-scripts
   code .
   ```

   ![](images/image21.png)

   ![](images/image22.png)

2. Navigate back to PowerShell and execute the automated tests. They need no Azure and no network.

   ```powershell
   npm test
   ```

   ![](images/image23.png)

   **Expected output**

   ```text
   # tests 53
   # suites 0
   # pass 53
   # fail 0
   ```

   ![](images/image3.png)

3. Look at the project layout in VS Code. Each file owns one job:

   | **File** | **What it does** |
   |---|---|
   | src/flow.mjs | The call state machine: identity, account selection, briefings, proposals, the explicit save, the undo window, policy refusals and the handoff. Imports nothing from Azure, Express or SQLite. |
   | src/providers/crm.mjs | Chooses the CRM adapter, and the rep directory that verifies callers (Teams user, or registered mobile plus PIN). |
   | src/db.mjs, src/crm-memory.mjs, src/crm-shared.mjs | The CRM: SQLite (default) or in-memory, with commits and field-level audit rows for undo; shared rules such as which accounts a rep may see. |
   | src/handoff.mjs | How the Sales Operations target is addressed, and the VoIP headers that carry the context to Teams. |
   | src/agent.mjs | The tools the AI may call and its standing instructions. |
   | src/offline.mjs | A keyword stand-in for the AI, used only in simulation mode. |
   | src/voice/acs.mjs | ACS Call Automation: answer, transfer to Teams, keypad, hang up. |
   | src/voice/voice-live.mjs, src/voice/call-bridge.mjs | The Voice Live session, and the bridge that joins a call’s audio to it and passes tool calls to the flow. |
   | src/server.mjs | Event Grid intake, ACS callbacks, the simulation and console APIs, /health and the WebSockets. |
   | public/ | The browser console you use in simulation and to watch live calls. |

   ![](images/image24.png)

4. Open the files in **config/**. They are the whole policy and the fictional data; changing them is a configuration change, not a code change:

   | **File** | **Holds** |
   |---|---|
   | reps.json | The reps: name, Entra and Teams IDs, registered mobile, PIN hash and the CRM owner ID their accounts are filed under. Alex Morgan’s PIN is 246810. |
   | crm-seed.json | 8 accounts, contacts, opportunities, activities and tasks, loaded into the SQLite CRM on first run |
   | writable-fields.json | The allow-list: which fields may be created or updated by phone, and their limits |
   | picklists.json | Allowed values, such as opportunity stages and activity types |
   | routing.json | The Sales Operations transfer target (a Teams call queue or user) |
   | briefing.json | What a briefing includes |

   **Note:** The routing target and the reps’ IDs are placeholders. That is fine for simulation. In Exercise 6 you create local copies with your own phone number and Teams user.

   ![](images/image25.png)

**Exercise conclusion:**

The project is open, its packages are installed and all 53 tests pass. You know which file owns which job and where the policy and the fictional CRM live.

## Exercise 2: Understand the key code

**Overview:**

In this exercise, you read the most important parts of the code. Each task shows a short excerpt and explains it. You do not need to change anything: open the file in VS Code (Ctrl+P, then type the file name) and find the code with Ctrl+F to see it in context.

### Task 1: Verified before anything is read

**Overview:**

See how the server identifies the caller before any record is opened.

1. Open **src/providers/crm.mjs** and find **beginPstn** and **verifyPin** in the RepDirectory class.

   **src/providers/crm.mjs — RepDirectory**

   ```javascript
     beginPstn(phone) {
       const rep = this.reps.find((r) => normalisePhone(r.registeredMobile) === normalisePhone(phone));
       return rep ? { repId: rep.id, displayName: rep.displayName, firstName: rep.firstName, needsPin: true } : null;
     }

     verifyPin(repId, pin) {
       const rep = this.reps.find((r) => r.id === repId);
       if (!rep || !verifyPin(pin, rep.pinHash)) return null;
       return publicRep(rep, { verifiedBy: "pstn-pin" });
     }
   ```

   ![](images/image26.png)

2. Open **src/flow.mjs** and find **submitPin**.

   **src/flow.mjs — submitPin()**

   ```javascript
     submitPin(callId, pin) {
       const call = this.#require(callId);
       if (!call.pendingPstnRep) return { ok: false, reason: "no_pin_pending" };
       const digits = String(pin ?? "").replace(/\D/g, "");
       call.pinDigits = "";
       const rep = this.reps.verifyPin(call.pendingPstnRep.repId, digits);
       if (!rep) return this.#verificationFailed(call, "wrong_pin");
       return this.#identify(call, rep);
     }
   ```

**What to notice:**

- A call from a registered mobile is only a hint: beginPstn finds the rep, but the caller is not verified until the six-digit PIN matches the stored SHA-256 hash. A Teams caller is matched by Teams user ID instead.

- The PIN is entered on the keypad and checked by the server; the AI never hears or handles it.

- A wrong PIN opens no record and reads nothing out (#verificationFailed): the call goes to Sales Operations with only the last digits of the number and the failed check. A number that is not on the rep list is told the line is for Contoso sales staff, and the call ends.

  ![](images/image27.png)

**Conclusion:**

Identity is decided by the server from the rep list and the PIN before anything is read out.

### Task 2: Only what this caller may see

**Overview:**

See which accounts a rep may open and what stays off the card.

1. Open **src/crm-shared.mjs** and find **canSee** and **maskContact**.

   **src/crm-shared.mjs — canSee() and maskContact()**

   ```javascript
   export function canSee(rep, account) {
     return Boolean(rep?.ownerId && account && (account.ownerId === rep.ownerId || account.teamOwnerIds?.includes(rep.ownerId)));
   }

   /** Contacts as the model sees them by default: no phone, no email. */
   export function maskContact(contact) {
     return contact && { id: contact.id, accountId: contact.accountId, name: contact.name, title: contact.title, isKey: contact.isKey };
   }
   ```

**What to notice:**

- A rep may open only accounts they own or are on the account team for. Alex can open Fabrikam, but not Tailspin Toys, which belongs to Jordan alone.

- By default a contact reaches the AI without phone or email. A number is read out only when the rep asks for it (get_contact_info), and a read-back names only its last four digits.

- Two misses in a call — an account not found, or a record the rep may not open — send the rep to a person.

  ![](images/image28.png)

**Conclusion:**

What a caller may hear is decided by ownership rules on the server, not by the AI.

### Task 3: Which fields may change — the allow-list

**Overview:**

See the write policy and how every proposed value is checked against it.

1. Open **config/writable-fields.json**.

   **config/writable-fields.json**

   ```json
   {
     "activity": {
       "create": {
         "subject": { "type": "text", "maxLength": 120 },
         "type": { "type": "picklist", "picklist": "activityType" },
         "notes": { "type": "text", "maxLength": 1000 },
         "date": { "type": "date" },
         "contacts": { "type": "contact-list" }
       }
     },
     "opportunity": {
       "update": {
         "stage": { "type": "picklist", "picklist": "opportunityStage" },
         "amount": { "type": "currency", "min": 0, "max": 10000000 },
         "closeDate": { "type": "date", "notBefore": "today" },
         "nextStep": { "type": "text", "maxLength": 200 }
       }
     },
     "task": {
       "create": {
         "subject": { "type": "text", "maxLength": 120 },
         "dueDate": { "type": "date", "withinDays": 90 },
         "owner": { "type": "fixed", "value": "me" }
       }
     },
     "contact": {
       "update": {
         "phone": { "type": "phone" },
         "email": { "type": "email" }
       }
     }
   }
   ```

   ![](images/image29.png)

2. Open **src/flow.mjs** and find **validateField**.

   **src/flow.mjs — validateField()**

   ```javascript
   export function validateField(entity, operation, field, value, { allowlist, picklists, today }) {
     const rule = allowlist?.[entity]?.[operation]?.[field];
     if (!rule) return { reason: "not_allowlisted" };
     switch (rule.type) {
       case "picklist":
         return picklists?.[rule.picklist]?.some((p) => p.value.toLowerCase() === String(value).toLowerCase()) ? null : { reason: "invalid_picklist" };
       case "currency": {
         const n = Number(value);
         return Number.isFinite(n) && n >= (rule.min ?? 0) && n <= (rule.max ?? Infinity) ? null : { reason: "out_of_bounds", limit: rule.max };
       }
       case "text":
         return String(value ?? "").length <= (rule.maxLength ?? Infinity) ? null : { reason: "too_long", limit: rule.maxLength };
       case "date":
         if (!/^\d{4}-\d{2}-\d{2}$/.test(String(value))) return { reason: "invalid_date" };
         if ((rule.notBefore === "today" || rule.withinDays) && value < today) return { reason: "date_in_past" };
         if (rule.withinDays && daysBetween(today, value) > rule.withinDays) return { reason: "date_too_far", limit: rule.withinDays };
         return null;
       case "phone":
         return /^\+1\d{10}$/.test(String(value)) ? null : { reason: "invalid_phone" };
       case "email":
         return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(String(value)) ? null : { reason: "invalid_email" };
       case "fixed":
         return value === rule.value ? null : { reason: "not_allowlisted" };
       default:
         return null;
     }
   }
   ```

**What to notice:**

- A dictated update becomes a proposal: an activity, deal changes and a task. Each value is checked against the allow-list: a stage must be on the picklist, an amount within limits, a close date not in the past, a task due within 90 days, a phone number in a valid format.

- A value outside its limits is left out of the read-back with a reason, for example “The close date can’t be in the past, so I left it alone”.

- Adding a writable field is a configuration change in writable-fields.json, not a prompt rewrite.

  ![](images/image2a.png)

**Conclusion:**

Every value the agent might write is checked by the server against a configured allow-list.

### Task 4: Not mine to change — policy refusals

**Overview:**

See the requests the agent never writes, whatever the AI makes of them.

1. In **src/flow.mjs**, find **POLICY_REFUSALS** near the top.

   **src/flow.mjs — POLICY_REFUSALS**

   ```javascript
   const POLICY_REFUSALS = [
     { entity: "opportunity", field: "discount", label: "the discount", pattern: /\bdiscount|\bpricing\b|\bprice cut\b|\bpercent off\b/ },
     { entity: "account", field: "owner", label: "the account owner", pattern: /\breassign|\bowner\b|\bgive .* account to\b|\bassign .* to\b/ },
     { entity: "account", field: "delete", label: "deleting an account", pattern: /\bdelete\b|\bremove the account\b/ },
     { entity: "account", field: "merge", label: "merging accounts", pattern: /\bmerge\b/ },
     { entity: "account", field: "creditLimit", label: "the credit limit", pattern: /\bcredit (?:limit|line)\b/ },
     { entity: "account", field: "paymentTerms", label: "the payment terms", pattern: /\bnet[ -]?\d{2}\b|\b(?:payment|billing|invoice) terms\b/ },
   ];
   ```

**What to notice:**

- Discounts and pricing, a new account owner, deleting or merging accounts, credit limits and payment terms are refused: nothing is written.

- The agent offers to put the rep through to Sales Operations, and the refused request travels in the handoff so the person who answers knows what was asked.

  ![](images/image2b.png)

**Conclusion:**

Sensitive changes are refused by server policy and handed to a person.

### Task 5: Read back, “yes, save it”, then the undo window

**Overview:**

See the explicit save, the undo window and undo after the write.

1. In **src/flow.mjs**, find **isExplicitSave**.

   **src/flow.mjs — isExplicitSave()**

   ```javascript
   export function isExplicitSave(text) {
     const lower = String(text ?? "").toLowerCase();
     if (/\bmaybe\b|\bi think\b|\bnot\b|\bdon'?t\b|\bdo not\b|\bnever\b/.test(lower)) return false;
     return /\byes[,.!]?\s+(?:please\s+)?save it\b|\byes[,.!]?\s+please save\b|\bgo ahead and save\b|\bsave it\b/.test(lower);
   }
   ```

   ![](images/image2c.png)

2. Find **commitProposal**.

   **src/flow.mjs — commitProposal()**

   ```javascript
     commitProposal(callId, proposalId, confirmation) {
       const call = this.#requireIdentified(callId);
       if (call.pendingCommit) return this.holdForWindow(callId);
       const proposal = this.#proposalFor(call, proposalId);
       if (!proposal) {
         const phrase = "There's nothing waiting to be saved.";
         this.#remember(call, phrase);
         this.#touch(call);
         return { ok: false, reason: "no_proposal", phrase };
       }
       if (!isExplicitSave(confirmation)) {
         const phrase = `Say "yes, save it" if you want me to write this to ${possessive(proposal.accountName)} record.`;
         this.#remember(call, phrase);
         this.#touch(call);
         return { ok: false, reason: "not_confirmed", message: phrase, phrase };
       }
       if (!hasWrite(proposal)) return { ok: false, reason: "nothing_to_write" };
       const windowMs = Math.max(0, Number(this.options.commitUndoMs) || 0);
       if (!windowMs) return this.#applyCommit(call, proposal);
       const startedAt = this.now();
       call.pendingCommit = {
         proposalId: proposal.id,
         startedAt,
         commitsAt: startedAt + windowMs,
         windowMs,
         startedPerf: performance.now(),
         timer: setTimeout(() => this.#closeWindow(call.id), windowMs),
         waiters: [],
       };
       this.#event(call, "agent", "commit_pending", `${Math.round(windowMs / 100) / 10}s undo window`);
       const phrase = `Saving in ${spokenSeconds(windowMs)}. Say "wait" to stop it.`;
       this.#enter(call, STATES.COMMITTING);
       this.#remember(call, phrase);
       return { ok: true, pending: true, proposalId: proposal.id, commitsAt: iso(startedAt + windowMs), windowMs, phrase };
     }
   ```

   ![](images/image2d.png)

3. Find **cancelPendingCommit** and **undoLastCommit**.

   **src/flow.mjs — cancelPendingCommit()**

   ```javascript
     cancelPendingCommit(callId, reason = "caller_changed_mind") {
       const call = this.#requireIdentified(callId);
       const pending = call.pendingCommit;
       if (!pending) {
         if (call.lastWrite?.status !== "committed") return { ok: false, reason: "nothing_pending" };
         const phrase = `That's already saved. Say "undo that" and I'll take it back.`;
         this.#remember(call, phrase);
         this.#touch(call);
         return { ok: false, reason: "already_committed", phrase };
       }
       const remainingMs = this.#remaining(call);
       clearTimeout(pending.timer);
       call.pendingCommit = null;
       const proposal = this.proposals.get(pending.proposalId) ?? null;
       this.proposals.delete(pending.proposalId);
       if (call.currentProposalId === pending.proposalId) call.currentProposalId = null;
       const accountName = proposal?.accountName ?? call.accountName;
       call.lastWrite = { commitId: null, status: "cancelled", accountName, rows: proposal?.rows ?? [], auditRows: 0, reason, remainingMs, at: iso(this.now()) };
       this.#event(call, "caller", "commit_cancelled", `${(remainingMs / 1000).toFixed(1)}s left · nothing written`);
       this.#refreshRecord(call);
       const phoneDiff = onlyPhoneChange(proposal);
       const phrase = phoneDiff
         ? `Stopped. Nothing was written: ${possessive(phoneDiff.name)} number is still the one ending ${phoneTail(phoneDiff.before)}.`
         : `Stopped. Nothing was written to ${possessive(accountName)} record.`;
       this.#remember(call, phrase);
       this.#rest(call);
       for (const resolve of pending.waiters) resolve({ ok: true, settled: "cancelled" });
       return { ok: true, cancelled: true, wrote: false, remainingMs, phrase };
     }
   ```

   ![](images/image2e.png)

   **src/flow.mjs — undoLastCommit()**

   ```javascript
     undoLastCommit(callId) {
       const call = this.#requireIdentified(callId);
       if (call.pendingCommit) return this.cancelPendingCommit(callId, "undo_in_window");
       const say = (reason, phrase, extra = {}) => {
         this.#remember(call, phrase);
         this.#touch(call);
         return { ok: false, reason, phrase, ...extra };
       };
       if (call.lastWrite?.status === "undone") return say("already_reverted", "That's already undone.");
       if (!call.lastCommitId || call.lastWrite?.status !== "committed") return say("nothing_to_undo", "There's nothing from this call to undo.");
       const result = this.crm.revertCommit(call.rep, call.lastCommitId);
       if (!result.ok) {
         this.#event(call, "agent", "undo_failed", result.reason.replace(/_/g, " "));
         if (result.reason === "conflict") return say("conflict", `I can't undo that: someone has changed the ${FIELD_NAMES[result.field] ?? result.field} since. Nothing was changed.`, { field: result.field });
         return say(result.reason, result.reason === "already_reverted" ? "That's already undone." : "I can't undo that one.");
       }
       this.#event(call, "caller", "undo", `audit ${call.lastCommitId.slice(0, 8)}`);
       call.lastWrite = { ...call.lastWrite, status: "undone", at: iso(this.now()) };
       this.#refreshRecord(call);
       const phrase = `Undone. ${possessive(call.lastWrite.accountName)} record is back to how it was.`;
       this.#remember(call, phrase);
       this.#rest(call);
       return { ok: true, commitId: result.commitId, phrase };
     }
   ```

**What to notice:**

- A bare “yes” is not enough, and “maybe” or “don’t” never counts: only “yes, save it” (or similar) starts a save.

- The save then waits COMMIT_UNDO_MS (6 seconds) on a server timer: “Saving in six seconds. Say ‘wait’ to stop it.” Only when the window closes does the write happen, with field-level audit rows.

- “Wait” inside the window cancels it: “Stopped. Nothing was written.”

- After the write, “undo that” reverts the commit — unless someone has changed the field since, in which case the CRM refuses the undo (conflict) and nothing changes.

  ![](images/image2f.png)

**Conclusion:**

A write needs a read-back, an explicit save and an undo window that runs out, and it can still be undone safely afterwards.

### Task 6: What the AI is allowed to do

**Overview:**

See the AI’s tools and how each one is handed to the server.

1. Open **src/voice/call-bridge.mjs** and find **#invokeTool**.

   **src/voice/call-bridge.mjs — tool mapping**

   ```javascript
         case "find_account": return this.flow.findAccount(this.callId, args.utterance);
         case "select_account": return this.flow.selectAccount(this.callId, args.accountId);
         case "get_briefing": return this.flow.getBriefing(this.callId, args.accountId);
         case "get_detail": return this.flow.getDetail(this.callId, args.handle);
         case "get_contact_info": return this.flow.getContactInfo(this.callId, args.contactId, args.field);
         case "propose_update": return this.flow.proposeUpdate(this.callId, args.accountId, args.dictation);
         case "amend_proposal": return this.flow.amendProposal(this.callId, args.proposalId, args.change);
         case "commit_proposal": return this.flow.commitProposal(this.callId, args.proposalId, args.confirmation);
         case "cancel_pending_commit": return this.flow.cancelPendingCommit(this.callId, "caller_changed_mind");
         case "undo_last_commit": return this.flow.undoLastCommit(this.callId);
         case "save_draft": return this.flow.saveDraft(this.callId, args.proposalId, "agent_requested");
         case "resume_draft": return this.flow.resumeDraft(this.callId);
         case "pause": return this.flow.pause(this.callId);
         // The model's free-text reason is not trusted into the handoff; the flow knows why.
         case "transfer": return this.flow.transfer(this.callId, "explicit_request");
         case "repeat_last": return this.flow.repeatLast(this.callId);
   ```

**What to notice:**

- Each tool maps to one flow method; the bridge makes no decisions of its own.

- No tool has a parameter for a Teams object ID, a PIN, a number to dial or a URL: the AI can neither vouch for a caller nor choose where a call goes.

- propose_update passes the dictation as text; the server builds the proposal and the read-back. commit_proposal passes the rep’s words, so the server, not the AI, decides whether they said “yes, save it”.

- The transfer reason the AI might give is ignored: the flow knows why it is transferring.

  ![](images/image30.png)

**Conclusion:**

The AI holds the conversation through narrow tools; every decision behind them is the server’s.

### Task 7: The handoff to Sales Operations and the transfer fix

**Overview:**

See what Teams receives on a transfer, and the fix that makes transfers work on an ACS phone number.

1. In **src/flow.mjs**, find **handoffContext**.

   **src/flow.mjs — handoffContext()**

   ```javascript
     handoffContext(call, reason = "explicit_request") {
       const base = { sessionId: call.sessionId ?? call.id, routeId: "sales-operations", target: this.#targetName() };
       if (!call.rep) {
         const tail = phoneTail(call.fromPhone);
         return {
           ...base,
           callTopic: "Unverified caller – no records opened",
           callContext: JSON.stringify({
             verified: false,
             reason,
             callerId: tail ? `number ending ${tail}` : "withheld",
             check: call.callerIdMatched ? `Caller ID matched ${call.claimedRepName}'s registered mobile; ${call.pinFailed ? "PIN failed" : "PIN not entered"}` : "Caller ID is not on the rep list",
             note: "No records were opened or read out.",
           }),
         };
       }
       const policyLabels = unique(
         (call.offeredHandoff?.labels ?? call.lastRefusal?.items?.filter((i) => i.reason === "not_allowlisted").map((i) => i.label) ?? []).map(withoutArticle),
       );
       const topic = reason === "refused_by_policy" && policyLabels.length ? `${call.accountName ?? "Sales rep"} – ${policyLabels.join(", ")}` : `Sales rep – ${call.accountName ?? "CRM assistance"}`;
       const proposal = (call.currentProposalId && this.proposals.get(call.currentProposalId)) || (call.pendingCommit && this.proposals.get(call.pendingCommit.proposalId)) || null;
       const context = {
         rep: call.rep.displayName,
         verifiedBy: call.verifiedBy === "pstn-pin" ? "registered mobile + PIN" : "Teams sign-in",
         account: call.accountName,
         reason,
         request: call.offeredHandoff?.request ?? null,
         refusedRequests: unique(call.refusedItems.map(withoutArticle)),
         proposal: proposal ? redact(proposal.readBack, 300) : null,
         lastWrite: lastWriteSummary(call.lastWrite),
         draftId: this.crm.latestDraftHint?.(call.rep)?.draftId ?? null,
       };
       if (JSON.stringify(context).length > 1000 && context.proposal) context.proposal = clip(context.proposal, 120);
       if (JSON.stringify(context).length > 1000) context.request = null;
       return { ...base, callTopic: clip(topic, 48), callContext: JSON.stringify(context) };
     }
   ```

**What to notice:**

- For an unverified caller, only the last digits of the number and the failed check travel, with “No records were opened or read out.”

- For a verified rep: who they are and how they were verified, the account, the reason, any refused requests, a pending proposal (masked) and the last write.

- The topic is clipped to 48 characters and reads, for example, “Fabrikam – credit limit, payment terms”. Note the en dash (–): it matters for the fix below.

  ![](images/image31.png)

2. Open **src/voice/acs.mjs** and find **transferToTeams**. Add the following asciiSafe() function before transferToTeams(). And replace the existing transferToTeams() function block with the following transferToTeams() function. Press CTRL+S to save the changes.

   **src/voice/acs.mjs — asciiSafe() and transferToTeams()**

   ```javascript
   /** Lab fix (ACS phone number): transfers reject dotted VoIP header keys and non-ASCII values. */
   const asciiSafe = (headers) =>
     headers.map((h) => ({
       ...h,
       key: String(h.key).replace(/[^A-Za-z0-9]/g, ""),
       value: String(h.value)
         .replace(/[\u2013\u2014]/g, "-")
         .replace(/[\u201C\u201D]/g, '"')
         .replace(/[\u2018\u2019]/g, "'")
         .replace(/\u2026/g, "...")
         .replace(/[^\x20-\x7E]/g, ""),
     }));

   /**
    * Hand the call to Teams and step out of the way.
    *
    * A blind transfer is deliberate for this template: the voice agent's job ends
    * once the context has been delivered, and staying on the line would mean a
    * silent third party on every routed call.
    */
   export async function transferToTeams({ callConnectionId, target, context, callId }) {
     if (!callConnectionId) throw new Error("no call connection to transfer");

     const connection = (await acsClient()).getCallConnection(callConnectionId);
     try {
       await connection.transferCallToParticipant(teamsIdentifier(target), {
         operationContext: `transfer:${callId}`,
         customCallingContext: asciiSafe(buildCustomCallingContext(context)),
       });
     } catch (e) {
       log("TRANSFER ERROR", "status:", e.statusCode, "| code:", e.code, "| message:", e.message);
       log("TRANSFER ERROR body:", e.response?.bodyAsText ?? "(none)");
       throw e;
     }

     log("transfer requested", callId, "->", target.displayName ?? target.objectId);
   }
   ```

   **Transfer fix:** Callers reach this agent on a phone number bought in ACS. On this kind of number, ACS rejects a transfer whose VoIP header names contain dots (CallDetails.CallTopic) or whose values contain non-ASCII characters, such as the en dash in “Fabrikam – credit limit, payment terms”. The transfer then fails with “Invalid CustomCallingContext in request”. asciiSafe() removes the dots from the header names (CallDetailsCallTopic) and replaces or removes non-ASCII characters in the values just before the transfer. The try/catch writes the full ACS error to the server log (TRANSFER ERROR) if a transfer is ever refused.

   ![](images/image32.png)

   ![](images/image33.png)

**Conclusion:**

Transfers carry a safe, minimal context to Sales Operations, with headers ACS accepts on an ACS phone number.

**Exercise conclusion:**

You have read the code behind every important decision: verification, visibility, the allow-list, policy refusals, the explicit save with an undo window, the AI’s tools and the handoff with the transfer fix.

## Exercise 3: Test the agent in simulation

**Overview:**

In this exercise, you run the complete agent in the browser console before connecting any Azure services. A typed call drives the same state machine a real call does — the same identity checks, allow-list, undo window and handoff. Only the AI and the audio are replaced by a keyword stand-in. The demo clock starts on Wednesday, June 10, 2026, at noon Pacific, so “today” and “Friday” mean the same thing on every run.

1. Start the server.

   ```powershell
   npm start
   ```

   **Expected output**

   ```text
   [bridge] ACS media bridge listening on /ws/media
   Customer record lookup & update demo on http://127.0.0.1:8097
     CRM adapter        : sqlite
     reps               : 2
     undo window        : 6s
     audit              : sqlite
     SIMULATION MODE    — set ACS_ENDPOINT, PUBLIC_BASE_URL, VOICE_LIVE_ENDPOINT to answer real calls
     demo clock         : 2026-06-10T19:00:00… (demo-day)
   ```

   ![](images/image34.png)

2. Open **http://127.0.0.1:8097** in the browser and click **Reset demo**.

   ![](images/image5.png)

3. Play the scripted demo: press **Next ▶** (or the → key) to run each of the five beats: verified before anything is read; read back, a yes, then the undo window; “Wait — keep the old number”; not mine to change; and can’t verify, won’t disclose. Watch the checklist, the record card, the write-back panel, the Teams toast and the audit trail.

   ![](images/image6.png)

   ![](images/image7.png)

   ![](images/image8.png)

   ![](images/image9.png)

   ![](images/imagea.png)

   ![](images/imageb.png)

   ![](images/imagec.png)

   ![](images/imaged.png)

   ![](images/imagee.png)

   ![](images/imagef.png)

   ![](images/image10.png)

   ![](images/image11.png)

   ![](images/image12.png)

   ![](images/image13.png)

   ![](images/image14.png)

   ![](images/image15.png)

   ![](images/image16.png)

   ![](images/image17.png)

   ![](images/image18.png)

4. Select **Reset Demo**.

   ![](images/image35.png)

5. Now drive one call yourself. Pick the Caller ID **Alex (Teams)**, click **Answer a call**, and type each line in order, clicking **Say** after each:

   | **Type this** | **Expected result** |
   |---|---|
   | Brief me on Fabrikam | Tier, last touch, three open deals, a task due today and the key contact — with no phone number |
   | What’s Dana’s number? | The number is read out because the rep asked, and only now appears on the card |
   | Log the Fabrikam meeting. Dana agreed to the renewal at 240k, moving to Closed Won pending signature, close date June 15th. Remind me to send the contract Friday. | A read-back of a meeting activity, four deal changes and a task due Friday, June 12, then “Save it?” |
   | Yes | Not enough: the agent asks for “yes, save it”, and nothing is written |
   | Yes, save it. | “Saving in six seconds”, a countdown, then “Saved to Fabrikam’s record.” with before → after and audit rows |
   | Undo that | “Undone. Fabrikam’s record is back to how it was.” |
   | Update Fabrikam: Dana has a new mobile, 425 555 0143. → Yes, save it. → wait | Stopped inside the window: nothing written, and the number still ends 0120 |
   | Raise Fabrikam’s credit limit to 500k and move them to net 60 payment terms. → Yes, put me through. | Refused by policy with nothing written, then a handoff to Contoso Sales Operations with the details in the VoIP headers |

   ![](images/image19.png)

   ![](images/image1a.png)

   ![](images/image1b.png)

   ![](images/image1c.png)

   ![](images/image1d.png)

   ![](images/image1e.png)

   ![](images/image1f.png)

   ![](images/image36.png)

   ![](images/image37.png)

   ![](images/image38.png)

   ![](images/image39.png)

   ![](images/image3a.png)

   ![](images/image3b.png)

   ![](images/image3c.png)

6. Select **Reset Demo**.

   ![](images/image3d.png)

7. Then try each of these on a new call:

   | **Caller ID and input** | **Expected result** |
   |---|---|
   | Alex (Teams): I need a person | Straight to Contoso Sales Operations, no questions asked |
   | Alex (Teams): Brief me on Tailspin, twice | Another rep’s private account stays closed; after the second miss, a person |
   | Alex (mobile): the keypad’s PIN wrong key (135790) | No record opened, nothing read out, a safe handoff |
   | Alex (mobile): the keypad’s PIN Alex’s key (246810) | “Thanks, Alex, you’re verified. Which account?” |
   | Enter any Unknown number | The line is for Contoso sales staff; the call ends politely |

   **Note:** The undo window is six seconds, so type just “wait” to stop it in time.

   ![](images/image3e.png)

   ![](images/image3f.png)

   ![](images/image40.png)

   ![](images/image41.png)

   ![](images/image42.png)

   ![](images/image43.png)

   ![](images/image44.png)

   ![](images/image45.png)

   ![](images/image46.png)

   ![](images/image47.png)

8. For each call, read the identity checklist, the record card, the write-back panel and the decision trail, and name the file and function from Exercise 2 that made each decision.

9. Press Ctrl+C in the PowerShell window to stop the server.

**Exercise conclusion:**

You have seen every rule fire in simulation — verification, visibility, the explicit save, the undo window, undo, refusals and the safe handoff — and traced each one to the code.

## Exercise 4: Create the Azure resources

**Overview:**

In this exercise, you create the Azure resources the live agent needs — an Azure Communication Services resource and an Azure AI Services resource for Voice Live — and give your sign-in the roles for keyless access. A small variables file keeps every name, number, endpoint and ID consistent across PowerShell windows.

### Task 1: Create the lab variables file

**Overview:**

Create lab-vars.ps1, a small file that holds your names and phone number and fetches every endpoint and ID from Azure, and load it into your PowerShell session.

1. In VS Code, create a new file named **lab-vars.ps1** in the code folder (next to package.json) and paste the below content.

   ```powershell
   # lab-vars.ps1 - load it in every PowerShell 7 window with:   . .\lab-vars.ps1
   # ---------------------------------------------------------------- EDIT
   $TENANT  = "<yourtenant>.onmicrosoft.com"
   $SUFFIX  = "<initials><nn>"                               # for example sv01 - keeps your names unique
   $LOC     = "eastus2"                                      # a region that serves Voice Live (gpt-realtime)
   $AGENT   = "<teams-user>@<yourtenant>.onmicrosoft.com"    # the Teams user who acts as Sales Operations
   $MYPHONE = "+<country code><your mobile number>"          # the phone you call from, e.g. +919876543210
   # ------------------------------------------------- derived (do not edit)
   $TUNNEL = "crmline-$SUFFIX"
   $RG  = "rg-crmline-$SUFFIX"; $ACS = "acs-crmline-$SUFFIX"; $AI = "ai-crmline-$SUFFIX"
   $ACSID        = az communication show -n $ACS -g $RG --query id -o tsv 2>$null
   $ACS_ENDPOINT = if ($ACSID) { "https://" + (az communication show -n $ACS -g $RG --query hostName -o tsv) }
   $IMMUTABLE    = if ($ACSID) { az communication show -n $ACS -g $RG --query immutableResourceId -o tsv }
   $AISUB        = az cognitiveservices account show -n $AI -g $RG --query properties.customSubDomainName -o tsv 2>$null
   $VL_ENDPOINT  = if ($AISUB) { "https://$AISUB.services.ai.azure.com" }
   $AGENT_ID     = az ad user show --id $AGENT --query id -o tsv 2>$null
   Write-Host "Resource group : $RG"
   Write-Host "ACS endpoint   : $ACS_ENDPOINT"
   Write-Host "ACS immutable  : $IMMUTABLE"
   Write-Host "Voice Live     : $VL_ENDPOINT"
   Write-Host "Teams user ID  : $AGENT_ID"
   Write-Host "Your phone     : $MYPHONE"
   Write-Host "Tunnel         : $TUNNEL"
   # ------------------------------------- added after the tunnel starts
   ```

   ![](images/image48.png)

2. Replace the five values in the EDIT section with your own. $MYPHONE is the mobile number you will call from, in international format: a plus sign, the country code and the number, with no spaces. Keep the quotes and remove the < > brackets.

   Note: you can find Tenant Id from: Go to Azure portal ->Login using the given credentials->Microsoft Entra ID->Overview page->Tenant ID.

   ![](images/image49.png)

3. Save the file (Ctrl+S).

4. In the PowerShell 7 window, execute the below command to load the variables.

   ```powershell
   . .\lab-vars.ps1
   az login --tenant $TENANT --use-device-code
   ```

   Resource group must show rg-crmline-<your suffix>, not the text <suffix>. If it still shows <suffix>, the EDIT section was not saved.

   **Note:** Blank endpoint and ID values after the first load are normal — the resources do not exist yet. Reload the file after each step that creates something.

   ![](images/image4a.png)

   ![](images/image4b.png)

   ![](images/image4c.png)

   ![](images/image4d.png)

   ![](images/image4e.png)

   ![](images/image4f.png)

   ![](images/image50.png)

**Conclusion:**

lab-vars.ps1 holds your names and phone number and loads into the session.

### Task 2: Sign in and check permissions

**Overview:**

Sign in to Azure, check your permissions and register the services the lab uses.

1. Copy the below code and paste it in the PowerShell to sign in and check the available permissions.

   ```powershell
   . .\lab-vars.ps1
   az login --tenant $TENANT --use-device-code
   az account show --query "{user:user.name, tenant:tenantId, subscription:name}" -o table
   $ME = az ad signed-in-user show --query id -o tsv
   az role assignment list --assignee $ME --all --query "[].{role:roleDefinitionName, scope:scope}" -o table
   az provider register -n Microsoft.Communication
   az provider register -n Microsoft.CognitiveServices
   az provider register -n Microsoft.EventGrid
   az extension add --name communication --upgrade
   ```

2. Select **Paste anyway** when you get a warning about multiple lines of code.

3. Follow the prompts to sign in.

You need a paid subscription and Owner (or Contributor + User Access Administrator) to create resources and assign roles. Check the region against the Voice Live supported models and regions table.

![](images/image51.png)

![](images/image52.png)

![](images/image53.png)

![](images/image54.png)

![](images/image55.png)

![](images/image56.png)

![](images/image57.png)

**Conclusion:**

You are signed in to the lab tenant with the required permissions, and the resource providers are registered.

### Task 3: Create the resources

**Overview:**

Create the resource group, the Azure Communication Services resource and the Azure AI Services resource.

1. Copy and paste the below block of code in the PowerShell to create the resources in Azure.

   ```powershell
   az group create -n $RG -l $LOC
   az communication create -n $ACS -g $RG --location global --data-location UnitedStates
   az cognitiveservices account create -n $AI -g $RG -l $LOC --kind AIServices --sku S0 --custom-domain $AI --yes
   ```

   ![](images/image58.png)

   ![](images/image59.png)

**Conclusion:**

The resource group, ACS resource and AI Services resource are created.

### Task 4: Assign roles to your sign-in

**Overview:**

Give your sign-in the roles it needs to use ACS and Voice Live without keys, and confirm the endpoints.

1. Copy and paste the below block of code in the PowerShell.

   ```powershell
   . .\lab-vars.ps1                 # reload: now fetches the new IDs
   $ME   = az ad signed-in-user show --query id -o tsv
   $AIID = az cognitiveservices account show -n $AI -g $RG --query id -o tsv
   az role assignment create --assignee $ME --role "Communication and Email Service Owner" --scope $ACSID
   az role assignment create --assignee $ME --role "Cognitive Services User" --scope $AIID
   az role assignment create --assignee $ME --role "Foundry User" --scope $AIID
   ```

   ![](images/image5a.png)

   ![](images/image5b.png)

2. Execute `. .\lab-vars.ps1` again. It now prints an ACS endpoint ending in .communication.azure.com and a Voice Live endpoint ending in .services.ai.azure.com (never /models).

   ![](images/image5c.png)

   ![](images/image5d.png)

3. Role assignments can take a few minutes to apply.

**Conclusion:**

The roles are assigned and lab-vars.ps1 now returns both endpoints.

**Exercise conclusion:**

Your Azure resources exist, the roles are assigned, and lab-vars.ps1 fetches the ACS and Voice Live endpoints automatically.

## Exercise 5: Get a US phone number in ACS

**Overview:**

In this exercise, you buy a US local phone number in your Azure Communication Services resource. This is the CRM line the rep dials.

**Note:** Callers reach the agent on a phone number purchased directly in Azure Communication Services. In the server log these calls are shown as acs-direct, and the caller is identified by their mobile number and PIN. Transfers to Teams work on this kind of number because acs.mjs cleans the handoff headers before each transfer (Exercise 2, Task 7).

1. Open the Azure portal at https://portal.azure.com and sign in using the folllowing credentials:

   - Username - +++@lab.CloudPortalCredential(User1).Username+++

   - TAP Token - +++@lab.CloudPortalCredential(User1).AccessToken+++

2. Open the resource group **rg-crmline-<suffix>** and select your Communication Service from there.

   ![](images/image5e.png)

3. Select **Phone numbers** under **Telephony and SMS** and then select **+ Get**.

   ![](images/image5f.png)

4. Select the below values.

- Country/region: United States.

- Number type: Local (geographic).

- Calling: Receive calls selected. SMS: None.

  ![](images/image60.png)

5. Add a number to the cart and then select **Next**.

   ![](images/image61.png)

   ![](images/image62.png)

6. Complete the purchase: select **Buy now**. Make a note of this number in +1 format (for example, +1 XXX XXX XXXX). You will dial it later.

   ![](images/image63.png)

**Exercise conclusion:**

You have a phone number that can receive calls. It reaches the agent once the server is live and the Event Grid subscription is in place (Exercise 7).

## Exercise 6: Register your phone and set up transfers to Teams

**Overview:**

In this exercise, you register your own mobile as Alex Morgan’s phone, point Sales Operations at your Teams user, enable that user for calling, and allow your ACS resource to call Teams users.

### Task 1: Sign in to Teams and enable the Teams user

**Overview:**

Sign in to Teams PowerShell and enable the Teams user who acts as Sales Operations for Teams calling.

1. The Teams user needs a Teams Phone licence (for example Teams Phone Standard or Microsoft 365 E5) and Teams. Copy and paste the below code in the PowerShell window.

   ```powershell
   . .\lab-vars.ps1
   $env:PSModulePath = "$HOME\PSModules;" + $env:PSModulePath
   Import-Module MicrosoftTeams
   Connect-MicrosoftTeams -UseDeviceAuthentication     # sign in as the tenant's Teams admin
   Get-CsOnlineUser -Identity $AGENT | Format-List DisplayName, FeatureTypes, EnterpriseVoiceEnabled, TeamsUpgradeEffectiveMode
   Set-CsPhoneNumberAssignment -Identity $AGENT -EnterpriseVoiceEnabled $true
   $AGENT_ID                                            # the object ID used next
   ```

   ![](images/image64.png)

   ![](images/image65.png)

   ![](images/image66.png)

   ![](images/image67.png)

   ![](images/image68.png)

   ![](images/image69.png)

2. Ensure that FeatureTypes includes PhoneSystem, the mode is TeamsOnly, and EnterpriseVoiceEnabled becomes True (this can take a few minutes).

   ![](images/image6a.png)

**Conclusion:**

The Teams user has Teams Phone, is in Teams-only mode, and has Enterprise Voice enabled.

### Task 2: Register your phone as Alex’s, and point Sales Operations at the Teams user

**Overview:**

Create local copies of reps.json and routing.json with your phone number and your Teams user.

1. Copy the below commands and execute them from the PowerShell window. They create **config\reps.local.json** and **config\routing.local.json**; the original files stay unchanged, and in the next exercise .env points at the local copies.

   ```powershell
   $r = Get-Content config\reps.json -Raw | ConvertFrom-Json
   ($r.reps | Where-Object id -eq "rep-alex").registeredMobile = $MYPHONE
   $r | ConvertTo-Json -Depth 10 | Set-Content config\reps.local.json -Encoding utf8

   $t = Get-Content config\routing.json -Raw | ConvertFrom-Json
   $t.salesOperations.type     = "user"
   $t.salesOperations.objectId = $AGENT_ID
   $t | ConvertTo-Json -Depth 10 | Set-Content config\routing.local.json -Encoding utf8
   ```

   ![](images/image6b.png)

2. Check the result:

   ```powershell
   (Get-Content config\reps.local.json -Raw | ConvertFrom-Json).reps | Format-Table id, displayName, registeredMobile
   (Get-Content config\routing.local.json -Raw | ConvertFrom-Json).salesOperations
   ```

   rep-alex must show your $MYPHONE, and salesOperations must show type user and your $AGENT_ID. When you call from your mobile, you are Alex Morgan, and your PIN is **246810**.

   **Note:** In production Sales Operations is a Teams call queue and each rep registers their own mobile and PIN. In this lab your mobile plays Alex, and your Teams user plays Sales Operations, so you can answer the transfer yourself.

   ![](images/image6c.png)

**Conclusion:**

Your phone is registered as Alex’s, and Sales Operations points at your Teams user.

### Task 3: Allow your ACS resource to call Teams users

**Overview:**

Allow your ACS resource to call Teams users in the tenant, and confirm the settings.

1. Execute the below commands in the PowerShell window.

   ```powershell
   Set-CsTeamsAcsFederationConfiguration -Identity Global -EnableAcsUsers $true -AllowedAcsResources @{Add=$IMMUTABLE}
   Set-CsExternalAccessPolicy -Identity Global -EnableAcsFederationAccess $true
   Get-CsTeamsAcsFederationConfiguration | Format-List Identity, EnableAcsUsers, AllowedAcsResources
   Get-CsExternalAccessPolicy -Identity Global | Format-List Identity, EnableAcsFederationAccess
   ```

   ![](images/image6d.png)

2. Ensure that EnableAcsUsers is True, your $IMMUTABLE value is listed under AllowedAcsResources, and EnableAcsFederationAccess is True. Teams can take up to an hour to apply these.

   ![](images/image6e.png)

**Conclusion:**

Your ACS resource is on the tenant’s allow-list and ACS calls to Teams users are enabled.

**Exercise conclusion:**

Your phone is a registered rep mobile, Sales Operations points at your Teams user, and your ACS resource is allowed to call Teams users.

## Exercise 7: Go live: tunnel, .env, server and Event Grid

**Overview:**

In this exercise, you take the agent live. You expose your local server to Azure through a dev tunnel, create the .env configuration, start the server in live mode, check its health, and subscribe it to incoming-call events.

You use three PowerShell 7 windows, all in the code folder, and load the variables in each one:

| **Window** | **Runs** | **Keep open?** |
|---|---|---|
| 1 | Dev tunnel | Yes, for the whole session |
| 2 | Server (npm start) | Yes, for the whole session |
| 3 | az commands, checks | As needed |

```powershell
cd "C:\LabFiles\voice-ai-template-gallery\docs\templates\customer-record-lookup-update\code"
pwsh
. .\lab-vars.ps1
```

### Task 1: Start a persistent tunnel (window 1)

**Overview:**

Start a named dev tunnel so Azure can reach the server running on your machine at a stable public address.

1. Use the window you have been using so far as window 1, and execute the below code to start a tunnel on port 8097.

   ```powershell
   devtunnel user login
   devtunnel create $TUNNEL --allow-anonymous      # first time only
   devtunnel port create $TUNNEL -p 8097           # first time only
   devtunnel host $TUNNEL
   ```

   ![](images/image6f.png)

2. When prompted, select **Work or school account** and sign in with your tenant credentials, then select **No, this app only**.

   ![](images/image70.png)

3. Copy the **Connect via browser** address it prints (it looks like https://<id>-8097.<region>.devtunnels.ms).

   ![](images/image71.png)

4. Add one line to the bottom of lab-vars.ps1 in VS Code, replacing the address with yours, and save the file:

   ```powershell
   $PUBLIC = "<paste the Connect via browser address, no trailing slash>"
   ```

   **Note:** A named tunnel keeps the same address every time you restart it, so .env and Event Grid never need updating. Window 1 is busy running the tunnel; windows 2 and 3 pick up $PUBLIC when they load lab-vars.ps1.

   ![](images/image72.png)

**Conclusion:**

The tunnel is running, and its public address is saved in lab-vars.ps1.

### Task 2: Create .env (window 2)

**Overview:**

Generate the .env file with your Azure values and the local reps and routing files.

1. Open a new PowerShell window (window 2), navigate to the code folder and execute **pwsh**.

   ```powershell
   cd "C:\LabFiles\voice-ai-template-gallery\docs\templates\customer-record-lookup-update\code"
   ```

2. Copy the below script block and paste it in the PowerShell.

   ```powershell
   . .\lab-vars.ps1
   Copy-Item .env.example .env -Force
   $set = [ordered]@{
     PUBLIC_BASE_URL     = $PUBLIC
     ACS_ENDPOINT        = $ACS_ENDPOINT
     VOICE_LIVE_ENDPOINT = $VL_ENDPOINT
     REPS_PATH           = "./config/reps.local.json"
     ROUTING_PATH        = "./config/routing.local.json"
   }
   $lines = Get-Content .env
   foreach ($k in $set.Keys) {
     $found = $false
     $lines = $lines | ForEach-Object { if ($_ -match "^\s*#?\s*$k=") { $found = $true; "$k=$($set[$k])" } else { $_ } }
     if (-not $found) { $lines += "$k=$($set[$k])" }
   }
   $lines | Set-Content .env -Encoding utf8
   Select-String "^(PUBLIC_BASE_URL|ACS_ENDPOINT|VOICE_LIVE_ENDPOINT|VOICE_LIVE_API_KEY|REPS_PATH|ROUTING_PATH)=" .env
   ```

   Expected: five lines with your values and VOICE_LIVE_API_KEY= empty (keyless sign-in). REPS_PATH and ROUTING_PATH must point at the .local.json files — with the original files your phone is unknown and the transfer target is a placeholder.

   ![](images/image73.png)

   ![](images/image74.png)

**Conclusion:**

.env contains your tunnel address, the ACS and Voice Live endpoints and the local reps and routing files.

### Task 3: Start the server (window 2)

**Overview:**

Start the server in live mode.

1. Execute the below commands in window 2.

   ```powershell
   az account show --query user.name -o tsv    # must be the account that received the roles in Exercise 4, Task 4
   npm start
   ```

   ![](images/image75.png)

2. The log must not say SIMULATION MODE, and must end with lines like these. The server reads .env and the config files only at start-up: restart it (Ctrl+C, npm start) after any change.

   **Expected output**

   ```text
     voice model        : gpt-realtime (WebSocket 2026-04-10)
     public base URL    : https://<id>-8097.<region>.devtunnels.ms
   ```

   ![](images/image76.png)

**Conclusion:**

The server is running in live mode with your tunnel as its public address.

### Task 4: Check health through the tunnel (window 3)

**Overview:**

Check the server’s health through the public tunnel address.

1. Open a new PowerShell window (window 3), navigate to the code folder, execute **pwsh**, and then the below commands.

   ```powershell
   cd "C:\LabFiles\voice-ai-template-gallery\docs\templates\customer-record-lookup-update\code"
   . .\lab-vars.ps1
   $h = Invoke-RestMethod "$PUBLIC/health"
   $h | Select mode, callReady
   $h.voiceLive; $h.telephony; $h.teams; $h.crm
   ```

   ![](images/image77.png)

2. Ensure that mode is live; voiceLive and telephony are ready (True); teams.ready is True with no unprovisionedTargets; and crm.ready is True.

   ![](images/image78.png)

**Conclusion:**

The health check reports live mode and ready for Voice Live, telephony, Teams and the CRM.

### Task 5: Create the Event Grid subscription (window 3)

**Overview:**

Subscribe the server to incoming-call events from your ACS resource.

1. The server and tunnel must be running: Event Grid sends a validation handshake that your server answers. Copy and paste the below code block in window 3.

   ```powershell
   az eventgrid event-subscription create -n incoming-call `
     --source-resource-id $ACSID `
     --endpoint "$PUBLIC/api/events" `
     --included-event-types Microsoft.Communication.IncomingCall
   ```

2. It can take up to 2 minutes with no output. Success: JSON ending in "provisioningState": "Succeeded".

3. Window 2 logs **[eventgrid] subscription validation handshake**.

   **Checkpoint:** Health shows live and ready, and the subscription is Succeeded. You are ready to call.

**Conclusion:**

The Event Grid subscription succeeded, so calls to your number now reach the agent.

**Exercise conclusion:**

The agent is live: the server runs with your Azure settings, reports ready, and receives incoming calls through Event Grid.

## Exercise 8: Test with real phone calls

**Overview:**

In this exercise, you call the agent from your mobile as Alex Morgan, verify with the PIN, hear a briefing, update a record by voice with the explicit save and the undo window, and get transferred to yourself in Teams, while watching the server log and the console.

### Task 1: Before you dial

**Overview:**

Get everything ready before you place the test call.

1. Window 1 (tunnel) and window 2 (server) are running.

2. The console http://127.0.0.1:8097 is open, so you can watch the checklist, the record card, the write-back and the decisions live.

3. You, as the Teams user, are Sales Operations. Open a browser and sign in to Teams (https://teams.microsoft.com) as $AGENT.

4. Use a headset or earphones, in a quiet room. A speakerphone causes echo the AI may pick up.

5. You call from $MYPHONE. You have a way to call the US (a mobile with international calling enabled). Your PIN is **246810**.

   **Note:** A live server uses the real date, not the demo day. The seeded tasks and close dates are from mid-2026, so the briefing calls some of them overdue, and a close date in the past is refused. When you dictate an update, use future dates, for example “close date December 15th”.

**Conclusion:**

The tunnel, server, console and Teams are ready for a live call.

### Task 2: Verify and get a briefing

**Overview:**

Call the CRM line, verify with your PIN and hear a briefing.

1. From your mobile, dial the ACS number (+1…) you noted in Exercise 5. The first call of a session may take up to ~20 seconds to answer.

2. The agent says it is the Contoso sales assistant, an automated agent, and asks for your six-digit PIN.

3. Key **246810** on the phone’s keypad. Expected: “Thanks, Alex, you’re verified. Which account?”

4. Say: “Brief me on Fabrikam.” Expected: the tier, the last touch, three open deals, any tasks and the key contact — with no phone number.

5. Say: “What’s Dana’s number?” The number is read out because you asked.

6. Window 2 should show a sequence like this (example; your call IDs and number differ):

   ```text
   [call] incoming 4d2a… via acs-direct (phone)
   [acs] answered 4d2a… -> 0b00…
   [bridge] media socket connected for call 4d2a…
   [acs] CallConnected 4d2a…
   [voice-live] connected (gpt-realtime)
   [acs] ContinuousDtmfRecognitionToneReceived 4d2a…
   [voice-live] tool find_account
   [voice-live] tool get_briefing
   ```

**Conclusion:**

You were verified by mobile and PIN, and the briefing came from the CRM without the contact’s number until you asked.

### Task 3: Update the record — read back, save, undo window

**Overview:**

Dictate an update, confirm it explicitly, and use the undo window and undo.

1. In the same call, say: “Log the Fabrikam meeting. Dana agreed to the renewal at 240k, moving to Closed Won pending signature, close date December 15th. Remind me to send the contract Friday.”

2. Expected: a read-back of a meeting activity, the deal changes and a task due Friday, then “Save it?”.

3. Say only: “Yes.” Expected: the agent asks you to say “yes, save it”; nothing is written.

4. Say: “Yes, save it.” Expected: “Saving in six seconds. Say ‘wait’ to stop it.”, then “Saved to Fabrikam’s record.” The console’s write-back panel shows before → after and the audit rows.

5. Say: “Undo that.” Expected: “Undone. Fabrikam’s record is back to how it was.”

6. Say: “Update Fabrikam: Dana has a new mobile, 425 555 0143.”, then “Yes, save it.”, then within six seconds “Wait!”. Expected: “Stopped. Nothing was written” — the number still ends 0120.

**Conclusion:**

A write needed a read-back and an explicit save, waited out its undo window, could be stopped with “wait” and undone afterwards.

### Task 4: Not mine to change — transfer to Sales Operations

**Overview:**

Ask for a change the agent may not make, and let it transfer you to yourself in Teams.

1. In the same call, say: “Raise Fabrikam’s credit limit to 500k and move them to net 60 payment terms.”

2. Expected: the agent says it cannot change those, nothing is written, and it offers to put you through to Contoso Sales Operations.

3. Say: “Yes, put me through.” Teams rings in your browser. Answer it — you are now Sales Operations.

4. Window 2 shows the transfer:

   ```text
   [acs] transfer requested 4d2a… -> Contoso Sales Operations
   [acs] CallTransferAccepted 4d2a…
   [call] transfer accepted 4d2a…
   ```

5. In the console’s Teams panel, read the VoIP headers: the topic “Fabrikam – credit limit, payment terms” and the context with the rep, how they were verified, the account and the refused requests. The headers sent to ACS have no dots in their names (CallDetailsCallTopic) and the en dash became a plain hyphen — this is the transfer fix at work.

**Conclusion:**

A refused request reached Sales Operations in Teams with the context attached, and nothing was written.

### Task 5: More live tests

**Overview:**

Try other flows on new calls and compare them with the expected behaviour.

1. The AI words things in its own way, but who is verified, what is refused and when a write lands are decided by the server and must match.

   | **#** | **Flow** | **Say or do** | **Pass if** |
   |---|---|---|---|
   | 1 | Wrong PIN | Key 135790 instead of the PIN | No record opened and nothing read out; transferred to Sales Operations with only the last digits of your number |
   | 2 | Human escape | “I need a person”, or press 0 | Transferred to Sales Operations |
   | 3 | Not your account | “Brief me on Tailspin”, twice | The account stays closed; after the second miss, a person |
   | 4 | Call me back | Mid-update: “Call me back later” | The update is kept as a draft, not written; the next call offers to resume it |
   | 5 | Past date | An update with “close date June 15th” | The close date is left out of the read-back: it can’t be in the past |
   | 6 | Barge-in | Talk over the agent mid-sentence | The agent stops speaking immediately |
   | 7 | Unknown caller | Call from a phone that is not $MYPHONE | The line is for Contoso sales staff; the call ends |

   **Important:** If Teams does not ring, look in window 2 for a TRANSFER ERROR line (the full ACS error) or for [call] transfer failed. Then check: $h.teams.ready is True, .env points REPS_PATH and ROUTING_PATH at the .local.json files, the Teams user is signed in to Teams, and the federation settings from Exercise 6, Task 3 show your $IMMUTABLE value (they can take up to an hour to apply). If the agent says the line is for sales staff when you call from $MYPHONE, check that rep-alex in reps.local.json shows exactly the number your phone presents.

2. Press Ctrl+C in windows 1 and 2 to stop the tunnel and the server.

**Conclusion:**

The live calls matched the expected behaviour: verification first, the explicit save with an undo window, refusals by policy, and transfers to Teams with the context.

**Exercise conclusion:**

You have completed real phone calls with the agent: it verified you by mobile and PIN, briefed you on an account, updated and undid a record by voice, refused a policy change and transferred you to Sales Operations in Teams.

## Exercise 9: Delete the resources

**Overview:**

In this exercise, you remove everything you created so it no longer incurs cost: the phone number, the Teams allow-list entry, the Azure resources and the dev tunnel.

Run in order, in one PowerShell 7 window with lab-vars.ps1 loaded. Steps 1–2 must run before the resource group is deleted, because they need the ACS IDs.

1. Release the phone number: Azure portal → ACS resource → **Phone numbers** → select the number → **Release**. Numbers bill monthly until released.

   ![](images/image79.png)

   ![](images/image7a.png)

2. Remove ACS from the Teams allow-list.

   ```powershell
   Connect-MicrosoftTeams
   Set-CsTeamsAcsFederationConfiguration -Identity Global -AllowedAcsResources @{Remove=$IMMUTABLE}
   ```

   ![](images/image7b.png)

3. Delete the Azure resources (ACS, AI Services and the Event Grid subscription).

   ```powershell
   az group delete -n $RG --yes
   ```

   ![](images/image7c.png)

4. Delete the tunnel.

   ```powershell
   devtunnel delete $TUNNEL
   ```

   ![](images/image7d.png)

5. Optional: in the code folder, delete .env and config\reps.local.json and config\routing.local.json, which hold your phone number and Teams ID. Delete data\crm.db to start the CRM again from the seed.

**Exercise conclusion:**

All lab resources are removed. The phone number is released, your ACS resource is no longer allowed in Teams, and the resource group and dev tunnel are deleted.

## Lab summary

In this lab, you explored and ran a hands-free CRM line end to end:

- You explored the project and read the code behind each decision: verification by mobile and PIN, which accounts and fields a rep may see, the field allow-list, policy refusals, the explicit save with an undo window, undo with a conflict check, the AI’s tools and the handoff with the transfer fix.

- You ran the complete agent in simulation mode and saw every rule fire, each with its reason.

- You created Azure Communication Services and Azure AI Services with keyless access, bought a phone number, registered your phone as a rep and prepared Teams so the agent can transfer to Sales Operations.

- You took the agent live with a dev tunnel and an Event Grid subscription, updated a record by voice with real calls, and removed the resources at the end.

The key design idea throughout is that the AI holds the conversation while the server owns every decision: who is calling, what may be read out, which fields may change and within what limits, when a write actually lands, and where a call goes. A write is never one misheard word away: it needs a read-back, an explicit “yes, save it” and an undo window that runs out — and every step is audited.
