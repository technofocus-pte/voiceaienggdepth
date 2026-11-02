# Lab 6 - Build Zava clinic’s Real-Time Voice receptionist with live appointment booking in Copilot Studio

## Lab Objective

Learners build **Zava Clinic Scheduler**, a Copilot Studio real-time voice agent that books, reschedules and cancels clinic appointments by voice. It reads and writes live appointment slots in SharePoint through Power Automate agent flows, and it sends a confirmation email after each booking.

**Scenario:** Zava Clinic offers three services: General Consultation, Dental Check-up and Physiotherapy. It's open Monday to Saturday, 09:00–18:00. Callers want to book, move or cancel an appointment, or ask simple questions about the clinic.

**By the end, learners can:**

1. Store appointment data in a SharePoint list and expose it through agent flows.
2. Create a real-time voice agent and configure its voice, speech sensitivity, DTMF, silence detection and latency messaging.
3. Combine model-led conversation (instructions) with deterministic topics, such as a spoken confirmation before booking.
4. Design helper topics with typed inputs and outputs that call agent flows.
5. Test the agent by voice and keypad, and confirm the results in SharePoint and in the confirmation email.

**The key design idea:** the agent reaches appointment data only through three helper topics: Check Availability, Book Appointment and Cancel Appointment. Each topic has fixed inputs and outputs and calls one flow.

![Lab architecture](images/image1.png)

Lab architecture · caller to SharePoint and Outlook

The model handles the conversation. Only the three helper topics call flows, and only the flows touch SharePoint and Outlook.

**Learner setup**

- A headset with a microphone. Laptop speakers cause echo, and the echo triggers barge-in.
- A quiet space. Background noise affects speech detection.

## Exercise 1: Create the appointments list

In this exercise, you create a SharePoint communication site called Zava Voice Lab and a list called ZavaAppointments inside it. The list is Zava Clinic’s booking system: it holds every appointment slot, and the agent flows you build in Exercise 2 read from it and write back to it. You also load 16 sample slots so that the agent has real data to book against.

### Task 1: Create the SharePoint site

In this task, you sign in to Microsoft 365 and create a communication site from the standard template. The site gives the lab its own space for the appointments list, separate from any existing content in your tenant.

1. Sign in to https://office.com with your tenant credentials.

2. From the app launcher (**Apps**), select **SharePoint**.

   ![Screenshot 2](images/image2.png)

3. In the left pane, select **Build**, then select **Site**.

   ![Screenshot 3](images/image3.png)

4. Select the **Communication site** tab, then select the **Standard communication** template.

   ![Screenshot 4](images/image4.png)

5. Review the template preview and select **Use template**.

   ![Screenshot 5](images/image5.png)

6. Enter **Zava Voice Lab** as the site name. Keep the generated site address and the default language, then select **Create site**.

   ![Screenshot 6](images/image6.png)

You have created the Zava Voice Lab communication site. It is ready to hold the appointments list that you create in the next task.

### Task 2: Create the appointments list and add the slots

In this task, you create a blank list, rename its Title column to SlotId and add seven text columns. You then paste the 16 sample slots from the excel file, making sure that dates and times keep the exact text format the flows filter on.

1. On the site home page, select the arrow next to **+ New**, then select **List**.

   ![Screenshot 7](images/image7.png)

2. Under **Create from blank**, select **List**.

   ![Screenshot 8](images/image8.png)

3. Enter **ZavaAppointments** as the list name, keep **Show list in site navigation** selected, and select **Create**.

   ![Screenshot 9](images/image9.png)

4. Select the **Title** column header, then select **Column settings > Rename**.

   ![Screenshot 10](images/image10.png)

5. Enter **SlotId** and select **Save**. Only the display name changes; the column’s internal name stays Title, which the flows use later.

   ![Screenshot 11](images/image11.png)

6. Select **+ Add column**, select **Text**, then select **Next**.

   ![Screenshot 12](images/image12.png)

7. Enter **Service** in **Name**, keep the type as **Single line of text**, and select **Save**.

   ![Screenshot 13](images/image13.png)

8. Repeat the previous two steps to add six more **Single line of text** columns. Type each name exactly as shown, with no spaces:
   - **AppointmentDate**
   - **AppointmentTime**
   - **Status**
   - **BookingRef**
   - **PatientName**
   - **PhoneNumber**

   > Note: SharePoint fixes a column’s internal name when the column is first created, and the flow filters use these internal names. If you mistype a name, delete the column and create it again rather than renaming it.

9. Open **ZavaAppointments.xlsx** from **C:\LabFiles**. **Copy** the data rows, leaving the Header row.

10. In the list, select **Edit in grid view**, click the first cell under **SlotId** (the **Add new item** row) and press **Ctrl+V** to paste the rows. Then select **Exit grid view**.

    ![Screenshot 14](images/image14.png)

11. Check the list. You should see 16 slots with the Status **Available**, dates such as **2026-11-10** and times such as **09:00**.

    ![Screenshot 15](images/image15.png)

    > Important: The flows match dates and times as exact text. If the list shows values such as 11/10/2026 or 9:00, correct them in grid view to the 2026-11-10 and 09:00 format, otherwise the flows return no slots.

You have created the ZavaAppointments list with eight columns and 16 available slots, stored in the exact format that the flows expect.

In this exercise, you created the SharePoint site and the appointments list that act as Zava Clinic’s booking system. Every booking, reschedule and cancellation that the agent makes later in this lab is recorded in this list, so you can watch it change during your test calls.

## Exercise 2: Build the agent flows

In this exercise, you build four agent flows: Check Availability, Book Appointment, Cancel Appointment and Send Confirmation. Each flow starts with **When an agent calls the flow** and ends with **Respond to the agent**, and its inputs and outputs match one helper topic that you build in Exercise 4. Building the flows first means each topic can call its flow as soon as you create it.

> Filter tip: In a Filter Query, type the text and insert each dynamic value directly, with no space between a single quote and the value. A filter such as Service eq ' Dental Check-up' (with a space) finds nothing. In this guide, [Service] means “insert the Service value from the dynamic content”; do not type the brackets or a + sign.

### Task 1: Build the Check Availability flow

In this task, you build the flow that finds free slots for one service on one date. It reads the matching rows from SharePoint, turns their times into a comma-separated list, and returns that list to the agent together with a flag that says whether anything was found.

1. In Copilot Studio, select **Flows** in the left pane, then select **+ New agent flow**.

   ![Screenshot 16](images/image16.png)

2. In **Add a trigger**, search for **skills** and select **When an agent calls the flow**.

   ![Screenshot 17](images/image17.png)

3. In the trigger, select **+ Add an input**.

   ![Screenshot 18](images/image18.png)

4. Select **Text** as the input type.

   ![Screenshot 19](images/image19.png)

5. Name the input **Service**. Repeat the previous two steps to add a second **Text** input named **PreferredDate**.

   ![Screenshot 20](images/image20.png)

6. Select **+** below the trigger, search for **SharePoint** and select **Get items**.

   ![Screenshot 21](images/image21.png)

7. Select the **Zava Voice Lab** site in **Site Address** and **ZavaAppointments** in **List Name**. Open **Advanced parameters** and select **Show all**.

   ![Screenshot 22](images/image22.png)

8. In **Filter Query**, type Status eq 'Available' and Service eq ', then select the lightning icon (dynamic content) and choose **Service** under **When an agent calls the flow**. Type ' and AppointmentDate eq ', insert **PreferredDate** the same way, and finish with '. The query reads: Status eq 'Available' and Service eq '[Service]' and AppointmentDate eq '[PreferredDate]'.

   ![Screenshot 23](images/image23.png)

9. In **Order By**, enter AppointmentTime asc. Leave **Top Count** empty so that all free slots are returned.

   ![Screenshot 24](images/image24.png)

10. Select **+** below Get items, search for **data operation** and select **Select**.

    ![Screenshot 25](images/image25.png)

11. In **From**, select the lightning icon and choose **body/value** from **Get items**. Then select the **Switch Map to text mode** icon next to **Map**.

    ![Screenshot 26](images/image26.png)

12. Click in **Map** and select the **fx** icon.

    ![Screenshot 27](images/image27.png)

13. Enter the expression item()?['AppointmentTime'] and select **Add**.

    ![Screenshot 28](images/image28.png)

14. Select **+** below Select, search for **skills** and select **Respond to the agent**.

    ![Screenshot 29](images/image29.png)

15. Select **Text** as the output type.

    ![Screenshot 30](images/image30.png)

16. Name the output **AvailableSlots**, with no trailing space. Click in the value box and select the **fx** icon.

    ![Screenshot 31](images/image31.png)

17. Enter the expression join(body('Select'), ', ') and select **Add**.

    ![Screenshot 32](images/image32.png)

18. Select **+ Add an output**, choose **Yes/No** and name it **SlotsFound**. For its value, select **fx**, enter greater(length(body('Get_items')?['value']), 0) and select **Add**.

    ![Screenshot 33](images/image33.png)

19. Select the flow name at the top left and rename it **Zava - Check Availability**. Select **Save draft**, then **Publish**.

    ![Screenshot 34](images/image34.png)

20. Select **Test**, choose **Manually**, then select **Test**.

    ![Screenshot 35](images/image35.png)

21. Enter **Dental Check-up** for **Service** and **2026-11-10** for **PreferredDate**, then select **Run flow**.

    ![Screenshot 36](images/image36.png)

22. Wait for the run to finish. Every step shows a green check mark.

    ![Screenshot 37](images/image37.png)

23. Select **Respond to the agent** and select **Show raw outputs**. The body shows availableslots as **10:00, 11:30, 15:00** and slotsfound as **true**.

    ![Screenshot 38](images/image38.png)

You have built, published and tested the Check Availability flow. Given a service and a date, it returns the free times from SharePoint and a Yes/No flag.

### Task 2: Build the Book Appointment flow

In this task, you build the flow that books one slot. It looks for an available row that matches the service, date and time, marks it as Booked with the caller’s details and a new booking reference, and returns the reference and a success flag to the agent.

1. Select **Flows**, then select **+ New agent flow**.

   ![Screenshot 39](images/image39.png)

2. Add the **When an agent calls the flow** trigger with five **Text** inputs: **PatientName**, **PhoneNumber**, **Service**, **AppointmentDate** and **AppointmentTime**.

   ![Screenshot 40](images/image40.png)

3. Select **+** below the trigger, search for **initialize** and select **Initialize variable**.

   ![Screenshot 41](images/image41.png)

4. Set **Name** to varRef and **Type** to **String**, and leave **Value** empty. Add a second **Initialize variable** with **Name** varSuccess, **Type** **Boolean** and **Value** false.

   ![Screenshot 42](images/image42.png)

5. Add **SharePoint > Get items**, select your site and list, open **Advanced parameters** and set:
   - **Filter Query:** Status eq 'Available' and Service eq '[Service]' and AppointmentDate eq '[AppointmentDate]' and AppointmentTime eq '[AppointmentTime]'
   - **Top Count:** 1

   ![Screenshot 43](images/image43.png)

6. Select **+** below Get items, search for **condition** and select **Condition**.

   ![Screenshot 44](images/image44.png)

7. Click in the first **Choose a value** box, select **fx**, enter length(body('Get_items')?['value']) and select **Add**.

   ![Screenshot 45](images/image45.png)

8. Set the operator to **is greater than** and enter 0 in the right-hand box. Delete the empty second row by selecting its **…** menu and **Delete**.

   ![Screenshot 46](images/image46.png)

9. In the **True** branch, add the following actions:

   1. **Set variable**: **Name** varRef, **Value** expression concat('ZAV-', rand(1000, 9999)). This creates a booking reference such as ZAV-4821.

   2. **SharePoint > Update item** on your site and list, with these values:
      - **Id:** expression first(body('Get_items')?['value'])?['ID']
      - **Title:** expression first(body('Get_items')?['value'])?['Title']
      - **Status:** Booked
      - **BookingRef:** select the lightning icon and choose **varRef** under **Variables**
      - **PatientName** and **PhoneNumber:** the trigger inputs with the same names

      ![Screenshot 47](images/image47.png)

      ![Screenshot 48](images/image48.png)

   3. **Set variable**: search for **set variable**, select **Set variable**, then set **Name** to varSuccess and **Value** to true.

      ![Screenshot 49](images/image49.png)

      ![Screenshot 50](images/image50.png)

      > Leave the False branch empty. If no slot is found, varRef stays empty and varSuccess stays false, and the next step returns those values to the agent.

10. Below the condition, outside both branches, select **+**, search for **skills** and select **Respond to the agent**.

    ![Screenshot 51](images/image51.png)

11. Add a **Text** output named **BookingRef**. For its value, select the lightning icon and choose **varRef** under **Variables**.

    ![Screenshot 52](images/image52.png)

12. Add a **Yes/No** output named **BookingSuccess** with the value **varSuccess**.

    ![Screenshot 53](images/image53.png)

13. Rename the flow **Zava - Book Appointment**, then select **Save draft** and **Publish**.

    ![Screenshot 54](images/image54.png)

You have built the Book Appointment flow. It books a matching slot in SharePoint and returns a ZAV booking reference, or returns BookingSuccess false when the slot is no longer free.

### Task 3: Build the Cancel Appointment flow

In this task, you build the flow that cancels a booking. It finds the booked row with the given reference, changes its status to Cancelled, and tells the agent whether the cancellation worked.

1. Select **Flows**, select **+ New agent flow**, and add the **When an agent calls the flow** trigger with one **Text** input named **BookingRef**.

   ![Screenshot 55](images/image55.png)

2. Add **Initialize variable** with **Name** varCancelled, **Type** **Boolean** and **Value** false.

   ![Screenshot 56](images/image56.png)

3. Add **SharePoint > Get items** on your site and list. In **Filter Query**, type BookingRef eq ', insert **BookingRef** from the dynamic content, then type ' and Status eq 'Booked'.

   ![Screenshot 57](images/image57.png)

4. Set **Top Count** to 1. The query reads BookingRef eq '[BookingRef]' and Status eq 'Booked'.

   ![Screenshot 58](images/image58.png)

5. Add a **Condition**. In the first box, enter the **fx** expression length(body('Get_items')?['value']), set the operator to **is greater than** and enter 0. Delete the empty second row.

   ![Screenshot 59](images/image59.png)

6. In the **True** branch, add **SharePoint > Update item** on your site and list, with these values:
   - **Id:** expression first(body('Get_items')?['value'])?['ID']
   - **Title:** expression first(body('Get_items')?['value'])?['Title']

   ![Screenshot 60](images/image60.png)

   - **Status:** Cancelled

   ![Screenshot 61](images/image61.png)

7. Below Update item, inside the True branch, add **Set variable** with **Name** varCancelled and **Value** true.

   ![Screenshot 62](images/image62.png)

8. Below the condition, outside both branches, add **Respond to the agent**. Add a **Yes/No** output named **CancelSuccess**, select the lightning icon and choose **varCancelled** under **Variables**.

   ![Screenshot 63](images/image63.png)

   ![Screenshot 64](images/image64.png)

9. Select **Save draft**.

   ![Screenshot 65](images/image65.png)

10. Rename the flow **Zava - Cancel Appointment**, then select **Save draft** and **Publish**.

    ![Screenshot 66](images/image66.png)

You have built the Cancel Appointment flow. It marks a booked slot as Cancelled and returns CancelSuccess true, or false when no booking matches the reference.

### Task 4: Build the Send Confirmation flow

In this task, you build the post-booking automation. The flow sends an email with the booking details to the front desk, which in this lab is your own mailbox, every time the agent confirms a booking.

1. Select **Flows**, then select **+ New agent flow**.

   ![Screenshot 67](images/image67.png)

2. Add the **When an agent calls the flow** trigger with five **Text** inputs: **PatientName**, **Service**, **AppointmentDate**, **AppointmentTime** and **BookingRef**.

   ![Screenshot 68](images/image68.png)

3. Select **+** below the trigger, search for **send an email (v2)** and select **Send an email (V2)** under **Office 365 Outlook**.

   ![Screenshot 69](images/image69.png)

4. If you are asked to create a connection, enter **Connection** as the connection name, select **Sign in**, and sign in with your tenant credentials.

   ![Screenshot 70](images/image70.png)

5. Fill in the email as follows:
   - **To:** your own email address. It stands in for the front-desk mailbox.
   - **Subject:** type New booking, insert **BookingRef**, then type – Zava Clinic
   - **Body:** type Patient:, insert **PatientName**, type , Service:, insert **Service**, type , Date:, insert **AppointmentDate**, type at, insert **AppointmentTime**, and finish with . Booked by the Zava voice agent.

   ![Screenshot 71](images/image71.png)

6. Below the email action, add **Respond to the agent** with a **Yes/No** output named **Sent**. For its value, select **fx**, enter true and select **Add**.

   ![Screenshot 72](images/image72.png)

7. Select **Save draft**.

   ![Screenshot 73](images/image73.png)

8. Rename the flow **Zava - Send Confirmation**, then select **Save draft** and **Publish**.

   ![Screenshot 74](images/image74.png)

9. Select **Flows** and check that all four Zava flows show the status **Published**.

   ![Screenshot 75](images/image75.png)

You have built the Send Confirmation flow, which emails the booking details to the front desk after each successful booking.

In this exercise, you built and published four agent flows that check availability, book, cancel and confirm appointments in SharePoint. The Check Availability test returned the expected times, which confirms that the list and the flows work together. Next, you create the voice agent that will call these flows.

## Exercise 3: Create the real-time voice agent

In this exercise, you create the Zava Clinic Scheduler agent, switch it to real-time voice and tune how it listens and speaks. You also set the authentication that voice testing needs and a fixed greeting that callers hear first.

### Task 1: Create the agent

In this task, you create a blank agent, name it Zava Clinic Scheduler and add a description. The description tells the model what the agent is for.

1. Go to copilotstudio.microsoft.com and select the lab environment in the environment picker at the top right.

2. Select **Agents** in the left pane, then select **+ Create blank agent**.

   ![Screenshot 76](images/image76.png)

3. Enter **Zava Clinic Scheduler** as the name and select **Create**.

   ![Screenshot 77](images/image77.png)

4. When the agent has been provisioned, select **Edit** in the **Details** section of the **Overview** tab.

   ![Screenshot 78](images/image78.png)

5. In **Description**, enter Voice receptionist that books, reschedules and cancels appointments for Zava Clinic. and select **Save**.

   ![Screenshot 79](images/image79.png)

You have created the Zava Clinic Scheduler agent and described its purpose.

### Task 2: Turn on real-time voice

In this task, you switch the agent to real-time voice and configure how it speaks and listens: the voice, how it decides that the caller has finished speaking, keypad (DTMF) input, silence handling and the message callers hear while a flow runs.

1. Select **Settings** at the top right.

   ![Screenshot 80](images/image80.png)

2. Select **Voice** and turn on **Enable voice**. Under **Voice type**, select **Real-time** and keep the model **GPT-Realtime**. The voice type can’t be changed later, so check your choice before you continue.

   ![Screenshot 81](images/image81.png)

3. Scroll down to **Phone Setup** and select **Select voice**. Choose **Coral** and select **Save**.

   ![Screenshot 82](images/image82.png)

4. Select **Languages** and make sure the agent’s primary language is **English (US)**.

5. Go back to **Voice** and, under **Phone Setup**, select **Speech input**.

   ![Screenshot 83](images/image83.png)

6. Select **Sensitivity**.

   ![Screenshot 84](images/image84.png)

7. Set **Decides user has stopped speaking** to **Based on sentence context** and **Eagerness** to **Medium**, then select **Save**. The agent now waits until the caller finishes a thought, not just until they pause.

   ![Screenshot 85](images/image85.png)

8. Go back to **Phone Setup** and select **Conversation behaviour**.

   ![Screenshot 86](images/image86.png)

9. Select **DTMF**, select **Allow DTMF**, set the values in the table below and select **Save**.

   | Setting | Value | Why |
   | --- | --- | --- |
   | DTMF -> Inter-digit timeout | 3000 ms | Gives callers time to key in a phone number |
   | DTMF -> Termination character | # | Lets callers press # to send input straight away |

   ![Screenshot 87](images/image87.png)

10. Configure **Silence detection** and **Latency messaging** with the values in the table below.

    | Setting | Value | Why |
    | --- | --- | --- |
    | Silence detection timeout | On, 7000 ms | Off by default. Without it the agent waits forever. |
    | Latency messaging | One moment while I check that for you. | Fills the gap while a flow runs |

    ![Screenshot 88](images/image88.png)

11. In **Silence detection**, select **Silence detection timeout**, enter 7000 and select **Save**.

    ![Screenshot 89](images/image89.png)

12. Select **Latency messaging**, select **Play a message**, enter One moment while I check that for you. and select **Save**.

    ![Screenshot 90](images/image90.png)

You have turned on real-time voice and set the voice, speech sensitivity, DTMF, silence detection and latency message.

### Task 3: Set authentication

In this task, you set the agent to No authentication. Voice testing in the Copilot Studio test panel only works with this setting.

1. In **Settings**, select **Security**, then select **Authentication**.

2. Select **No authentication** and select **Save**.

   ![Screenshot 91](images/image91.png)

3. In the **Save this configuration?** dialog, select **Save**.

   ![Screenshot 92](images/image92.png)

You have set authentication to No authentication, so you can test the agent by voice in the test panel.

### Task 4: Set the greeting

In this task, you replace the default greeting with a fixed message that tells callers what the agent can do. Static greeting text plays faster than a greeting that the model writes.

1. Select the **Topics** tab, select **System**, and open **Conversation Start**.

   ![Screenshot 93](images/image93.png)

2. Replace the text in the **Message** node with: Thank you for calling Zava Clinic. I can book, change or cancel an appointment, or answer questions about the clinic. How can I help?

3. Select **Save**.

   ![Screenshot 94](images/image94.png)

You have set a fixed greeting that plays at the start of every call.

In this exercise, you created the Zava Clinic Scheduler agent and configured it for real-time voice. Checkpoint: the voice settings show Real-time voice, the Coral voice, semantic VAD, DTMF with #, silence detection on and a latency message, and authentication is set to No authentication.

## Exercise 4: Instructions, knowledge and helper topics

In this exercise, you give the agent its instructions and an FAQ file, then build the three helper topics that connect the conversation to your flows. The instructions control how the agent speaks and what it may and may not do, and the helper topics are the only way it reaches appointment data.

> Treat each topic as a function the voice model calls. **Inputs** are values the model collects from the caller and passes in. **Outputs** are values the topic hands back, which the model uses to decide what to say. The descriptions tell the model what each value means and what format to use, so write them carefully.

### Task 1: Write the instructions and add knowledge

In this task, you paste the agent instructions, which set its tone, the booking, rescheduling and cancelling steps, and strict rules for dates and times. You then upload the clinic FAQ so that the agent can answer simple questions such as parking and opening hours.

1. Select the **Overview** tab, scroll to **Instructions** and select **Edit**.

   ![Screenshot 95](images/image95.png)

2. Paste the text below, then select **Save**.

   ```text
   You are the voice receptionist for Zava Clinic. You speak to callers on the phone.

   Services: General Consultation, Dental Check-up, Physiotherapy.
   Opening hours: Monday to Saturday, 09:00 to 18:00. Closed on Sunday.

   How to speak:
   - Keep every reply to one or two short sentences. Ask one question at a time.
   - Never use lists, symbols or formatting.
   - Read appointment times exactly as they appear in AvailableSlots, converted to spoken form only. For example, 15:00 is "three p.m.". Never change the hour or minutes.
   - Read booking references one character at a time, for example "Z A V, 4 8 2 1".

   Dates and times (strict rules):
   - Never choose, guess or suggest a date yourself. The date must always come from the caller.
   - If the caller has not given a date, ask: "Which day would you like?"
   - If the caller uses a relative date such as "next Tuesday" or "tomorrow", work it out from today's date, then confirm it with the day and full date, for example "That's Tuesday the 10th of November, is that right?" Wait for a yes before checking availability.
   - If the date is unclear, or the month is missing and could mean more than one date, ask the caller to confirm. Never assume.
   - Never offer a time that was not returned in AvailableSlots.
   - Never say a slot is available unless Check Availability has just returned it for that exact service and date.

   Booking an appointment:
   1. Find out which service the caller needs.
   2. Get the date from the caller and confirm it, following the date rules above.
   3. Convert the confirmed date to YYYY-MM-DD and call Check Availability.
   4. If SlotsFound is false or AvailableSlots is empty, say there are no free times that day and ask which other day they would like. Do not propose a day yourself.
   5. Read out only the times in AvailableSlots, exactly as returned, and let the caller choose one.
   6. Collect the caller's full name and phone number. Tell them they can key in the phone number and press hash.
   7. Call Book Appointment with the confirmed date and the chosen time. If BookingSuccess is true, give the booking reference. If false, say the slot could not be booked and ask whether they would like another time or day.

   Rescheduling:
   1. Ask for the booking reference and repeat it back one character at a time to confirm it.
   2. Ask which service the booking is for, and ask for the caller's full name and phone number again. Never reuse or guess these details.
   3. Ask the caller for the new date and confirm it using the date rules. Call Check Availability and let the caller choose a time from AvailableSlots.
   4. Call Book Appointment for the new slot. If BookingSuccess is false, keep the old booking and tell the caller nothing has changed.
   5. Only after the new booking succeeds, call Cancel Appointment for the old reference.
   6. If CancelSuccess is true, give the new booking reference and confirm the old one is cancelled. If CancelSuccess is false, give the new reference, say the old booking could not be found, and suggest calling the clinic to check it.

   Cancelling: ask for the booking reference, confirm the caller wants to cancel, then call Cancel Appointment.

   Other rules:
   - Answer questions about the clinic only from the knowledge sources. If you don't know, say so and suggest calling back during opening hours.
   - Never give medical advice. If the caller describes an emergency, tell them to hang up and call their local emergency number straight away.
   ```

   ![Screenshot 96](images/image96.png)

3. Scroll down to **Knowledge** and select **+ Add knowledge**.

   ![Screenshot 97](images/image97.png)

4. Select **Upload file**.

   ![Screenshot 98](images/image98.png)

5. Browse to **C:\LabFiles**, select **Zava-Clinic-FAQ.docx** and select **Add to agent**.

   ![Screenshot 99](images/image99.png)

6. Wait until the file status shows **Ready**.

   ![Screenshot 100](images/image100.png)

You have added the agent instructions and the clinic FAQ. The agent now knows its role, the steps for each task and the rules it must follow for dates and times.

### Task 2: Build the Check Availability topic

In this task, you build the helper topic that the agent calls before it offers any appointment time. The topic takes a service and a date as inputs, cleans up the service name, calls the Check Availability flow and hands the free times back to the model as outputs.

1. Select the **Topics** tab, select the arrow next to **+ Add a topic**, and select **From blank**.

   ![Screenshot 101](images/image101.png)

2. Rename the topic **Check Availability**. Under **Trigger**, keep **The agent chooses**, and in **Describe what the topic does**, enter: Finds open appointment times for one service on one date. Always use this before offering any appointment times to the caller.

   ![Screenshot 102](images/image102.png)

3. Select **More (…)** at the top right, then select **Details**.

   ![Screenshot 103](images/image103.png)

4. Select the **Input** tab, then select **Create a new variable**.

   ![Screenshot 104](images/image104.png)

5. Create two inputs with the values below. For each one, keep **How will the agent fill this input?** set to **Dynamically fill with best option**.

   | Input name | Type | Description |
   | --- | --- | --- |
   | **Service** | String | The service the caller wants: General Consultation, Dental Check-up or Physiotherapy. |
   | **PreferredDate** | String | The date the caller asked for and confirmed, in YYYY-MM-DD format. Never a date chosen by the agent. |

   ![Screenshot 105](images/image105.png)

   ![Screenshot 106](images/image106.png)

6. Select the **Output** tab, then select **Create a new variable**.

   ![Screenshot 107](images/image107.png)

7. Create two outputs with the values below.

   | Output name | Type | Description |
   | --- | --- | --- |
   | **AvailableSlots** | String | The only free times for this service and date. Offer these exact times and no others. Empty means no free times. |
   | **SlotsFound** | Boolean | True if at least one time is free. |

   ![Screenshot 108](images/image108.png)

8. Close the **Topic details** pane. Below the trigger, select **+**, then select **Variable management > Set a variable value**.

   ![Screenshot 109](images/image109.png)

9. Under **Set variable**, select **>**, then select **Create a new variable**.

   ![Screenshot 110](images/image110.png)

10. In **Variable properties**, name the variable **ServiceName**. It shows as Topic.ServiceName.

    ![Screenshot 111](images/image111.png)

11. In **To value**, select **…**, open the **Formula** tab, enter the formula below and select **Insert**. It turns what the caller said, such as “dental check up”, into the exact value stored in SharePoint.

    ```text
    If(
        "dental" in Topic.Service, "Dental Check-up",
        "physio" in Topic.Service, "Physiotherapy",
        "consult" in Topic.Service || "general" in Topic.Service, "General Consultation",
        Topic.Service
    )
    ```

    ![Screenshot 112](images/image112.png)

    ![Screenshot 113](images/image113.png)

12. Select **+** below the node, select **Add a tool**, and choose **Zava - Check Availability**.

    ![Screenshot 114](images/image114.png)

13. Map the inputs: **Service** = ServiceName and **PreferredDate** = PreferredDate. Map the outputs: **availableslots** to **AvailableSlots** and **slotsfound** to **SlotsFound**.

    ![Screenshot 115](images/image115.png)

14. Don’t add a Message node, because the model speaks the result. Select **Save**.

    ![Screenshot 116](images/image116.png)

You have built the Check Availability topic. The agent can now look up free slots for any service and date before it offers times to a caller.

### Task 3: Build the Book Appointment topic

In this task, you build the helper topic that books the chosen slot. It reads the booking details back to the caller and asks for a spoken or keypad confirmation, then calls the Book Appointment flow and, when the booking succeeds, the Send Confirmation flow.

1. Create a blank topic and rename it **Book Appointment**. Keep the trigger **The agent chooses** and enter the description: Books one appointment after the caller has chosen a time and given their name and phone number. Then open **Details** and select the **Input** tab.

   ![Screenshot 117](images/image117.png)

2. Select **Create a new variable** and add five inputs with the values below, each set to **Dynamically fill with best option**.

   | Input name | Type | Description |
   | --- | --- | --- |
   | PatientName | String | The caller's full name, as they said it. |
   | PhoneNumber | String | The caller's phone number, digits only, with no spaces or symbols. |
   | Service | String | The service being booked: General Consultation, Dental Check-up or Physiotherapy. |
   | AppointmentDate | String | The appointment date the caller chose, in YYYY-MM-DD format. |
   | AppointmentTime | String | The appointment time the caller chose from the available slots, in HH:MM 24-hour format, for example 14:30. |

   ![Screenshot 118](images/image118.png)

   > Why String for all five? As a Number, a phone number loses a leading zero or a + sign. Fixed-format text dates and times match the plain-text SharePoint columns exactly.

3. Select the **Output** tab and add two outputs with the values below.

   | Output name | Type | Description |
   | --- | --- | --- |
   | BookingRef | String | Booking reference such as ZAV-4821. Empty if not booked. |
   | BookingSuccess | Boolean | True if the appointment was booked. |

   ![Screenshot 119](images/image119.png)

4. Close **Topic details**. Below the trigger, select **+ > Ask a question**. This is the deterministic confirmation step, so it is the same on every call. Set it up as follows:
   - **Message:** Just to confirm: {Service} on {AppointmentDate} at {AppointmentTime} for {PatientName}. Shall I book it? Say Yes or press 1. Say No or press 2. Insert each variable with the **{x}** button.
   - **Identify:** **Multiple choice options**, with the options **Yes** and **No**.

   ![Screenshot 120](images/image120.png)

   - Select **Assign DTMF keys to options** and check that **Yes** is mapped to key **1** and **No** to key **2**.
   - Under **Save user response as**, select **Var1**, rename it **Confirm** in **Variable properties**, and keep **Topic (limited scope)**. It shows as Topic.Confirm.

   ![Screenshot 121](images/image121.png)

5. Copilot Studio adds a condition for each option and an **All other conditions** branch. In the **Yes** branch, add the following:

   1. **Set a variable value** (**+ > Variable management > Set a variable value**). Under **Set variable**, select **Service** (Topic.Service). In **To value**, open the **Formula** tab, enter the formula below and select **Insert**. It replaces the caller’s wording with the exact service name stored in SharePoint.

      ```text
      If(
          "dental" in Topic.Service, "Dental Check-up",
          "physio" in Topic.Service, "Physiotherapy",
          "consult" in Topic.Service || "general" in Topic.Service, "General Consultation",
          Topic.Service
      )
      ```

      ![Screenshot 122](images/image122.png)

   2. **+ > Add a tool > Zava - Book Appointment**. Map the five inputs to the topic variables with the same names (PatientName, PhoneNumber, Service, AppointmentDate and AppointmentTime), and map the outputs **bookingref** to **BookingRef** and **bookingsuccess** to **BookingSuccess**.

      ![Screenshot 123](images/image123.png)

      ![Screenshot 124](images/image124.png)

   3. **+ > Add a condition**: **BookingSuccess** is equal to **true**.

      ![Screenshot 125](images/image125.png)

   4. Below that condition, select **+ > Add a tool > Zava - Send Confirmation**, map its five inputs from the topic variables, and save its **sent** output to a new variable named **Sent**. This is the post-booking automation.

      ![Screenshot 126](images/image126.png)

6. On the **No** condition under the question, select **…**, then select **Delete**. A No answer then falls through to **All other conditions**.

   ![Screenshot 127](images/image127.png)

7. Under **All other conditions**, add two **Set a variable value** nodes:
   - **BookingRef:** open the **Formula** tab and enter "" (two double quotes, which make empty text). Don’t leave the plain value box empty.
   - **BookingSuccess:** choose **false** in the value box.

   ![Screenshot 128](images/image128.png)

8. Select **Save**.

   ![Screenshot 129](images/image129.png)

You have built the Book Appointment topic. It confirms the details with the caller, books the slot through the flow and sends a confirmation email after every successful booking.

### Task 4: Build the Cancel Appointment topic

In this task, you build the helper topic that cancels a booking. It cleans up the booking reference the caller gives, calls the Cancel Appointment flow and returns whether the cancellation worked.

1. Create a blank topic and rename it **Cancel Appointment**. Keep the trigger **The agent chooses** and enter the description: Cancels an existing appointment using its booking reference. Then select **Save**.

   ![Screenshot 130](images/image130.png)

2. Open **Details > Input** and create one **String** input named **BookingRef**, with the description: The booking reference in the format ZAV-1234. Remove spaces and use capitals.

   ![Screenshot 131](images/image131.png)

3. On the **Output** tab, create one **Boolean** output named **CancelSuccess**, with the description: True if the appointment was cancelled.

   ![Screenshot 132](images/image132.png)

4. Below the trigger, select **+ > Add a tool > Zava - Cancel Appointment**.

   ![Screenshot 133](images/image133.png)

5. For the **BookingRef** input, select **…**, open the **Formula** tab, enter the formula below and select **Insert**. It removes spaces, uses capitals and rebuilds the reference if the caller left out the hyphen, so “zav 4821” becomes “ZAV-4821”.

   ```text
   If(
       "-" in Topic.BookingRef,
       Upper(Substitute(Topic.BookingRef, " ", "")),
       "ZAV-" & Right(Substitute(Topic.BookingRef, " ", ""), 4)
   )
   ```

   ![Screenshot 134](images/image134.png)

6. Map the **cancelsuccess** output to **CancelSuccess**, then select **Save**.

   ![Screenshot 135](images/image135.png)

You have built the Cancel Appointment topic. The agent can now cancel a booking from the reference a caller gives, even if they leave out the hyphen.

In this exercise, you wrote the agent instructions, added the clinic FAQ and built the three helper topics. Checkpoint: each helper topic has typed inputs and outputs and calls its flow, none has a Message node, and Book Appointment has a Yes/No confirmation with DTMF keys 1 and 2.

## Exercise 5: Test by voice

In this exercise, you talk to the agent through the Copilot Studio test panel and run a set of test scenarios. You watch the SharePoint list and your mailbox change during each call to confirm that the conversation and the flows work together.

### Task 1: Start a voice test

In this task, you open the test panel, switch it to speech and keypad mode, and start your first voice conversation with the agent.

1. Put on your headset, and open the ZavaAppointments list in another browser tab.

2. In Copilot Studio, select **Test** at the top right to open the test panel.

   ![Screenshot 136](images/image136.png)

3. Select the chat mode icon and make sure **Speech & DTMF (Preview)** is selected.

   ![Screenshot 137](images/image137.png)

4. Select the green call button to **Start voice conversation**.

   ![Screenshot 138](images/image138.png)

5. When the browser asks for microphone access, select **Allow while visiting the site**.

   ![Screenshot 139](images/image139.png)

6. Wait for the greeting to finish. The label shows **Listening…** when it’s your turn and **Talk to interrupt** while the agent speaks. Use the keypad button to press keys, and the red button to end the call.

   ![Screenshot 140](images/image140.png)

7. End the call after each scenario, and start a new voice conversation for the next one so that each scenario starts fresh.

You have started a voice conversation with the agent and heard the greeting.

### Task 2: Run the test scripts

In this task, you run seven scenarios that cover booking, no free slots, declining, rescheduling, cancelling, an FAQ question and barge-in. After each call, you check the SharePoint list and your mailbox to confirm what the agent did.

1. Run each scenario in the table below. Use the dates in your SharePoint list and say them naturally, for example “Tuesday the 10th”.

   | # | Scenario | What you say or press | Expected result |
   | --- | --- | --- | --- |
   | 1 | Book an appointment | “I'd like a dental check-up on Tuesday the 10th.” Pick a time, then give your name and phone number. Press **1** to confirm. | Offers 10:00, 11:30 and 15:00, then reads back the details and gives a ZAV-#### reference. The row changes to Booked, and a confirmation email arrives. |
   | 2 | No slots | “Can I see a physiotherapist on Sunday?” | Says nothing is free that day and asks which other day you’d like. Doesn’t invent times or suggest a date. |
   | 3 | Decline at confirmation | Book as in scenario 1, but press **2** at the confirmation. | Doesn't book. Asks what you'd like to change. The list doesn't change. |
   | 4 | Reschedule | “I need to move booking” plus your reference from scenario 1, “to Thursday.” | Asks for your service, name and phone number again, books a Thursday slot, then the old row changes to Cancelled. |
   | 5 | Cancel | “Please cancel my appointment.” Give the reference when asked. | Confirms first, then the row changes to Cancelled. |
   | 6 | FAQ | “Is there parking?” | Answers from the FAQ: free parking for 40 cars behind the building. |
   | 7 | Barge-in | Start talking while the greeting is playing. | The agent stops and listens. |

2. After scenario 1, open Outlook and check the confirmation email. It shows the booking reference, patient, service, date and time.

   ![Screenshot 141](images/image141.png)

3. Refresh the ZavaAppointments list. Booked and cancelled slots show their status, booking reference, name and phone number.

   ![Screenshot 142](images/image142.png)

You have tested the agent end to end and confirmed that each call updates SharePoint and, for bookings, sends a confirmation email.

In this exercise, you tested the agent by voice and keypad, and confirmed that every booking, reschedule and cancellation updated the SharePoint list and that bookings sent a confirmation email.

## Summary

In this lab, you built Zava Clinic Scheduler, a real-time voice receptionist in Copilot Studio that books, reschedules and cancels live clinic appointments. You:

- Created a SharePoint site and the ZavaAppointments list as the clinic’s booking system, with dates and times stored in an exact text format.
- Built four Power Automate agent flows (Check Availability, Book Appointment, Cancel Appointment and Send Confirmation) that read and update the list and email the front desk.
- Created a real-time voice agent and configured its voice, speech sensitivity, DTMF keypad input, silence detection, latency message and greeting.
- Wrote instructions that combine natural conversation with strict rules for dates, times and rescheduling, and added an FAQ as knowledge.
- Built three helper topics with typed inputs and outputs that call the flows, including a deterministic spoken and keypad confirmation before booking.
- Tested the agent by voice and keypad, and confirmed each booking change in SharePoint and in your mailbox.

The key design idea carries over to any voice agent you build: let the model handle the conversation, and route everything that must be exact, such as data access, confirmations and safety rules, through deterministic topics and flows with clearly described inputs and outputs.
