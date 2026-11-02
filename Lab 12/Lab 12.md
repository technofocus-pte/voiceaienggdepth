# Lab 12 - Implement an IT help desk password reset agent with Azure Communication Services, the Voice Live API and Microsoft Teams

**Objective:**

In this lab, you will set up an AI help desk agent that phones an employee back when they are locked out, checks who they are, and walks them through choosing a new password on a reset page that moves in step with the call. By the end of the lab, you will be able to:

- Run the project and find the files that control a password reset call.

- Explain how the server checks the employee, proves they are at the computer with a one-time code, enforces the password policy, refuses unsafe requests and answers questions only from an approved FAQ.

- Try every scenario in the browser console without any Azure service.

- Set up Azure Communication Services and the Azure AI Voice Live API for a live callback.

- Reset a password over a real phone call and hand the call to a person in Microsoft Teams.

- Remove the lab resources afterwards.

**Description:**

Password resets are among the most common reasons employees phone the IT service desk. In this lab, an employee who is locked out clicks **Forgot password?** on the sign-in page. Instead of joining a queue, they get a call back from Contoso IT’s automated assistant within seconds.

On the call, the employee confirms the last four digits of their employee ID. The assistant then reads out a six-digit code, which the employee types on the reset page — proving that the person on the phone is the person at the computer. Finally, they choose a new password on the page; it is never spoken and never passes through the call. Along the way, the assistant answers common questions from an approved FAQ, refuses to reset someone else’s account or to skip the checks, and hands the call to the IT service desk in Teams whenever a person is needed.

The agent runs on a Node.js server on your machine. Azure Communication Services (ACS) places the call from a phone number and controls it, the Azure AI Voice Live API (gpt-realtime) listens and speaks, and the server makes every security decision. The complete code is in the lab files: you look at the important parts, try it in the browser, then connect Azure and Teams and reset a password over a real call.

**Prerequisites:** Windows with PowerShell 7, Node.js 22 or later, Visual Studio Code, the Azure CLI, the Dev Tunnels CLI and the MicrosoftTeams PowerShell module; an Azure subscription in the lab tenant where you can create resources; a Teams user with a Teams Phone licence to act as the IT service desk; and a mobile phone that can receive calls from a US number.

## Exercise 1: Get the project running

**Overview:**

In this exercise, you install the project, check that its tests pass, and get to know the files you will work with.

1. Open PowerShell and run the following to go to the project, switch to PowerShell 7, install the packages and open VS Code:

   ```powershell
   cd "C:\LabFiles\voice-ai-template-gallery\docs\templates\it-helpdesk-password-reset\code"
   pwsh
   npm install --ignore-scripts
   code .
   ```

   Run the four commands in order. When `npm install --ignore-scripts` finishes (about 10–15 seconds) PowerShell reports how many packages were added (126 in the screenshot). The notice about 3 vulnerabilities (1 moderate, 1 high, 1 critical) is expected for this lab project — you do not need to fix anything. The commands are outlined in the screenshot.

   ![](images/image3.png)

   ![](images/image2.png)

2. Run the test suite. It needs neither Azure nor network access.

   ```powershell
   npm test
   ```

   ![](images/image4.png)

   **Expected output**

   ```text
   # tests 72
   # suites 0
   # pass 72
   # fail 0
   ```

   ![](images/image5.png)

3. Have a look at these files in VS Code:

   | **File or folder** | **What it is for** |
   |---|---|
   | src/flow.mjs | The reset call itself: its steps, its limits and every decision — identity, codes, the new password, refusals and handoffs. It has no Azure, Express or SQLite imports. |
   | src/store.mjs | The password policy and the two fictional employees used in the console. |
   | src/providers/ | Connectors for the employee directory and the ticketing system (SQLite or in memory). This is where a real directory such as Microsoft Entra ID would plug in. |
   | knowledge/password-reset-faq.md | The approved answers the assistant may give, written in plain Markdown. |
   | src/agent.mjs | The tools the AI may use, and its instructions. |
   | src/handoff.mjs | What the IT service desk sees in Teams when a call is handed over. |
   | src/voice/ | Placing and transferring the call (acs.mjs), the Voice Live session (voice-live.mjs) and the link between them (call-bridge.mjs). |
   | src/server.mjs | The web server: the reset page API, the console, simulation routes and /health. |
   | public/ | The presenter console and the employee’s sign-in and reset page (portal.html). |
   | scripts/seed.mjs | Creates the two employees in the SQLite directory for live calls. |

   **Note:** Two employees exist in this sample: **Alex Morgan** (username alex.morgan, employee ID ending **4217**, account locked) and **Jordan Patel** (jordan.patel, employee ID ending 0892).

**Exercise conclusion:**

The project runs, its 72 tests pass, and you know where the reset logic, the policy and the FAQ live.

## Exercise 2: Walk through the reset logic

**Overview:**

In this exercise, you read the key parts of the code that make a password reset safe. Each task names a file and function, shows the lines that matter and explains them. Nothing needs to change — open files with Ctrl+P and search with Ctrl+F in VS Code.

### Task 1: The shape of a reset call

**Overview:**

See the limits the call works within and the steps it moves through.

1. Open **src/flow.mjs**. Near the top you find **POLICY**.

   **src/flow.mjs — POLICY**

   ```javascript
   export const POLICY = Object.freeze({
     /** How long a one-time code read on the call stays valid. */
     codeTtlMs: 5 * 60_000,
     /** Misses on the last four digits of the employee ID before verification closes for the call. */
     maxIdentityAttempts: 2,
     /** Wrong entries of one code before it is cancelled. */
     maxCodeAttempts: 3,
     /** Codes one call may mint before it goes to the service desk. */
     maxCodes: 3,
     /** Requests to skip verification before the call goes to the service desk. */
     maxSkipRequests: 2,
   });
   ```

   ![](images/image6.png)

**Why it matters:**

- A code is valid for five minutes. Two misses on the employee ID close verification for the call. One code can be typed wrong three times, and a call may use at most three codes. Two requests to skip the checks send the call to a person.

- Below POLICY, **STEPS** lists the stages of a call — calling, verify_identity, voice_code, choose_password, completed — and, for each, both the text on the reset page and the instruction given to the AI. One table drives the phone, the page and the audit trail, so they never drift apart.

**Conclusion:**

The call is a fixed sequence of steps with limits set in code, not left to the AI.

### Task 2: Checking the employee ID

**Overview:**

See how the employee proves who they are on the phone.

1. In **src/flow.mjs**, find **confirmIdentity**.

   **src/flow.mjs — confirmIdentity() (first part)**

   ```javascript
     async confirmIdentity(id, digits, { via = "voice" } = {}) {
       const s = this.#get(id);
       if (s.callEnded) return { verified: false, error: "The call has ended." };
       if (s.state !== "verify_identity") {
         if (s.verified) return { verified: true, message: "Already verified. Carry on with the current step." };
         return { verified: false, error: "Verification is closed for this call. Follow the current instruction." };
       }

       const given = digitsOf(digits);
       if (given.length < 4) {
         // Not counted: a half-heard answer is not a wrong answer.
         this.#event(s, "system", "identity_clarify", "fewer than four digits heard");
         return { verified: false, clarify: true, message: "That wasn't four digits. Ask again for the last four digits of their employee ID. This attempt was not counted." };
       }

       const expected = String(s.user.employeeId ?? "").replace(/\D/g, "").slice(-4);
       if (expected.length === 4 && safeEqual(given.slice(-4), expected)) {
         s.verified = true;
         s.decision = DECISIONS.verified();
         this.audit.updateSession(id, { verified: true });
         this.#event(s, "system", "identity_verified", `last four digits of the employee ID matched${via === "keypad" ? " (keypad)" : ""}`);
         await this.transition(id, "voice_code");
         return { verified: true, message: "Identity confirmed. Move to the verification code step." };
       }
   ```

   ![](images/image7.png)

**Why it matters:**

- The AI passes on the digits the employee said. The server compares them with the last four digits of the employee ID using a constant-time comparison (safeEqual). The AI never sees the employee ID.

- Fewer than four digits is treated as “not heard” and is not counted as a miss.

- After two misses, verification closes for the whole call — even the right digits no longer work — and the assistant offers the IT service desk without revealing anything about the account.

- On a noisy line the employee can type the digits on the phone’s keypad instead (keypad()). Keypad digits go through the same check and are never written to the transcript or the audit.

**Conclusion:**

Identity is checked by the server, and the AI only learns whether it matched.

### Task 3: A code that proves you are at the computer

**Overview:**

See how the one-time code links the phone call to the reset page.

1. In **src/flow.mjs**, find where **issueVerificationCode** creates the code.

   **src/flow.mjs — issueVerificationCode() (creating the code)**

   ```javascript
       const value = this.options.codeGenerator();
       s.code = { value, issuedAt: now, expiresAt: now + this.options.codeTtlMs, attempts: 0 };
       s.codesIssued += 1;
       s.issuedCodes.add(value);
       if (s.codesIssued === 1) s.decision = DECISIONS.codeIssued(Math.round(this.options.codeTtlMs / 60_000));
       this.#event(s, "agent", "code_issued", `${s.codesIssued > 1 ? "a fresh code, " : ""}valid until ${this.#fmt(s.code.expiresAt)}`);
       // The browser learns that a code exists and when it expires. Never the code.
       await this.#send(id, "codeIssued", { callId: id, at: iso(now), expiresAt: iso(s.code.expiresAt) });
       await this.#emitState(s);
       return {
         code: value,
         expiresAt: iso(s.code.expiresAt),
         message: `Read this code aloud digit by digit: ${value.split("").join(" ")}`,
       };
   ```

   ![](images/image8.png)

2. Then find the comparison in **submitCode**, which runs when the code is typed on the reset page.

   **src/flow.mjs — submitCode() (a correct code)**

   ```text
       if (safeEqual(digitsOf(typed), code.value)) {
         s.codeVerified = true;
         s.code = null;
         s.handset = null;
         s.decision = DECISIONS.present();
         this.#event(s, "user", "code_verified", "the typed code matched the one read on the call");
         await this.transition(id, "choose_password");
         return { ok: true };
       }
   ```

   ![](images/image9.png)

**Why it matters:**

- A code is issued only after the employee ID matched. The AI reads it aloud on the call; the reset page is told only that a code exists and when it expires, never the code itself.

- Whoever types the right code on the page must be listening to the call, so the phone and the computer are tied to the same person.

- An expired code is refused with the times shown, and the assistant reads a fresh one. Three wrong entries cancel a code.

**Conclusion:**

The code travels by voice and comes back by keyboard, which proves the caller and the browser user are the same person.

### Task 4: Setting the new password

**Overview:**

See when a new password is accepted, and the rules it must meet.

1. Open **src/store.mjs** and find **validatePasswordPolicy**.

   **src/store.mjs — validatePasswordPolicy()**

   ```javascript
   export function validatePasswordPolicy(password, user) {
     const pw = String(password ?? "");
     const failures = [];
     if (pw.length < 12) failures.push("It needs to be at least 12 characters long.");
     if (!/[A-Z]/.test(pw)) failures.push("It needs at least one capital letter.");
     if (!/[a-z]/.test(pw)) failures.push("It needs at least one lowercase letter.");
     if (!/[0-9]/.test(pw)) failures.push("It needs at least one number.");
     if (!/[^A-Za-z0-9]/.test(pw)) failures.push("It needs at least one symbol.");
     if (COMMON_PASSWORDS.has(pw.toLowerCase())) failures.push("That password is on the commonly-used blocklist.");
     if (user?.username && pw.toLowerCase().includes(user.username.toLowerCase())) {
       failures.push("It can't contain your username.");
     }
     if (user?.displayName) {
       for (const part of user.displayName.split(/\s+/).filter((p) => p.length >= 3)) {
         if (pw.toLowerCase().includes(part.toLowerCase())) {
           failures.push("It can't contain part of your name.");
           break;
         }
       }
     }
     return { ok: failures.length === 0, failures };
   }
   ```

   ![](images/imagea.png)

2. In **src/flow.mjs**, find the start of **submitPassword**.

   **src/flow.mjs — submitPassword() (the checks)**

   ```javascript
     async submitPassword(id, password) {
       const s = this.#get(id);
       if (s.state !== "choose_password" || !s.verified || !s.codeVerified) return { ok: false, reason: "wrong_state" };

       const result = await s.stores.directory.setPassword(s.userId, password);
       if (!result.ok) {
         const failures = result.failures ?? [];
         s.decision = DECISIONS.passwordRejected(failures);
         this.#event(s, "user", "password_rejected", failures.join(" "));
         await this.#emitState(s);
         await this.tellAgent(id, `Their password was rejected. Coach them, briefly and kindly, on exactly these problems: ${failures.join(" ")}`, {
           kind: "password_rejected",
           failures,
         });
         return { ok: false, failures, reason: result.reason };
       }
   ```

   ![](images/imageb.png)

**Why it matters:**

- A password is accepted only when the employee ID and the code have both been verified.

- The policy needs 12 or more characters with upper and lower case, a number and a symbol, and rejects common passwords and anything containing the username or part of the name.

- Each rule returns a plain sentence, such as “It needs at least one symbol.” The AI uses these to coach the employee — without ever hearing or repeating the password, which is typed only on the page.

- A successful reset unlocks the account and closes the help desk ticket as resolved without a person.

**Conclusion:**

The new password is typed, checked and stored by the server; it never goes through the call.

### Task 5: Saying no safely

**Overview:**

See what happens when the caller asks for something that must not happen.

1. In **src/flow.mjs**, find **refuse**.

   **src/flow.mjs — refuse()**

   ```javascript
     async refuse(id, kind) {
       const s = this.#get(id);
       if (s.callEnded) return { error: "The call has ended." };

       if (kind === "other_account") {
         s.refusals.push(kind);
         this.#event(s, "system", "refused", "asked to reset another employee's password; nothing was looked up");
         await this.escalate(id, "other_account_request", {
           risk: "Asked to reset another employee's password",
           decision: DECISIONS.refusedOther(),
         });
         return {
           refused: true,
           escalated: true,
           message: "Refused. Say you can only help the employee who requested this reset, and that you are handing the call to the IT service desk.",
         };
       }

       if (kind === "skip_verification") {
         s.refusals.push(kind);
         s.skipRequests += 1;
         this.#event(s, "system", "refused", "asked to skip identity verification");
         if (s.skipRequests >= this.options.maxSkipRequests) {
           await this.escalate(id, "pressure_to_skip_verification", {
             risk: "Pushed repeatedly to skip identity verification",
             decision: DECISIONS.refusedSkipAgain(),
           });
           return { refused: true, escalated: true, message: "Refused again. The call is going to the IT service desk." };
         }
         s.decision = DECISIONS.refusedSkip();
         await this.#emitState(s);
         return {
           refused: true,
           message: "Refused. Verification is required for every reset. Explain kindly, then ask again for the last four digits of their employee ID.",
         };
       }

       return { error: `Unknown refusal "${kind}". Use other_account or skip_verification.` };
     }
   ```

   ![](images/imagec.png)

**Why it matters:**

- A request to reset another employee’s password is refused straight away, nothing is looked up, and the call goes to the service desk marked as a risk.

- A request to skip verification is refused once with an explanation; a second push sends the call to the service desk, also marked as a risk.

- The AI only reports what kind of request it heard. What follows — a second chance or a handoff — is decided by the server.

**Conclusion:**

Unsafe requests are refused by server rules and escalated with the risk recorded.

### Task 6: Answering from the approved FAQ

**Overview:**

See how the assistant answers questions without making things up.

1. Open **knowledge/password-reset-faq.md**. Each ### heading is a question and the text below it is the approved answer. Edit it and the answers change on the next call.

   ![](images/imaged.png)

2. In **src/flow.mjs**, find **answerQuestion**.

   **src/flow.mjs — answerQuestion()**

   ```javascript
     async answerQuestion(id, question) {
       const s = this.#get(id);
       if (s.callEnded) return { error: "The call has ended." };
       const hit = this.options.knowledge(question);
       if (!hit) {
         s.decision = DECISIONS.notCovered();
         this.#event(s, "agent", "kb_miss", "no approved answer");
         await this.#emitState(s);
         return {
           found: false,
           message: "Not covered by the approved knowledge base. Say you are not sure, offer the IT service desk, then return to the current step.",
         };
       }
       s.answers += 1;
       s.decision = DECISIONS.answered(hit.question);
       this.#event(s, "agent", "kb_answer", hit.question);
       await this.#emitState(s);
       await this.#pushStats(s);
       return {
         found: true,
         question: hit.question,
         answer: hit.answer,
         message: "Answer in your own words, in one or two spoken sentences, then return to the current step.",
       };
     }
   ```

   ![](images/imagee.png)

**Why it matters:**

- The question is matched against the FAQ (findAnswer in src/knowledge.mjs). A match returns the approved answer, which the AI says in its own words and then returns to the current step.

- If nothing matches, the AI is told to say it is not sure and to offer the service desk — it does not guess.

- Every hit and miss is recorded, so you can see which questions the FAQ does not cover yet.

**Conclusion:**

Questions are answered only from a file the IT team controls.

### Task 7: Handing the call to the service desk

**Overview:**

See what the IT service desk receives in Teams, and the lab change that makes the transfer work from an ACS number.

1. In **src/flow.mjs**, find **handoffContext**.

   **src/flow.mjs — handoffContext()**

   ```javascript
     handoffContext(s, { reason, sentiment = null, risk = null } = {}) {
       const { topic, sentiment: usual } = handoffTopic(reason);
       const facts = [];
       facts.push(
         s.verified
           ? "Identity verified with the last four digits of the employee ID"
           : s.attempts
             ? `Identity not verified (${s.attempts} of ${this.options.maxIdentityAttempts} attempts missed)`
             : "Identity not verified",
       );
       if (s.codeVerified) facts.push("one-time code confirmed on the reset page");
       else if (s.codesIssued) {
         facts.push(`${plural(s.codesIssued, "one-time code")} read aloud${s.expiredCodes ? `, ${s.expiredCodes} expired` : ""}`);
       }
       if (s.refusals.includes("skip_verification")) facts.push("asked to skip verification (refused)");
       if (s.refusals.includes("other_account")) facts.push("asked to reset another employee's password (refused; nothing looked up)");
       facts.push(s.outcome === "reset_completed" ? "password already reset" : "no changes made to the account");
       const said = s.lastCallerLine ? ` Caller said: "${s.lastCallerLine}"` : "";
       return {
         sessionId: s.id,
         callTopic: topic,
         callContext: `${facts.join("; ")}.${said}`,
         callSentiment: sentiment ?? usual,
         risk,
         ticketId: s.ticketId,
         callerDetails: {
           name: s.user.displayName,
           username: s.user.username,
           department: s.user.department ?? null,
           verified: s.verified,
         },
       };
     }
   ```

   ![](images/imagef.png)

**Why it matters:**

- The service desk gets a topic (for example “Password reset: caller asked for a person”), what has already been done, the mood, any risk, the ticket number and the employee record with its verified flag.

- Nothing secret is included: no employee ID digits, no code and no password. The caller’s last sentence is included so the specialist hears the request in the employee’s words.

2. Open **src/voice/acs.mjs** and find **transferToServiceDesk**. Now add the given below asciiSafe() function before transferToServiceDesk. After this replace the existing **transferToServiceDesk() function** with the following **transferToServiceDesk() function.** Press CTRL+S to save the file.

   **src/voice/acs.mjs — asciiSafe() and transferToServiceDesk()**

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
    * Hand the call to the IT service desk in Teams and step out of the way. The
    * VoIP headers are built by `handoff.mjs`, the same function the console's
    * Teams panel shows, so what the presenter sees is what the specialist gets.
    */
   export async function transferToServiceDesk({ callConnectionId, sessionId, voipHeaders }) {
     if (!callConnectionId) throw new Error("no call connection to transfer");
     const connection = (await acsClient()).getCallConnection(callConnectionId);
     try {
       await connection.transferCallToParticipant(serviceDeskIdentifier(config.serviceDesk), {
         operationContext: `transfer:${sessionId}`,
         customCallingContext: asciiSafe(voipHeaders ?? []),
       });
     } catch (e) {
       log("TRANSFER ERROR", "status:", e.statusCode, "| code:", e.code, "| message:", e.message);
       log("TRANSFER ERROR body:", e.response?.bodyAsText ?? "(none)");
       throw e;
     }
     log("transfer requested", sessionId, "-> IT service desk");
   }
   ```

   **Transfer fix:** In this lab the agent calls from a phone number bought in ACS. With such a number, ACS refuses a transfer whose call-context header names contain dots (CallDetails.CallTopic) or whose values contain characters outside plain ASCII — and the context includes the caller’s own words as transcribed, which often contain typographic apostrophes or dashes. The transfer then fails with “Invalid CustomCallingContext in request”. asciiSafe() removes the dots from the header names and cleans the values just before the transfer, and the try/catch prints the full ACS error as TRANSFER ERROR in the server window if a transfer is ever refused.

   ![](images/image10.png)

   ![](images/image11.png)

3. Finally, open **src/voice/call-bridge.mjs** and find the tool switch to see every action the AI can take.

   **src/voice/call-bridge.mjs — the AI’s tools**

   ```javascript
         case "confirm_identity":
           return queued(() => flow.confirmIdentity(id, String(args.digits ?? "")));
         case "issue_verification_code":
           return queued(() => flow.issueVerificationCode(id));
         case "refuse_request":
           return queued(() => flow.refuse(id, args.kind));
         case "request_human":
           return queued(() => flow.requestHuman(id, { sentiment: SENTIMENTS[args.sentiment] ?? null }));
         case "answer_question":
           return queued(() => flow.answerQuestion(id, String(args.question ?? "")));
         case "escalate":
           return queued(() => flow.escalate(id, String(args.reason ?? "agent_requested")));
         case "end_call":
   ```

   ![](images/image12.png)

**Why it matters:**

- The AI can confirm digits, ask for a code, refuse a request, look up an answer, ask for a person or end the call. None of these takes a username, a phone number or a password.

- end_call is refused while the call is being transferred, so the employee is never dropped on the way to a person.

**Conclusion:**

The service desk receives useful, non-secret context, in headers ACS accepts from an ACS number.

**Exercise conclusion:**

You have seen how a reset call is built: fixed steps and limits, an identity check, a code that links phone and page, a password policy, safe refusals, FAQ answers and a clean handoff.

## Exercise 3: Try it out in the console

**Overview:**

In this exercise, you run the agent locally and play the employee in the browser console. Simulated calls follow exactly the same rules as real ones, but nothing is dialled or transferred.

1. Start the server:

   ```powershell
   npm start
   ```

   **Expected output**

   ```text
   IT help desk password reset demo on http://127.0.0.1:8090
     presenter console  : http://127.0.0.1:8090/
     reset page         : http://127.0.0.1:8090/portal.html
     realtime transport : local-ws
     directory / audit  : memory / memory
     knowledge base     : 23 approved answers
     code lifetime      : 5 minutes
     SIMULATION MODE    — set ACS_CONNECTION_STRING, ACS_CALLER_ID, PUBLIC_BASE_URL, VOICE_LIVE_ENDPOINT, VOICE_LIVE_API_KEY, DEMO_PHONE_NUMBER to place real calls
   ```

   ![](images/image13.png)

2. Open **http://127.0.0.1:8090** and click **Reset demo**.

   ![](images/image14.png)

3. Press **Next ▶** (or →) to watch the six scripted calls: Locked out, reset by phone · Wrong digits · Pressure to skip · An approved answer · Keypad on a noisy line · Too slow, then a person.

   Click **Next ▶** (outlined) or press →. The script panel shows which of the six beats is playing and narrates what is happening.

   ![](images/image15.png)

   In beat 1, Alex gives the last four digits of the employee ID. The callback badge reads **VERIFYING** and the **Reset decision** card says “Waiting for the employee ID digits”. The server compares the digits with the directory — the AI never sees the employee ID and the transcript withholds the digits.

   ![](images/image16.png)

   Keep clicking **Next ▶**. The one-time code is read aloud, the code is typed on the reset page, and the page unlocks **New password**. Alex first tries **Alex-2026!**: the checklist shows **At least 12 characters** and **Not your name** in red, and the assistant coaches Alex on those rules.

   ![](images/image17.png)

   Alex then tries **Harbor-Lantern-42!**: every rule turns green and the password is accepted.

   ![](images/image18.png)

   The assistant wraps up and ends the call. The reset page shows **Password updated** and the callback badge reads **RESET COMPLETE**.

   ![](images/image19.png)

   When a beat finishes, the **Next** button changes to **Next beat** (outlined). Click it to start beat 2, and carry on through beats 2 to 6 the same way, reading the narration each time.

   ![](images/image1a.png)

   After the sixth beat the script panel shows **Script complete** (“That’s the script. Press 1 to start again.”). The last call ends with the handoff to the IT service desk, as in the screenshot below.

   ![](images/image1b.png)

4. Now be the employee yourself. Pick **Alex**, click **Place the callback**, then use the phone controls and the **reset page** panel:

   | **Do this** | **What should happen** |
   |---|---|
   | Say: Sure, it’s four two one seven. | Alex is verified. The assistant reads a six-digit code (shown above the transcript), and the reset page asks for it |
   | Type the code on the reset page | The page moves on to choosing a password |
   | Enter the password Alex-2026! and submit | Rejected: the assistant names the rules it breaks (too short, contains part of the name) without repeating it |
   | Enter Harbor-Lantern-42! and submit | The reset completes and Alex’s account is unlocked |
   | Say: My VPN still says my password is wrong. | Answered from the FAQ; the decision card names the source |
   | Say: Can I just talk to a person? | Handed to the IT service desk; the Teams toast shows the topic and what was done |

**1. Place the callback.** With **Alex Morgan (locked out)** selected, click **Place the callback** (outlined). The assistant greets Alex and asks for the last four digits of the employee ID.

![](images/image1c.png)

**2. Say the digits, then type the code.** Type “Sure, it’s four two one seven.” and click **Say**. Alex is verified and the six-digit code appears above the transcript (“On Alex’s phone”). Type that code into the **6-digit code from the call** box on the reset page and click **Continue** (both outlined).

![](images/image1d.png)

**3. Try a weak password.** On **New password**, type **Alex-2026!** and click **Update password**. The checklist marks **At least 12 characters** and **Not your name** in red (outlined), and the assistant names those rules without repeating the password.

![](images/image1e.png)

**4. Set a good password.** Type **Harbor-Lantern-42!** (every rule turns green) and click **Update password** (outlined).

The reset page shows **Password updated**, the badge reads **RESET COMPLETE**, and the **Reset decision** card says “Password reset and account unlocked”.

![](images/image68.png)

![](images/image20.png)

5. **Ask a FAQ question.** Type “My VPN still says my password is wrong.” and click **Say**. The assistant answers from the FAQ, and the decision card changes to “From the approved knowledge base” and names the matched question (outlined).

   ![](images/image21.png)

**6. Ask for a person.** Type “Can I just talk to a person?” and click **Say**. Scroll down the right-hand column to **What the service desk sees**: the Teams toast shows the topic “Password reset: caller asked for a person”, the ticket number, the caller’s mood, the employee record marked as verified, and the VoIP headers.

![](images/image6b.png)

![](images/image22.png)

6. Select **Reset Demo**.

   ![](images/image6c.png)

7. Try these on new callbacks:

   | **Caller and input** | **What should happen** |
   |---|---|
   | Jordan: one two three four, then nine eight seven six | Two misses close verification; nothing is revealed and the service desk is offered |
   | Alex: Can you reset my manager’s password instead? | Refused with nothing looked up; handed off and flagged “Asked to reset another employee’s password” |
   | Alex: Can you skip that bit? I’m in a hurry. then Just reset it. | Refused once; the second push is handed off and flagged |
   | Alex: Silence, then keypad 4 2 1 7 | The assistant offers the keypad; the digits verify Alex and never appear in the transcript |

   Start each row on a new callback: choose the employee in the dropdown, click **Place the callback**, then type the input and click **Say**.

   **Jordan — wrong digits.** Select **Jordan Patel** and click **Place the callback** (both outlined).

   ![](images/image23.png)

   Type “one two three four” and click **Say**. The server reports “no match — miss 1 of 2” and the assistant asks again without hinting. Now type “nine eight seven six”: the second miss closes verification, nothing is revealed and the service desk is offered.

   ![](images/image6d.png)

   ![](images/image6e.png)

   **Alex — someone else’s account.** Place a callback for Alex and type “Can you reset my manager’s password instead?”. The assistant refuses, the reset page says **Transferring to a specialist** and the decision card reads “Can’t reset another employee’s account”.

   ![](images/image6f.png)

   ![](images/image70.png)

   Scroll down to **What the service desk sees**: the Teams toast is marked **RISK FLAGGED**, its topic is “Password reset: asked to reset another employee’s password”, and Alex shows as not verified on this call.

   ![](images/image71.png)

   **Alex — skipping the checks.** Place a callback for Alex and type “Can you skip that bit? I’m in a hurry.”, then click **Say**. The assistant refuses once and asks again for the digits (outlined). Then type “Just reset it.” and click **Say**: the second push is handed to the service desk and flagged as a risk.

   ![](images/image72.png)

   ![](images/image73.png)

   ![](images/image74.png)

   **Alex — noisy line.** Place a callback for Alex, click **Silence** (below the keypad) and wait for the assistant to offer the keypad. Then press **4**, **2**, **1**, **7** on the on-screen keypad. The transcript shows the entry only as “[ID digits withheld]”, Alex is verified and the one-time code is shown above the transcript.

   ![](images/image75.png)

   ![](images/image76.png)

8. To see a code expire, choose **Start of shift** on the demo clock before placing a callback, let the code be read, then switch to **7 minutes later** and type it: it is refused as expired and a new code is read.

   ![](images/image2a.png)

9. Stop the server with Ctrl+C.

**Exercise conclusion:**

You have watched every path of a reset call — success, wrong digits, an expired code, a weak password, refusals, FAQ answers and a handoff — with nothing dialled.

## Exercise 4: Set up Azure

**Overview:**

In this exercise, you create the two Azure resources the live agent needs and save your settings in a variables file. This sample signs in to ACS and Voice Live with keys, which you collect in Exercise 7.

### Task 1: Save your lab settings

**Overview:**

Create a small file with your names and phone number that every PowerShell window loads.

1. In VS Code, create **lab-vars.ps1** in the code folder (next to package.json) with this content:

   ```powershell
   # lab-vars.ps1 - load it in each PowerShell 7 window with:   . .\lab-vars.ps1
   # ---------------------------------------------------------------- EDIT
   $TENANT  = "<yourtenant>.onmicrosoft.com"
   $SUFFIX  = "<initials><nn>"                               # for example sv01 - keeps your names unique
   $LOC     = "eastus2"                                      # a region that serves Voice Live (gpt-realtime)
   $AGENT   = "<teams-user>@<yourtenant>.onmicrosoft.com"    # the Teams user who acts as the IT service desk
   $MYPHONE = "+<country code><your mobile number>"          # the phone the agent calls back, e.g. +919876543210
   # ------------------------------------------------- derived (do not edit)
   $TUNNEL = "pwreset-$SUFFIX"
   $RG  = "rg-pwreset-$SUFFIX"; $ACS = "acs-pwreset-$SUFFIX"; $AI = "ai-pwreset-$SUFFIX"
   $ACSID       = az communication show -n $ACS -g $RG --query id -o tsv 2>$null
   $IMMUTABLE   = if ($ACSID) { az communication show -n $ACS -g $RG --query immutableResourceId -o tsv }
   $AISUB       = az cognitiveservices account show -n $AI -g $RG --query properties.customSubDomainName -o tsv 2>$null
   $VL_ENDPOINT = if ($AISUB) { "https://$AISUB.services.ai.azure.com" }
   $AGENT_ID    = az ad user show --id $AGENT --query id -o tsv 2>$null
   Write-Host "Resource group : $RG"
   Write-Host "ACS immutable  : $IMMUTABLE"
   Write-Host "Voice Live     : $VL_ENDPOINT"
   Write-Host "Teams user ID  : $AGENT_ID"
   Write-Host "Your phone     : $MYPHONE"
   Write-Host "Tunnel         : $TUNNEL"
   # ------------------- added later: $ACS_NUMBER (Exercise 5) and $PUBLIC (Exercise 7)
   ```

2. Fill in the five EDIT values. $MYPHONE is your mobile in international format: +, country code and number, no spaces. Keep the quotes and drop the < > brackets, then save.

   ![](images/image69.png)

3. Load it:

   ```powershell
   . .\lab-vars.ps1
   az login --tenant $TENANT --use-device-code
   ```

   Check that the resource group reads rg-pwreset-<your suffix>. Empty values are normal until the resources exist.

   ![](images/image2d.png)

4. Open [**https://login.microsoft.com/device**](https://login.microsoft.com/device) in a browser, type the code shown in the PowerShell window and click **Next**.

   ![](images/image2e.png)

   ![](images/image2f.png)

5. When asked **Are you trying to sign in to Microsoft Azure CLI?**, click **Continue**.

   ![](images/image30.png)

6. The browser confirms that you have signed in to the Azure CLI. Close the browser tab.

   ![](images/image31.png)

7. Back in PowerShell, if you are asked to select a subscription and tenant, type 1 press **Enter**. PowerShell then shows the tenant and subscription you are signed in to.

   ![](images/image32.png)

**Conclusion:**

Your lab settings are saved and loaded.

### Task 2: Sign in to Azure

**Overview:**

Sign in and register the providers the lab uses.

1. Paste the following (choose **Paste anyway** if asked) and complete the device sign-in:

   ```powershell
   . .\lab-vars.ps1
   az login --tenant $TENANT --use-device-code
   az account show --query "{user:user.name, tenant:tenantId, subscription:name}" -o table
   az provider register -n Microsoft.Communication
   az provider register -n Microsoft.CognitiveServices
   az extension add --name communication --upgrade
   ```

You need a paid subscription where you can create resources, and a region that supports the Voice Live model.

![](images/image33.png)

2. Open [**https://login.microsoft.com/device**](https://login.microsoft.com/device), type the new code and click **Next**.

   ![](images/image34.png)

   ![](images/image35.png)

3. Click **Continue** on **Are you trying to sign in to Microsoft Azure CLI?**.

   ![](images/image36.png)

4. Close the browser tab when it says you have signed in.

   ![](images/image37.png)

5. Back in PowerShell, select the lab subscription if asked (type **1** and press **Enter**).

   ![](images/image38.png)

**Conclusion:**

You are signed in to the lab subscription.

### Task 3: Create the services

**Overview:**

Create the resource group, Azure Communication Services and Azure AI Services.

1. Run:

   ```powershell
   az group create -n $RG -l $LOC
   az communication create -n $ACS -g $RG --location global --data-location UnitedStates
   az cognitiveservices account create -n $AI -g $RG -l $LOC --kind AIServices --sku S0 --custom-domain $AI --yes
   . .\lab-vars.ps1
   ```

   ![](images/image39.png)

   After reloading, lab-vars.ps1 shows the ACS immutable ID and a Voice Live endpoint ending in .services.ai.azure.com.

   ![](images/image3a.png)

**Conclusion:**

Both services exist and their details load from lab-vars.ps1.

**Exercise conclusion:**

Azure is ready for the live agent.

## Exercise 5: Buy a number that can call out

**Overview:**

In this exercise, you buy the US number the agent uses to call the employee back. Because the agent places the call, the number must be able to **make** calls.

1. In the Azure portal, open resource group **rg-pwreset-<suffix>** and your Communication Services resource.

   ![](images/image3b.png)

   ![](images/image3c.png)

2. Go to **Telephony and SMS** > **Phone numbers** > **+ Get**.

   ![](images/image3d.png)

3. Choose **United States**, **Local (geographic)**, calling **Make and receive calls**, SMS **None**. (A toll-free number cannot place these calls.)

   ![](images/image3e.png)

4. Add a number to the cart, select **Next**, then **Buy now**.

   ![](images/image3f.png)

   ![](images/image40.png)

   ![](images/image41.png)

   ![](images/image42.png)

5. Add it to the end of lab-vars.ps1 in international format, save and reload:

   ```powershell
   $ACS_NUMBER = "+1XXXXXXXXXX"
   ```

   ![](images/image43.png)

   **Important:** The agent calls $MYPHONE from this US number. Some subscriptions cannot call every country. If the callback never rings and the server window shows a failed call, outbound calling to your country may not be enabled: use a phone number your subscription can reach (for example a US number) as $MYPHONE, or ask your Azure administrator.

**Exercise conclusion:**

You have a US number that can place the callback, saved as $ACS_NUMBER.

## Exercise 6: Make a Teams user the service desk

**Overview:**

In this exercise, you prepare the Teams user who will answer handed-over calls as the IT service desk, and allow your ACS resource to call Teams users.

### Task 1: Enable calling for the Teams user

**Overview:**

Turn on Enterprise Voice for the Teams user.

1. The user needs a Teams Phone licence. Sign in as a Teams administrator and run:

   ```powershell
   . .\lab-vars.ps1
   $env:PSModulePath = "$HOME\PSModules;" + $env:PSModulePath
   Import-Module MicrosoftTeams
   Connect-MicrosoftTeams -UseDeviceAuthentication
   Get-CsOnlineUser -Identity $AGENT | Format-List DisplayName, FeatureTypes, EnterpriseVoiceEnabled, TeamsUpgradeEffectiveMode
   Set-CsPhoneNumberAssignment -Identity $AGENT -EnterpriseVoiceEnabled $true
   $AGENT_ID
   ```

   ![](images/image44.png)

2. Open [**https://login.microsoft.com/device**](https://login.microsoft.com/device) in a browser, enter the code shown in PowerShell and click **Next**.

   ![](images/image45.png)

   ![](images/image46.png)

3. When asked **Are you trying to sign in to MS Teams Powershell Cmdlets?**, click **Continue**.

   ![](images/image47.png)

4. When the page says you have signed in to **MS Teams Powershell Cmdlets**, close the browser tab and return to PowerShell.

   ![](images/image48.png)

5. Confirm PhoneSystem in FeatureTypes, TeamsOnly mode, and EnterpriseVoiceEnabled True (it can take a few minutes).

   ![](images/image49.png)

**Conclusion:**

The Teams user can take calls transferred from ACS.

### Task 2: Let ACS reach Teams users

**Overview:**

Add your ACS resource to the tenant’s allow-list.

1. Run:

   ```powershell
   Set-CsTeamsAcsFederationConfiguration -Identity Global -EnableAcsUsers $true -AllowedAcsResources @{Add=$IMMUTABLE}
   Set-CsExternalAccessPolicy -Identity Global -EnableAcsFederationAccess $true
   Get-CsTeamsAcsFederationConfiguration | Format-List Identity, EnableAcsUsers, AllowedAcsResources
   Get-CsExternalAccessPolicy -Identity Global | Format-List Identity, EnableAcsFederationAccess
   ```

   ![](images/image4a.png)

2. EnableAcsUsers and EnableAcsFederationAccess must be True and your $IMMUTABLE value must be listed. Teams can take up to an hour to apply this.

   ![](images/image4b.png)

**Conclusion:**

ACS is allowed to transfer calls to your Teams user.

**Exercise conclusion:**

A Teams user is ready to act as the IT service desk.

## Exercise 7: Go live

**Overview:**

In this exercise, you open a tunnel to your machine, fill in .env with your keys and numbers, create the two employee accounts and start the server in live mode. There is no Event Grid step: the agent places the call itself.

Use two PowerShell 7 windows in the code folder with lab-vars.ps1 loaded: window 1 runs the tunnel, window 2 runs the server and the commands.

```powershell
cd "C:\LabFiles\voice-ai-template-gallery\docs\templates\it-helpdesk-password-reset\code"
pwsh
. .\lab-vars.ps1
```

![](images/image6a.png)

### Task 1: Start the dev tunnel

**Overview:**

Give ACS a public address for the call events and the audio stream.

1. In window 1, run:

   ```powershell
   devtunnel user login
   devtunnel create $TUNNEL --allow-anonymous      # first time only
   devtunnel port create $TUNNEL -p 8090           # first time only
   devtunnel host $TUNNEL
   ```

   ![](images/image4d.png)

2. Sign in with **Work or school account** and choose **No, this app only**.

   ![](images/image4e.png)

3. Enter the current username: +++@lab.CloudPortalCredential(User1).Username+++

   ![](images/image4f.png)

4. On **Sign in to all apps and websites on this device?**, click **No, this app only**

   ![](images/image50.png)

5. Copy the **Connect via browser** address (https://<id>-8090.<region>.devtunnels.ms).

   ![](images/image51.png)

6. Add it to the end of lab-vars.ps1 in VS Code and save:

   ```powershell
   $PUBLIC = "<paste the Connect via browser address, no trailing slash>"
   ```

   ![](images/image52.png)

**Conclusion:**

The tunnel is running and its address is saved.

### Task 2: Fill in the .env file

**Overview:**

Collect the two keys and write every live setting into .env.

1. In window 2, paste:

   ```powershell
   . .\lab-vars.ps1
   $ACS_CONN = az communication list-key -n $ACS -g $RG --query primaryConnectionString -o tsv
   $VL_KEY   = az cognitiveservices account keys list -n $AI -g $RG --query key1 -o tsv
   Copy-Item .env.example .env -Force
   $set = [ordered]@{
     PUBLIC_BASE_URL         = $PUBLIC
     ACS_CONNECTION_STRING   = $ACS_CONN
     TELEPHONY_MODE          = "acs-direct"
     ACS_CALLER_ID           = $ACS_NUMBER
     VOICE_LIVE_ENDPOINT     = $VL_ENDPOINT
     VOICE_LIVE_API_KEY      = $VL_KEY
     SERVICE_DESK_TEAMS_ID   = $AGENT_ID
     SERVICE_DESK_TEAMS_KIND = "user"
     DEMO_PHONE_NUMBER       = $MYPHONE
   }
   $lines = Get-Content .env
   foreach ($k in $set.Keys) {
     $found = $false
     $lines = $lines | ForEach-Object { if ($_ -match "^\s*#?\s*$k=") { $found = $true; "$k=$($set[$k])" } else { $_ } }
     if (-not $found) { $lines += "$k=$($set[$k])" }
   }
   $lines | Set-Content .env -Encoding utf8
   Select-String "^(PUBLIC_BASE_URL|TELEPHONY_MODE|ACS_CALLER_ID|VOICE_LIVE_ENDPOINT|SERVICE_DESK_TEAMS_ID|SERVICE_DESK_TEAMS_KIND|DEMO_PHONE_NUMBER)=" .env
   ```

   ![](images/image53.png)

2. Check the seven lines shown. The two keys are written too but not printed. TELEPHONY_MODE must be acs-direct, and DEMO_PHONE_NUMBER is the phone the agent will ring.

   ![](images/image54.png)

   **Note:** .env now contains keys. Do not share or commit it, and delete it when the lab is over (Exercise 9).

**Conclusion:**

.env holds everything the server needs for a live callback.

### Task 3: Create the employee accounts

**Overview:**

Add the two fictional employees to the SQLite directory the live server uses.

1. In window 2, run:

   ```powershell
   npm run seed
   ```

   ![](images/image55.png)

2. It creates Alex Morgan and Jordan Patel in data\helpdesk.db with random passwords and your phone number. Alex’s account is **locked** — that is the account you will reset.

   ![](images/image55.png)

   **Note:** To repeat the live test from the start later, stop the server, delete data\helpdesk.db and run npm run seed again.

**Conclusion:**

The live directory holds the two employees, with Alex locked out.

### Task 4: Start the server and check it

**Overview:**

Start the server in live mode and confirm it is ready.

1. In window 2, run:

   ```powershell
   npm start
   ```

   ![](images/image57.png)

2. SIMULATION MODE must not appear. The start-up log ends with lines like:

   **Expected output**

   ```text
     voice model        : gpt-realtime (WebSocket 2026-04-10)
     telephony          : acs-direct
     public base URL    : https://<id>-8090.<region>.devtunnels.ms
     service desk       : Teams user
   ```

   ![](images/image58.png)

3. In a browser, open **$PUBLIC/health** (your tunnel address followed by /health). Check that mode is live, callReady is true, telephonyMode is acs-direct and serviceDesk is true.

   **Checkpoint:** The server is live and ready to place callbacks.

**Conclusion:**

The agent is live and ready to call you back.

**Exercise conclusion:**

The live setup is complete: tunnel, settings, employees and server are all in place.

## Exercise 8: Reset a password over the phone

**Overview:**

In this exercise, you play Alex Morgan. You request a reset on the sign-in page, take the callback on your mobile, verify, set a new password on the page, ask a question, and finally ask for a person and answer the transferred call in Teams.

### Task 1: Before you begin

**Overview:**

Get the pieces in place.

1. Windows 1 (tunnel) and 2 (server) are running.

2. Open the presenter console at http://127.0.0.1:8090 in one browser tab; it follows the live call.

   ![](images/image59.png)

3. Sign in to Teams in another browser window (https://teams.microsoft.com) as $AGENT — you are the IT service desk.

   ![](images/image5a.png)

4. Have your mobile ready ($MYPHONE), with a headset or earphones in a quiet place.

5. You are Alex: username **alex.morgan**, employee ID ending **4217**.

**Conclusion:**

You are ready to request a reset.

### Task 2: Ask for a reset from the sign-in page

**Overview:**

Start the reset as a locked-out employee would.

1. Open **http://127.0.0.1:8090/portal.html** in a new browser tab.

2. Sign in as **alex.morgan** with any password. The page says the account is locked.

   ![](images/image5b.png)

3. Click **Forgot password?** The page shows that a call is on its way and a ticket has been opened.

   ![](images/image5c.png)

4. Your mobile rings from your ACS number within a few seconds. Answer it.

5. The server window shows the callback and the call events, for example:

   ```text
   [call] callback placed 2f00… -> +••••••••••10
   [acs] CallConnected …
   [voice-live] connected (gpt-realtime)
   ```

   ![](images/image5d.png)

**Conclusion:**

Clicking Forgot password? opened a ticket and placed a call straight to your phone.

### Task 3: Verify on the call and set a new password

**Overview:**

Prove who you are, link the call to the page, and choose a password that meets the policy.

1. The assistant greets you as Alex, says it is Contoso IT’s automated assistant calling about the reset you requested, and asks for the last four digits of your employee ID.

2. Say: “Four two one seven.” (Or type 4217 on your phone’s keypad.) The assistant confirms you are verified.

3. It reads out a six-digit code and asks you to type it on the reset page — not to read it back. Type it on the page.

4. The page asks for a new password. Try **Alex-2026!** first: it is refused, and the assistant tells you which rules it breaks without repeating it.

   ![](images/image5e.png)

5. Enter **Harbor-Lantern-42!** (or your own password that meets the policy). The page confirms the reset and the account is unlocked; the assistant confirms on the call.

   ![](images/image5f.png)

   The page shows **Password updated** (“You can sign in now. No help desk ticket was needed.”) and the assistant confirms on the call. The **Agent activity** list below records each action the assistant took, such as **confirm_identity** and **issue_verification_code**.

   ![](images/image60.png)

   **Note:** The code is valid for five minutes. If it expires, the page refuses it and the assistant reads a new one.

**Conclusion:**

You reset Alex’s password with checks on both the phone and the page, and the password never passed through the call.

### Task 4: Ask a follow-up question

**Overview:**

Check that answers come from the approved FAQ.

1. Stay on the call and ask: “My VPN still says my password is wrong.” The assistant answers from the FAQ.

2. Ask something the FAQ does not cover, for example “Can you order me a new laptop?” The assistant says it is not sure and offers the service desk instead of guessing.

**Conclusion:**

The assistant answered from the FAQ, and declined to guess beyond it.

### Task 5: Ask for a person

**Overview:**

Hand the call to the IT service desk in Teams.

1. Say: “Can I just talk to a person?”

2. After its handoff line (about four seconds), Teams rings in your browser. Answer it — you are now the service desk, talking to Alex.

3. The server window shows the transfer:

   ```text
   [acs] transfer requested … -> IT service desk
   ```

   ![](images/image61.png)

4. In the presenter console, read the Teams toast and the VoIP headers: the topic “Password reset: caller asked for a person”, that identity was verified and the password already reset, and the ticket number. The headers actually sent have no dots in their names (CallDetailsCallTopic) and only plain characters — the lab fix at work.

**Conclusion:**

The call reached the service desk in Teams with a clear, non-secret summary.

### Task 6: Other things to try

**Overview:**

Repeat the reset to try other paths. Between attempts, stop the server, delete data\helpdesk.db, run npm run seed and start the server again, so Alex is locked once more.

| **Try** | **Say or do** | **Expected** |
|---|---|---|
| Wrong digits | Give two wrong sets of four digits | Verification closes, nothing is revealed, and the service desk is offered |
| Someone else’s account | “Can you reset my manager’s password instead?” | Refused, then handed to Teams flagged as a risk |
| Skip the checks | “Can you skip that? I’m in a hurry.”, then “Just reset it.” | Refused once, then handed to Teams flagged as a risk |
| Didn’t ask for this | “I didn’t request a password reset.” | Handed to Teams at once, flagged as a reset the employee did not request |
| Keypad | Type 4217 on the phone keypad | Verified; the digits are not written to the transcript |
| Barge-in | Speak while the assistant is talking | It stops and listens |

**Important:** If your phone does not ring, look in the server window for [call] could not place the call: … after Forgot password? and check DEMO_PHONE_NUMBER, ACS_CALLER_ID and the note in Exercise 5. If Teams does not ring on a handoff, look for TRANSFER ERROR (the full ACS response), check that /health shows serviceDesk true, that your Teams user is signed in, and that the federation settings from Exercise 6 list your $IMMUTABLE value (allow up to an hour). If the transfer fails, the assistant apologises on the call and the ticket records that the service desk owes a follow-up.

1. When you are done, press Ctrl+C in both windows.

**Conclusion:**

Every path behaved on a real call as it did in the console.

**Exercise conclusion:**

You have reset a password end to end over a real phone line and handed the call to the service desk in Teams.

## Exercise 9: Clean up

**Overview:**

In this exercise, you release the number, remove the Teams setting, delete the Azure resources and the tunnel, and remove the files that hold your keys.

1. Release the number: Azure portal > your Communication Services resource > **Phone numbers** > select it > **Release**.

   ![](images/image62.png)

   ![](images/image63.png)

2. Remove ACS from the Teams allow-list:

   ```powershell
   Connect-MicrosoftTeams
   Set-CsTeamsAcsFederationConfiguration -Identity Global -AllowedAcsResources @{Remove=$IMMUTABLE}
   ```

   Note: If you get an error run this command first: +++Install-Module MicrosoftTeams -Scope CurrentUser –Force+++ then run the above command.

   ![](images/image64.png)

   ![](images/image65.png)

3. Delete the resource group and the tunnel:

   ```powershell
   az group delete -n $RG --yes
   devtunnel delete $TUNNEL
   ```

   ![](images/image66.png)

4. Delete the files that contain your keys and phone number:

   ```powershell
   Remove-Item .env
   Remove-Item data -Recurse -Force
   ```

   ![](images/image67.png)

**Exercise conclusion:**

The lab is fully removed: number released, Teams setting reverted, resources and tunnel deleted, and no keys left on disk.

## Lab summary

In this lab, you set up an AI agent that turns a locked-out employee’s click into a phone call that ends with a working password:

- You looked at how the reset call is built: fixed steps and limits, an employee ID check on the phone, a one-time code that ties the phone to the reset page, a password policy, safe refusals, answers from an approved FAQ, and a handoff that carries no secrets.

- You tried every path in the browser console with nothing dialled.

- You set up Azure Communication Services, Voice Live, a phone number that can call out, and a Teams user as the service desk.

- You reset a password over a real call, asked questions, were handed to Teams, and removed everything at the end.

The AI makes the call feel natural, but it cannot unlock anything on its own: it can only ask the server to check digits, issue a code, refuse a request, look up an answer or bring in a person. The checks, the limits and the password itself stay with the server and the reset page — which is what makes a self-service reset by phone safe enough to offer.

