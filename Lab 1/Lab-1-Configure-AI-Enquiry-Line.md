# Lab 1 - Configure an AI Enquiry Line with Teams Phone Agent and Copilot Studio

## Overview

In this lab you build an AI-answered phone line for a training institute, with no code and no phone number. A **Teams Phone Agent** answers course questions from a course catalogue and, when a caller wants to enroll, hands the call to a **Copilot Studio** voice agent that collects the enrolment details using condition logic.

**Scenario:** **Contoso Learning Academy** runs instructor-led Microsoft courses. Most calls are routine questions — batch dates, fees, format — or enrollment requests. The AI front line handles both, at any hour.

![call flow · two destinations, conditions in Copilot Studio](images/image1.png)

The Teams Phone Agent decides where each call goes from the caller's words, using the descriptions you write. The enrollment assistant then branches on the caller's answers: classroom versus online.

**What you build:**

- A Teams Phone Agent with an AI greeting and two menu options: **Course information** and **Enroll**.

- A **Questions and Answers** tool backed by the course catalogue document.

- A Copilot Studio basic voice agent with an enrollment topic, condition branches and a read-back.

## Exercise 1 — Prepare the environment

In this exercise you confirm that billing and the agent's resource account are ready.

### Task 1 — Prepare the resource account

In this task, you create and license the resource account that gives the agent an identity callers can reach. It is the "number" of the agent, even though no phone number is needed for this lab.

1.  In **Teams Admin Center** (https://admin.teams.microsoft.com/), go to **Voice → Resource accounts**. Select **+ Add**.

![](images/image2.png)

![](images/image3.png)

2.  Enter the below details.

    - **Display name:** Contoso Learning (this is the name callers search for in Teams).

    - **Username:** contoso-learning@\<your-domain\>.

    - **Resource account type:** **Teams Phone Agent**. Select **Save**.

> ![](images/image4.png)

3.  In **M365 admin center** (https://admin.microsoft.com)**→ Users → Active users**, open the Contoso Learning Resource account.

> ![](images/image5.png)

4.  Select the **Licenses and apps** tab and select **United States under Location.**

> ![](images/image6.png)

5.  Select the checkbox against **Microsoft Teams Phone Resource Account** and select **Save changes**. Leave **Sign-in blocked** as it is — resource accounts are meant to be blocked.

> ![](images/image7.png)
>
> ![](images/image8.png)

6.  Add the resource account to the billing group. Select **Teams & groups → Security groups.** Select your security group – **TPA-Billing-Pilot**.

![](images/image9.png)

7.  Select **Members -\> View all and manage members -\> Add members**.

![](images/image10.png)

8.  Select the Resource account **Contoso Learning** and click **Add**.

![](images/image11.png)

![](images/image12.png)

9.  Back in **Teams admin center → Voice → Resource accounts**, the account shows **Licensed**.

![](images/image13.png)

## Exercise 2 — Build the Teams Phone Agent with the course catalogue

In this exercise you create the **Teams Phone Agent**, give it the course catalogue as its knowledge, and test it with natural questions.

### Task 1 — Add the course catalogue as a knowledge tool

In this task, you give the agent its knowledge. A **Questions and Answers** tool lets the agent answer from a document you control instead of from general AI knowledge, so answers stay accurate and on-topic.

1.  In Teams admin center (https://admin.teams.microsoft.com), go to **Voice → Templates and resources → Tools → Add**.

![](images/image14.png)

2.  **Name:** Course catalogue.

3.  **Description:** Courses, batch dates, fees, duration, online or classroom format, class timings, prerequisites, certification, exam vouchers, enrollment process, payment and refunds at Contoso Learning Academy.

4.  **Tool type:** **Questions and Answers**.

![](images/image15.png)

5.  Under **Question and Answers – setup**, select **Knowledge – Document**.

![](images/image16.png)

6.  Upload **Contoso-Learning-Course-Catalogue.docx** from **C:\LabFiles**.

![](images/image17.png)

7.  Select **Save**. *Course catalogue* now appears in the Tools list.

![](images/image18.png)

![](images/image19.png)

### Task 2 — Create the Teams Phone Agent

In this task, you create the Teams Phone Agent itself, connect it to the resource account, and set its language, voice and business hours.

1.  In **TAC**, go to **Voice → Auto attendants → Add → Teams Phone Agent**.

![](images/image20.png)

2.  Under **General info**, give the below details.

    - Name: **Contoso Learning enquiry line**

    - How callers reach this Teams Phone Agent: **Through a resource account** -\> **Assign resource account** → Search for **Contoso** and select **Contoso Learning** → **Add**.

> ![](images/image21.png)
>
> ![](images/image22.png)

- Select **Save**.

> ![](images/image23.png)

- **Language:** English (United States)

- **Voice:** Ava.

- **Operator:** **No operator**.

- Select **Next**.

> ![](images/image24.png)

3.  **Call handling hours:**

    - **Time zone:** (UTC-08:00) Pacific Time (US & Canada) (or your own).

    - **Business hours:** set them to cover the time you will be testing. To enable testing anytime, let’s select **Use the same hours every day** and then switch the toggle, **Open 24 hours, 7 days a week** ON. You'll narrow them in Task 4.

    - Select **Next**.

> ![](images/image25.png)

### Task 3 — Set up the greeting and the first menu option

In this task, you define what callers hear first and the first thing the agent can do for them. The menu option's description is what the AI uses to decide that a question belongs to the course catalogue.

1.  **Business-hours call flow:**

    - Select **Add a greeting message** and enter: Thanks for calling Contoso Learning Academy. I'm an AI assistant. I can answer questions about our courses, batch dates and fees, or help you enroll. How can I help?

> ![](images/image26.png)

- Scroll down to the **Call routing options** and select **Play menu options**.

- **Add a greeting message** – “Thanks for calling Contoso Learning Academy. I'm an AI assistant. I can answer questions about our courses, batch dates and fees, or help you enroll. How can I help?”.

- Under **Set menu options**, select, **+ Assign a dial key**

> ![](images/image27.png)

2.  Fill the details as below

| Field         | Value                                                                                                                                                                                                                  |
|---------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Dial key      | 1                                                                                                                                                                                                                      |
| Voice command | Course information                                                                                                                                                                                                     |
| Description   | Questions about courses, course content, upcoming batch dates, duration, fees, discounts, online or classroom format, class timings, prerequisites, exam vouchers, certificates, payment options and the refund policy |
| Destination   | **Tools** → **Course catalogue**                                                                                                                                                                                       |

3.  Directory search - **None**. Select **Next**.

![](images/image28.png)

4.  **After-hours call flow:**

    - Choose **Play menu options**

    - Add greeting message - "Our admissions team is available Monday to Saturday, 9 AM to 6 PM".

    - Select **+ Assign a dial key**

> ![](images/image29.png)

- Fill in the details as below

| Field         | Value                                                                                                                                                                                                                  |
|---------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Dial key      | 1                                                                                                                                                                                                                      |
| Voice command | Course information                                                                                                                                                                                                     |
| Description   | Questions about courses, course content, upcoming batch dates, duration, fees, discounts, online or classroom format, class timings, prerequisites, exam vouchers, certificates, payment options and the refund policy |
| Destination   | **Tools** → **Course catalogue**                                                                                                                                                                                       |

![](images/image30.png)

5.  Select **Next** through **Holidays**, **Templates**, **Dial scope** and **Authorized users**, then **Submit**.

![](images/image31.png)

6.  Wait **15–30 minutes** before testing.

### Task 4 — Test the agent

In this task, you confirm by voice that the agent understands natural questions, answers them correctly from the catalogue, declines what it shouldn't answer, and follows business hours.

1.  Open Teams from the browser. Select **Calls**, search **Contoso Learning** (the resource account name, not the agent name) and call it.

2.  You should hear the greeting and AI disclaimer. Ask each question naturally and compare the answer:

| \#  | Ask                                                   | Expected answer (from the catalogue)                           |
|-----|-------------------------------------------------------|----------------------------------------------------------------|
| 1   | When does the next Azure Administrator batch start?   | 16–19 November 2026, online; a classroom batch 14–17 December  |
| 2   | How much is the Power BI course?                      | \$1,795 plus applicable sales tax                              |
| 3   | Is Copilot Studio taught online or in a classroom?    | Both: classroom 30 Nov–1 Dec in Redmond, online 28–29 December |
| 4   | Do I need any experience for AZ-104?                  | AZ-900 level knowledge or about six months of Azure            |
| 5   | If I cancel five days before, do I get my money back? | 50% refund; full refund only 7+ days before                    |
| 6   | Is the Microsoft exam included?                       | No; vouchers can be added for \$165                            |
| 7   | What's the weather in Seattle today?                  | Declines politely and offers course help                       |

3.  **Business hours check:** set business hours back to Monday–Saturday, 09:00–18:00, submit, wait, and call outside those hours. You should hear the after-hours greeting and still get answers.

## Exercise 3 — Hand enrollments to a Copilot Studio voice agent 

In this exercise you build **Contoso Enrolment Assistant**, a Copilot Studio basic voice agent that collects enrollment details and uses **condition** nodes to change what it says. Then you connect it to the Teams Phone Agent as a second menu option, so the AI decides from the caller's words which agent handles the call.

The logic lives in two places:

| Where                | Decides                         | How                                                              |
|----------------------|---------------------------------|------------------------------------------------------------------|
| Teams Phone Agent    | Which destination gets the call | Matches the caller's words to each menu option's **Description** |
| Copilot Studio agent | What to say next                | **Condition** nodes on the caller's answers: classroom vs online |

### Task 1 — Prepare the Copilot Studio environment

In this task, you create a Power Platform environment that can host a basic voice agent.

1.  Go to **https://admin.powerplatform.microsoft.com → Manage → Environments**. If an environment you can use is already listed (and is not marked *early release*), skip to step 3.

2.  Select **+ New** and enter:

    - **Name:** Dev env

    - **Get new features early:** **No** (an early-release environment can't enable basic voice)

    - **Region:** United States

    - **Type: Developer**

    - **Add a Dataverse data store:** **Yes**

    - Select **Next**, keep English and USD, and select **Save**. Wait until the state shows **Ready** (a few minutes).

> ![](images/image32.png)
>
> ![](images/image33.png)
>
> ![](images/image34.png)

3.  Go to **https://copilotstudio.microsoft.com** and choose this environment from the left pane.

![](images/image35.png)

4.  Click on the 3 dots next to the user name and select **Open classic experience**. Select **Skip feedback** to continue to the classic experience.

![](images/image36.png)

### Task 2 — Create the agent and turn on voice

In this task, you create the Copilot Studio agent and configure it for the phone channel: basic voice, classic orchestration and no authentication, plus the catalogue as knowledge for side questions.

1.  Select **Agent**.

![](images/image37.png)

2.  Enter the Name as **Contoso Enrollment Assistant** and select **Create**.

![](images/image38.png)

3.  Select **Edit** under the **Details** pane and enter the **Description** as below.

> **Takes enrollment requests for Contoso Learning Academy courses over the phone.**

Select **Save**.

> ![](images/image39.png)

4.  Edit the **Instructions** and enter, **You help callers enroll in a Contoso Learning Academy course. Ask one question at a time. Keep every reply to one short sentence. Never ask for card numbers, bank details or passwords.**

![](images/image40.png)

5.  **Turn on voice:** **Settings → Voice** → turn on **Enable voice**.

6.  Turn on **Make voice the default modality for creating and updating within topics**, choose **Basic**, select **Save**.

![](images/image41.png)

![](images/image42.png)

7.  Select **Save** in the Confirm voice type dialog.

![](images/image43.png)

8.  Choose **Generative AI** from the left pane → select **No - Use classic orchestration** and select **Save**.

Generative orchestration isn't supported on the Teams Phone Agent channel and adds delay.

> ![](images/image44.png)

9.  Select **Security** from the left pane. Select **Authentication** → **No authentication** → **Save**.

![](images/image45.png)

10. Confirm **Save**.

![](images/image46.png)

11. **Close** the Settings pane.

![](images/image47.png)

12. Back in the agent **Overview** page, scroll down to the **Knowledge** section and select **Add knowledge.**

![](images/image48.png)

13. Select **Upload file** and upload the **Contoso-Learning-Course-Catalogue.docx** from **C:\LabFiles**. This lets the assistant answer a quick question in the middle of an enrollment.

![](images/image49.png)

14. Select **Add to agent**.

![](images/image50.png)

![](images/image51.png)

### Task 3 — Build the enrollment topic

In this task, you build a structured conversation that collects the enrollment details — course, format, name and call-back number — into variables, one question at a time.

1.  Go to **Topics → + Add a topic → From blank**.

![](images/image52.png)

2.  Name it **Enroll in a course**. Select Edit in the **Trigger** node -\>**The user says a phrase**.

![](images/image53.png)

3.  Enter and add phrases like enroll, I want to enroll, register for a course, sign up, book a seat, join a batch.

![](images/image54.png)

4.  Select **+** under the trigger → **Ask a question**:

![](images/image55.png)

- **Message:** Which course would you like to join?

- **Identify:** **Multiple choice options**. Add the options Azure Fundamentals, Azure Administrator, Power BI, Microsoft 365 Copilot, Teams Phone, Copilot Studio.

> **Note:** You can also open each option's **synonyms** and add how people say it: AZ-900 for Azure Fundamentals; AZ-104 for Azure Administrator; PL-300, data analyst for Power BI; Copilot for business users for Microsoft 365 Copilot; Teams voice for Teams Phone.

- **Save user response as:** select the variable and rename it **Course**.

> ![](images/image56.png)

- Delete the condition branches that are added automatically under the question, so the topic continues in a single line.

> ![](images/image57.png)

5.  Add another **question node.**

    - Question - **Would you prefer an online or a classroom batch?**

    - Identify - **Multiple choice options**

    - Options - **Online**, **Classroom**

    - Variable name - Rename to **Format**

    - Delete the automatic branches.

> ![](images/image58.png)

6.  Add another **question node.**

    - Question - **May I have your full name?**

    - Identify - **User's entire response**

    - Variable name - Rename to **CallerName**

> ![](images/image59.png)

7.  Add another **question node.**

    - Question - **What's the best mobile number to call you back on? You can also type it on your keypad.**

    - Identify - **Phone number**

    - Variable name - Rename to **CallbackNumber**

> ![](images/image60.png)

8.  Select **Save**.

![](images/image61.png)

### Task 4 — Add the condition logic

Make the conversation react to the caller's answers. Condition node changes what the agent says, classroom versus online.

**Condition 1 — classroom or online**

1.  Under the last question, select **+ → Add a condition**.

![](images/image62.png)

2.  In the condition, choose variable **Format**, operator **is equal to**, value **Classroom**.

3.  Under that branch, add **Send a message:** Classroom batches are held at the Contoso Learning Center, 100 Contoso Way, Suite 200, Redmond, Washington, from 9:30 AM to 5:30 PM Pacific Time.

4.  Under **All other conditions**, add **Send a message:** Online batches run live on Microsoft Teams. We'll email you the joining link before the course starts.

> ![](images/image63.png)

9.  Below the point where both branches join, add **Send a message:** Thanks, {CallerName}. I've noted {Course}, {Format} batch. Admissions will call you on {CallbackNumber} within one working day to confirm your seat and payment. Insert each variable with the **{x}** button.

> ![](images/image64.png)

10. Add **+ → Topic management → End conversation**.

![](images/image65.png)

11. Select **Save**.

![](images/image66.png)

12. Go to **Topics → System → Conversation Start**.

![](images/image67.png)

13. Replace its message with **Hi, I'm the Contoso enrollment assistant. I'll take a few details so admissions can confirm your seat.**

14. Below it, add **+ → Topic management → Go to another topic →** *Enroll in a course*. **Save.**

![](images/image68.png)

15. **Save** the topic.

![](images/image69.png)

### Task 5 — Test and publish

Check the topic logic before it reaches callers, then publish the agent to the Teams Phone Agent channel. In *Speech & DTMF* test mode the pane shows exactly what a caller would hear and offers a keypad, but you type the caller's words — live voice testing happens by phone in Task 7.

1.  Open the **Test** pane, select **Reset** (↻), and type through two enrollments:

    - Azure Fundamentals → Classroom → your name → a 10-digit number.

> ![](images/image70.png)
>
> ![](images/image71.png)

2.  Publish to the phone channel: **Channels → Teams Phone Agent**.

![](images/image72.png)

3.  Select **Add channel**.

![](images/image73.png)

4.  Select **turn on Telephony** to enable voice.

![](images/image74.png)

![](images/image75.png)

5.  Check the confirmation and then close the pane.

![](images/image76.png)

6.  Select **Publish** and confirm **Publish**.

> ![](images/image77.png)

7.  After a few minutes, from the Teams Admin Center, select **Voice → Templates and resources → Agents and Queues**, confirm that **Contoso Enrollment Assistant** is listed.

![](images/image78.png)

### Task 6 — Add the Enroll option to the Teams Phone Agent

In this task, you connect the two agents. A second menu option, with its own description, lets the Teams Phone Agent decide from the caller's words whether to answer from the catalogue or hand the call to Copilot Studio.

1.  In **TAC → Voice → Auto attendants**, select **Contoso Learning enquiry line**

![](images/image79.png)

2.  Select **Business-hours call flow**.

3.  Scroll down and select **+ Assign a dial key**. Fill in the below details and select **Next**.

| Field         | Value                                                                                           |
|---------------|-------------------------------------------------------------------------------------------------|
| Dial key      | 2                                                                                               |
| Voice command | Enrol                                                                                           |
| Description   | The caller wants to enrol, register, sign up, book a seat or join a batch for a specific course |
| Destination   | **Agents and Queues** → *Contoso Enrolment Assistant* (leave the tag template empty)            |

![](images/image80.png)

4.  Repeat steps 1–3 in the **After-hours call flow**.

5.  Select **Submit** and wait for few minutes.

### Task 7 — Test end to end

In this task, you perform the testing of the agents end to end. Check the complete caller journey by voice — a question answered by the Teams Phone Agent, and an enrolment handed to Copilot Studio with the classroom/online condition working.

1.  Call **Contoso Learning** from Teams opened in the browser.

![](images/image81.png)

| \#  | Say                                              | Expected                                                   |
|-----|--------------------------------------------------|------------------------------------------------------------|
| 1   | "What does the Power BI course cost?"            | Answered by the Teams Phone Agent from the catalogue       |
| 2   | "I'd like to enrol in Azure Fundamentals course" | Handed to the enrolment assistant; course question.        |
| 3   | Answer *Classroom*, your name and number         | Redmond address and timings, then the read-back; call ends |
| 4   | Press **2** at the start                         | Goes straight to the enrolment assistant                   |

## Summary

In this lab you built an AI-answered enquiry line for Contoso Learning Academy that handles the two most common kinds of call — course questions and enrolment requests — without a person on the line and without writing code.

**What you built:**

- A **Teams Phone Agent** that greets callers with an AI disclosure, follows business hours, and decides from the caller's own words where each call should go.

- A **course catalogue knowledge tool** that lets the agent answer questions about batch dates, fees, format, prerequisites, refunds and certification from a document you control.

- A **Copilot Studio basic voice agent** that collects enrolment details in a structured conversation and uses **condition** nodes to tailor what it says: the classroom address or the online joining details.

- A **hand-off** between the two agents through a second menu option, so routing and follow-up are each done by the tool best suited to them.

**Key takeaways:**

1.  **Descriptions are the routing logic.** The Teams Phone Agent chooses a destination by matching what callers say to each option's description, so clear, non-overlapping descriptions matter more than dial keys.

2.  **Grounded answers need owned content.** The agent is only as accurate as the catalogue; short, spoken-style answers and a regular re-upload routine keep it right.

3.  **Use each tool for what it does best.** Teams Phone Agent handles the front door and free-form understanding; Copilot Studio handles step-by-step conversations with variables and conditions.

4.  **Voice changes the design.** Keep prompts short, avoid long option lists, allow keypad input for numbers, and test on a real call — the test pane checks logic, not speech.

5.  **Licensing and billing come first.** A licensed resource account and a Copilot Credits spending policy are prerequisites; most setup problems trace back to them.
