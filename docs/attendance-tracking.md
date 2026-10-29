# Student Attendance Tracking with Wazuh and Active Directory

This lab extension uses Active Directory identities and Windows Security events to estimate when students start and end a lab session. It is suitable for a controlled classroom or training lab where each student has an individual domain account.

> Attendance data is an estimate based on workstation activity. A successful sign-in does not prove that a student remained present, and missing telemetry does not prove absence. Use this dashboard as an operational aid rather than the sole attendance record.

## What the dashboard measures

| Status or metric | Source |
| --- | --- |
| Present | At least one interactive sign-in during the class window |
| Late | First interactive sign-in occurred after the configured start time |
| First sign-in | Earliest event `4624` for the student |
| Last sign-out | Latest event `4634` or `4647` for the student |
| Estimated session time | Last sign-out minus first sign-in |
| Absent | A rostered student has no qualifying sign-in during the class window |

The relevant Windows Security event IDs are:

- `4624`: successful account sign-in
- `4634`: account session ended
- `4647`: user initiated sign-out

Filter event `4624` to interactive logon types:

- `2`: local interactive sign-in
- `10`: Remote Desktop sign-in

This avoids counting service, network share, scheduled task, and machine account activity as attendance.

## 1. Prepare student accounts

Create one standard Active Directory account per student and place the accounts in a dedicated organizational unit or security group, such as `Lab Students`. Do not infer attendance from shared accounts.

Use a consistent identifier in `sAMAccountName`, for example `student.001`. Keep names and other personal data outside the SIEM unless they are required by the class.

## 2. Enable Windows sign-in auditing

Apply a Group Policy Object to the student workstations:

1. Open **Computer Configuration > Policies > Windows Settings > Security Settings > Advanced Audit Policy Configuration > Audit Policies > Logon/Logoff**.
2. Enable **Audit Logon** for **Success**.
3. Enable **Audit Logoff** for **Success**.
4. Run `gpupdate /force` on a test workstation.

Verify the events in **Event Viewer > Windows Logs > Security** by signing in and out with a test student account.

## 3. Collect the events with the Wazuh agent

Add this block inside `<ossec_config>` in the Wazuh agent configuration on each monitored Windows workstation:

```xml
<localfile>
  <location>Security</location>
  <log_format>eventchannel</log_format>
  <query>Event/System[(EventID=4624 or EventID=4634 or EventID=4647)]</query>
</localfile>
```

Restart the agent from an elevated PowerShell prompt:

```powershell
Restart-Service -Name wazuh
```

Confirm that events arrive before creating dashboard panels. The expected Wazuh fields include:

- `data.win.system.eventID`
- `data.win.eventdata.targetUserName`
- `data.win.eventdata.logonType`
- `data.win.eventdata.workstationName`
- `data.win.eventdata.ipAddress`
- `agent.name`
- `timestamp`

Field availability can vary by Wazuh and Windows versions. Inspect a test event in **Wazuh Dashboard > Discover** and adjust saved searches to match the fields in the deployed version.

## 4. Create the attendance searches

Use `wazuh-alerts-*` as the data view. If successful sign-ins are not retained as alerts in the installed ruleset, validate collection with the event archives and then create an appropriate low-severity local rule. Avoid enabling full event archives permanently without reviewing storage and retention requirements.

### Successful interactive sign-ins

```text
data.win.system.eventID:"4624" and
(data.win.eventdata.logonType:"2" or data.win.eventdata.logonType:"10") and
not data.win.eventdata.targetUserName:*$ and
not data.win.eventdata.targetUserName:"ANONYMOUS LOGON"
```

### Sign-outs

```text
(data.win.system.eventID:"4634" or data.win.system.eventID:"4647") and
not data.win.eventdata.targetUserName:*$
```

### One student

Append the following filter to either search:

```text
and data.win.eventdata.targetUserName:"student.001"
```

Save the searches as `Attendance - Interactive Sign-ins` and `Attendance - Sign-outs`.

## 5. Build the Wazuh dashboard

Create a dashboard named `Student Attendance` with the following panels.

### Present students

- Visualization: data table
- Filter: saved interactive sign-ins search
- Time range: scheduled class window
- Rows: `data.win.eventdata.targetUserName.keyword`
- Metric: unique count of `data.win.eventdata.targetUserName.keyword`

### First sign-in

- Visualization: data table
- Filter: saved interactive sign-ins search
- Rows: `data.win.eventdata.targetUserName.keyword`
- Metric: minimum `timestamp`

Sort by the minimum timestamp. Students whose first sign-in is later than the scheduled start time are late.

### Last sign-out

- Visualization: data table
- Filter: saved sign-outs search
- Rows: `data.win.eventdata.targetUserName.keyword`
- Metric: maximum `timestamp`

### Activity timeline

- Visualization: line chart
- Filter: both saved searches
- Horizontal axis: `timestamp`
- Split series: `data.win.eventdata.targetUserName.keyword`
- Break down by: `data.win.system.eventID`

### Workstation usage

- Visualization: data table
- Filter: saved interactive sign-ins search
- Rows: `data.win.eventdata.targetUserName.keyword`
- Split rows: `agent.name` or `data.win.eventdata.workstationName.keyword`
- Metric: event count

## 6. Add late, absent, and session-time calculations

Raw event panels can show who signed in and out, but they cannot reliably identify students who generated no events. For complete attendance:

1. Maintain a roster containing the expected domain usernames.
2. Define the class date, scheduled start, scheduled end, and late threshold.
3. Aggregate each student's earliest qualifying `4624` and latest `4634` or `4647`.
4. Compare the aggregation with the roster.
5. Write one derived record per student and class to a dedicated attendance index.

Example derived record:

```json
{
  "class_id": "cyber-lab-101",
  "class_date": "2026-10-28",
  "student_id": "student.001",
  "status": "present",
  "first_sign_in": "2026-10-28T09:02:11Z",
  "last_sign_out": "2026-10-28T10:27:43Z",
  "estimated_minutes": 85,
  "workstation": "LAB-PC-04"
}
```

Apply these rules when producing the derived record:

- `absent`: rostered username has no qualifying `4624`
- `late`: first qualifying `4624` is after the late threshold
- `present`: first qualifying `4624` is on or before the late threshold
- `incomplete`: a sign-in exists but no sign-out exists
- `estimated_minutes`: only calculate when both timestamps exist and are in a reasonable order

Do not treat screen lock and unlock events as sign-out and sign-in. They can be added later as supporting activity signals using events `4800` and `4801`.

## 7. Validate before classroom use

Test these cases with dedicated lab accounts:

1. Normal local sign-in and sign-out.
2. Remote Desktop sign-in and sign-out.
3. A student who never signs in.
4. A late sign-in.
5. A machine restart without a clean sign-out.
6. A service or computer account event that must be excluded.
7. The same student signing into more than one workstation.

Compare the dashboard with Event Viewer on the workstation and confirm that timestamps use the intended time zone.

## Privacy and retention

- Notify students what is collected and why.
- Limit access to instructors and authorized lab administrators.
- Prefer student IDs over names in the SIEM.
- Set a short retention period appropriate for the course.
- Do not use the dashboard as the sole basis for disciplinary or grading decisions.
- Follow the institution's privacy, records, and acceptable-use policies.
