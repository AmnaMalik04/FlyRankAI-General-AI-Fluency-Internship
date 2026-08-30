# AI Tutoring Session & Parent Communication Assistant

## Overview
The agent is an n8n based agent designed to act as a personal assistant to a tutor with the following administrative tasks:
1. Manage parent requests to reschedule tutoring sessions.
2. Monitor scheduled tutoring sessions for student attendance.

The agent uses Gmail, Google Calendar and ZOOM API calls and credentials to handle these tasks. 

The goal is to send low risk communications and reduce repetitive tasks for the tutor while requiring the tutor's approval where necessary. 

## Intended Audience
The agent is designed for a tutor who:
- Manages recurring tutoring sessions
- Communicates with parents through Gmail
- Manages sessions through Google Calendar
- Conducts lessons via Zoom
- Needs to handle rescheduling and student absences without repeatedly checking multiple resources

## Working
The agent is divided into two n8n workflows because scheduling requests and sessions monitoring have different triggers and actions.

                      Tutor
                        │
             ┌──────────┴──────────┐
             │                     │
      Scheduling Workflow    Monitoring Workflow
             │                     │
          Gmail              Google Calendar
             │                     │
          AI Parser             Data Table
             │                     │
      Google Calendar          Zoom API
             │                     │
      Tutor Approval        Attendance Checks
             │                     │
          Gmail              Gmail + Approval


### Workflow 1 - AI Tutoring Parent Scheduler Assistant 
This workflow handles parent requests to change an existing tutoring session.
Trigger - Parent's Email
Actions: Gmail detects tutoring email, AI interprets the request, extract session details, retrieve the relevant information, convert date/time to match the tutor's timezone, check Google Calendar, check requested slot's availability, draft an email, get tutor's approval, act accordingly. 

### Workflow 2 - Tutoring Session Monitoring Agent
This workflow handles student attendance by asking the tutor for confirmation every 5 minutes till the 15 minute mark after the scheduled session starts.
Trigger -  meeting is about to start
Actions: Get events from Google Calendar, filter the events to ensure it is a tutoring event, retrieve student's details, retrieve recurring ZOOM meeting after successful authentication, confirm student attendance every 5 minutes. Only stop monitoring if student is marked present by the tutor. If absent, continue till 15 minutes. Send an email every 5 minutes if no parent response is received. After 15 minutes, prepare the final no-show action. The tutor must approve the final email before it is sent, or choose to handle the situation manually.

### Key Design Decisions

#### Human Approval
The agent is designed to automate repetitive and low risk tasks without taking important decisions away from the tutor. 

For the scheduling workflow, the agent can interpret the parent's request, check the calendar and draft the response, but the tutor has to approve the schedule change before the final action is taken.

For session monitoring, the 5 and 10 minute emails are simple check-in emails and can be sent automatically. However, the final 15 minute email mentions the tutoring policy and the possibility of the session being charged. Therefore, it requires the tutor's approval before it is sent.

#### Two Separate Workflows
The agent is divided into two workflows because they have different triggers and responsibilities. The scheduling assistant starts when a relevant parent email is received, while the session monitoring agent starts based on scheduled tutoring sessions.

Keeping them separate also makes the workflows easier to understand, test and troubleshoot.

#### Stop Monitoring When the Situation Changes
The session monitoring workflow should not continue sending emails when they are no longer necessary.

Monitoring stops if:
- The tutor confirms that the student has joined.
- A response from the parent is detected.
- At the 15 minute mark, the tutor chooses to handle the situation manually.

If a parent response is detected, the agent asks the tutor to check the email instead of trying to interpret the situation and continuing automatically.


## Tools Used

| Tool | Use |
|---|---|
| n8n | Workflow orchestration |
| Gmail | Receiving parent emails and sending communications |
| Google Calendar | Retrieving tutoring sessions and checking availability |
| n8n Data Tables | Storing student name, parent email and parent timezone |
| AI Model | Interpreting scheduling requests and extracting structured information |
| Zoom API | Authenticating and retrieving the recurring Zoom meeting |
| n8n Forms | Tutor approval and manual attendance confirmation |


## Setup

## Workflow Files

The agent consists of two exported n8n workflows:

- `AI-Tutoring-Parent-Scheduler-Assistant.json`
- `Tutoring-Session-Monitoring-Agent.json`

To import a workflow into n8n:

1. Open n8n.
2. Create a new workflow.
3. Select **Import from File**.
4. Import the relevant JSON file.
5. Reconnect your own Gmail, Google Calendar, AI model and Zoom credentials.
6. Create the required Parent Data Table.
7. Replace any test email addresses, Calendar IDs and meeting IDs with your own information.

Credentials and API secrets are not included in the exported files.

### Requirements
To reproduce the agent, the following are required:
- n8n
- Google account with Gmail and Google Calendar
- Gmail OAuth credentials
- Google Calendar OAuth credentials
- An AI model supported by n8n
- Zoom account
- Zoom Server-to-Server OAuth application
- Student and parent information
- Scheduled tutoring events in Google Calendar


### 1. Set Up Gmail
Connect a Gmail account to n8n using Google OAuth credentials.

The Gmail connection is used by both workflows. The scheduling workflow uses it to retrieve relevant parent emails and send the approved response. The monitoring workflow uses it to check whether the parent has responded during the session and to send the 5, 10 and approved 15 minute emails.

The scheduling workflow should only process emails relevant to tutoring.


### 2. Set Up Google Calendar
Connect the tutor's Google Calendar account to n8n.

Google Calendar is used to check requested session availability in the scheduling workflow and to retrieve scheduled tutoring sessions in the monitoring workflow.

For the monitoring workflow, tutoring events follow the naming structure:

`Tutoring - Student Name`

For example:

`Tutoring - Sarah`

The workflow filters Calendar events for the word "Tutoring" and uses the student's name to retrieve their information from the Data Table.


### 3. Set Up the Parent Data Table
Create an n8n Data Table to store the information required by the workflows.

The table should include:

- `student_name`
- `parent_email`
- `parent_timezone`

For the monitoring workflow, the student's recurring Zoom Meeting ID should also be available to the workflow.

The student name in the table should match the student name used in the Google Calendar event.


### 4. Set Up the AI Model
Connect an AI model supported by n8n.

In the scheduling workflow, the model is responsible for analysing the parent's email and extracting information such as:

- Whether the email is a schedule change request
- Student name
- Subject
- Original session date and time
- Requested date and time
- Relevant scheduling information

The model is instructed not to invent information when the parent's request is unclear.


### 5. Set Up Zoom
Create a Server-to-Server OAuth application through the Zoom App Marketplace.

The application provides:
- Account ID
- Client ID
- Client Secret

These credentials should be kept private.

The workflow first requests an access token from:

`POST https://zoom.us/oauth/token`

The request uses:
- Client ID as the Basic Authentication username
- Client Secret as the Basic Authentication password
- `grant_type = account_credentials`
- The Zoom Account ID

The returned access token is then used to authenticate the Zoom API request and retrieve the recurring tutoring meeting.

The required Zoom scopes should be added to the Server-to-Server OAuth application before activating it.


### 6. Set Up Session Monitoring
The Schedule Trigger runs periodically and retrieves upcoming Google Calendar events.

The workflow then:
1. Filters for tutoring events.
2. Retrieves the correct student and parent information.
3. Authenticates with Zoom.
4. Retrieves the recurring Zoom meeting.
5. Asks the tutor to confirm whether the student has joined.
6. Starts the monitoring process if the student is absent.

The workflow checks again at approximately 5, 10 and 15 minutes.

For testing, these Wait nodes are temporarily changed from minutes to seconds so the entire workflow can be tested quickly. They should be changed back to the intended intervals before actual use.


## Usage Examples

### Example 1 - Schedule Change
A parent sends an email asking to move their child's tutoring session from the original time to another date or time.

The agent identifies the request, extracts the relevant information and converts the requested time where necessary. It then checks the tutor's Google Calendar for availability.

The requested change and availability are shown to the tutor for approval before the workflow continues with the parent response.


### Example 2 - Student Is Absent
A scheduled tutoring session begins and the tutor confirms that the student has not joined.

After approximately 5 minutes, the agent checks whether the parent has responded. If there is no response and the student is still absent, a polite check-in email is sent to the parent.

The process is repeated at approximately 10 minutes.

At 15 minutes, if there is still no parent response and the student has not joined, the agent prepares the final no-show action. The tutor can either:

- Approve and send the no-show email
- Handle the situation manually

The agent does not send the final email without tutor approval.


### Example 3 - Parent Responds
If the parent responds while the student is being monitored, the agent stops the automated monitoring process and informs the tutor that a parent response has been detected.

The tutor is asked to check the email and decide how to proceed rather than allowing the agent to assume what the parent's response means.


## Guardrails

The agent includes the following guardrails:

- Schedule changes require tutor approval.
- Ambiguous scheduling information should not be guessed.
- The agent does not assume that a student has joined the meeting.
- Session monitoring stops once the tutor confirms the student is present.
- A parent response stops the automatic absence escalation.
- The 15 minute no-show email requires tutor approval.
- The agent does not independently decide whether a session should be charged.
- The agent does not change the tutor's policies.
- The agent does not end the Zoom meeting automatically.


## V2 Evaluation Results

| Test Case | Scenario | Expected Result | Result |
|---|---|---|---|
| TC-01 | Student attends | Monitoring stops when attendance is confirmed | Pass |
| TC-02 | Student is absent at 5 minutes | Parent response is checked and check-in process begins | Pass |
| TC-03 | Parent responds during monitoring | Monitoring stops and tutor is asked to check the email | Pass |
| TC-04 | Student joins during monitoring | Further monitoring and emails stop | Pass |
| TC-05 | Student remains absent for 15 minutes | Final no-show email requires tutor approval | Pass |
| TC-06 | Tutor chooses Handle Manually | Final email is not sent and automated monitoring stops | Pass |
| TC-07 | Automatic Zoom attendance detection | Automatically retrieve live participant attendance | Could not be implemented due to Zoom API plan restriction |


## Limitations

### ZOOM Live Participant Detection
The original plan was for the agent to automatically check whether the student had joined the Zoom meeting.

The Zoom Server-to-Server OAuth authentication and meeting retrieval were successfully implemented. However, when testing the live participant API, Zoom returned:

> "This API is only available for ZMP and Business or higher accounts that have enabled the Dashboard feature."

Because the current Zoom subscription does not provide access to this endpoint, the agent cannot automatically determine whether a student has joined.

As a fallback, the agent asks the tutor to confirm attendance manually. The rest of the monitoring workflow remains automated.


### Dependence on Consistent Data
The workflows rely on the information in the Data Table and Google Calendar being consistent.

For example, an event named:

`Tutoring - Sarah`

needs to correspond to the student named `Sarah` in the Data Table.

Incorrect names, missing parent information or outdated timezone information can prevent the workflow from retrieving the correct data.


### Ambiguous Scheduling Requests
The scheduling workflow depends on the parent providing enough information to understand the requested change.

For example:

`Can we move the session a little later next week?`

does not provide enough information to safely determine the intended session. The agent is instructed not to invent missing information, so requests like this may need to be handled manually.


### Calendar Availability
The scheduling workflow checks whether the requested slot is free in Google Calendar. A free Calendar slot, however, does not always mean that the tutor wants to teach at that time.

A future version could include the tutor's preferred working hours and availability rules rather than relying only on existing Calendar events.


## Future Improvements

Future versions of the agent could include:

- Automatic Zoom participant detection with access to the required Zoom plan/API.
- Tutor availability rules in addition to Google Calendar conflict checking.
- A dedicated notification interface instead of relying on n8n forms.
- More flexible identification of tutoring sessions instead of depending on a naming convention.
- Improved handling of parent responses during session monitoring.

## AI Transparency

AI was used both as part of the agent and during its development.

Within the agent, an AI model is used in the scheduling workflow to interpret parent emails and extract structured scheduling information from natural language.

I also used ChatGPT as a development assistant while building the project, including helping me plan the workflows, troubleshoot implementation issues and refine the documentation. I built and configured the workflows in n8n myself and tested their behaviour and guardrails using different scenarios before finalising the agent.

## Conclusion

The agent was developed to reduce repetitive administrative work involved in private tutoring without removing the tutor from important decisions.

The final system uses two workflows to handle scheduling requests and session monitoring separately. Automation is used for repetitive and low risk actions, while schedule changes and the final no-show communication remain under the tutor's control.
