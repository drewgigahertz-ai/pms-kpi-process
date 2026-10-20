# PMS KPI Process (prototype)

A clickable prototype of the flow:

**Build form → Schedule Set Up → Accuracy Set Up → Collect → Responses → Pivot table → Visualize (chart or table)**

It is one static file (`index.html`), with no build step and no backend. Data is saved in the browser's `localStorage`. **Reset to sample data** restores the examples: 10 people (8 staff and 2 reviewers), 3 forms, 6 schedules from Sep 30 to Oct 23, 2026, and about 60 submissions.

## Deploy to Vercel

- **Drag and drop:** vercel.com → Add New → Project → deploy this folder.
- **CLI:** `npm i -g vercel`, then `cd pms-kpi-process` and `vercel --prod`.
- **Git:** push this folder to a repo and import it in Vercel. Set the framework preset to **Other**. There is no build command, and the output directory is the root.

## The flow

| Step | Who | What happens |
|---|---|---|
| 1. Build form | Manager | Builds forms: questions only, no dates |
| 2. Schedule Set Up | Manager | Creates a task: End time stamp, Name of task, Description, Assigned to (many), Reviewer 1, Name of form (many) |
| 2b. Accuracy Set Up | Manager | For forms measured by accuracy: Ratee, KPI, Rater, Name of form |
| 3. Collect | Employee / Rater | Sees the tasks assigned to them and submits each form in the task |
| 4. Responses | Manager / Head | Every submission to one form, with its task, assignee and End time stamp |
| 5. Pivot | Head | **Picks one form**, adds one or more fields to Rows and Columns, filters their values, and picks Values and date grouping |
| 6. Visualize | Head | Picks a bar, stacked bar, horizontal bar, line, pie or doughnut chart, or a table |

## Schedule Set Up

The schedule sets the **deadline for submissions**. It does not decide what goes into a pivot table: each pivot covers one form.

| Field | Type | Notes |
|---|---|---|
| End time stamp | date and time | Deadline. Submissions after it are **accepted and marked Late** |
| Name of task | text | Required |
| Description | text | Optional |
| Assigned to | many people | Pick by person or a whole department at once |
| Reviewer 1 | one person | |
| Name of form | many forms | Every assignee must submit **every** selected form once |

Required submissions for a schedule = assignees × forms. For example, 8 people × 2 forms = 16.

The schedule list shows each task's progress ("12 of 16 submitted") and status:
- **Open:** before the end time stamp.
- **Past due:** after the end time stamp, with forms still missing.
- **Complete:** all required forms submitted.

The **People list** under Schedule Set Up is where assignees and reviewers come from. In production, it comes from the employee directory.

## Accuracy (for forms measured by accuracy)

A form is **for accuracy** when it is picked under *Name of form* in any Accuracy Set Up. Only those forms follow this workflow.

**Accuracy Set Up** (one row per set up):

| Field | Type | Notes |
|---|---|---|
| Ratee | many people | Whose submissions are rated |
| KPI | text | e.g. "Report accuracy". Earlier KPI names are suggested |
| Rater | one person | Separate from Reviewer 1. Cannot be one of their own ratees |
| Name of form | many forms | Any existing form. Picking it here turns on the accuracy workflow for it |

**Workflow:**
1. The ratee submits the form in Collect, against a schedule as usual.
2. The submission appears in the rater's **To rate** list in Collect, as Pending.
3. The rater opens it, reads the answers and enters an **accuracy score of 0–100%**, with an optional comment.
4. The rating is saved on the submission as `ratings[setupId] = { score, comment, by, at }`. The ratee sees "Accuracy 92%" on their task.

If a ratee resubmits for the same task, only the latest submission is rated and counted.

**In the pivot** (only when the selected form is in an Accuracy Set Up):
- Value **Accuracy % = average score** of rated submissions. Pending ones are not counted. Cells show "n rated · m pending".
- Fields **KPI**, **Rater** and **Rating status** (Rated, Pending, No rater set). A submission with two KPIs, e.g. Report accuracy and Target computation accuracy, appears under both.

## Recorded on every submission

| Field | Source |
|---|---|
| Task, Assignee, Department, Reviewer 1, Form, End time stamp | From the schedule and the person who submitted |
| `sys_submitted` | Submission time. Use **server time** in production. The prototype's "Submission time (demo)" box is only for trying late cases |
| `sys_timeliness` | **Late** if `submitted_at > schedule.end`, otherwise **On time** |
| `sys_hours_late` | Hours past the end time stamp, or 0 |

## Pivot (one form at a time)

The head picks **one form**. The pivot uses every response to that form, from all the schedules that include it. Forms are never mixed in one pivot table.

**Rows and Columns** can each hold several fields. They nest in the order listed, for example Rows = Department › Assignee and Columns = Task › Timeliness. Reorder them with ‹ ›, remove them with ×, and add more with **+ Add field**.

**Filters:** every row and column field has a filter button, both on its chip and in the pivot table header. It opens a checklist of that field's values with counts, Select all and Clear, and a search box for long lists. Unticked values are hidden from the table, its totals and the chart. A filter is stored as the list of hidden values, so new values show up automatically.

Available fields:
- **Schedule:** Task, Assignee, Department, Reviewer 1, End time stamp (taken from the schedule each response was submitted under)
- **Form questions:** the form's own questions
- **Recorded automatically:** Submitted at, Timeliness, Hours late

## Completeness (calculated in the pivot)

```
for the selected form:
required  = every (schedule that includes the form, assignee of that schedule)
submitted = those with at least one response (late ones count; duplicates count once)
Completeness % = submitted ÷ required × 100
```

Examples from the sample data (Daily KPI Report):
- It appears in 5 schedules. Staff in Finance and HR are assigned 4 of them, and Sales and Operations staff are assigned all 5.
- Ben Reyes: required 4, submitted 2, so **50%**.
- People who submitted nothing for a task still appear as "0 of 1", because required counts come from the schedule, not from responses.

Grouping by a schedule field splits both submitted and required. Grouping by a question answer or by Timeliness splits only the submitted side.

**On-time rate %** = submissions made by the task's End time stamp ÷ all submissions.

## Question types

| Group | Types | Stored as | In the pivot |
|---|---|---|---|
| Text | Short answer, Paragraph, Email, Phone number, Link | text | Groupable (Paragraph is not). Links group by website, e.g. drive.google.com |
| Choice | Multiple choice, Checkboxes, Dropdown | text, or a list for Checkboxes | Groupable. A Checkboxes answer counts once under each ticked choice |
| Scale and number | Linear scale, Rating, Number | number | Values (sum, average, min, max). Scale and Rating can also be grouped |
| Grid | Multiple choice grid, Checkbox grid | `{ "row": "column" }` or `{ "row": ["columns"] }` | Each grid row becomes its own pivot field |
| Date and time | Date, Time, Date and time | ISO text | Dates group by day, week or month. Times group by hour |
| Upload | File upload (allowed types, max files, max size) | `[{ "name", "size", "type" }]` | Groups as "File uploaded" or "No file" |

Validation: required answers, number format, email format, phone format (digits, spaces, dashes, brackets, leading +), links must be http/https (https:// is added if missing), file count, size and type, and every row answered for required grids.

The prototype keeps only file **names and sizes**. In production, upload files to storage (for example Vercel Blob or S3) and save the file URL with the response.

**Choices editor:** one box per choice. Press Enter to add the next one, or Backspace in an empty box to remove it. Paste a multi-line list into any box to add every line as a choice. **Copy all** copies the choices one per line, and **Edit as list** opens them as plain text for bulk editing.

## Data shapes

```json
{ "id": "p_ben", "name": "Ben Reyes", "dept": "Finance" }

{ "id": "form_update", "title": "Weekly Team Update",
  "fields": [{ "id": "f_status", "label": "Overall status", "type": "radio", "options": ["On track","At risk","Delayed"], "required": true }] }

{ "id": "s_u1016", "end": "2026-10-16T17:00", "name": "Weekly update – Oct 16",
  "description": "Weekly KPI and team update.",
  "assignees": ["p_ana","p_ben","..."], "reviewer": "p_rosa", "formIds": ["form_update","form_kpi"] }

{ "id": "r_x1y2z3", "scheduleId": "s_u1016", "formId": "form_update", "by": "p_ben",
  "submittedAt": "2026-10-16T18:40",
  "values": { "f_status": "On track", "sys_submitted": "2026-10-16T18:40", "sys_timeliness": "Late", "sys_hours_late": 1.7 } }

{ "id": "a_report", "ratees": ["p_ana","p_ben","..."], "kpi": "Report accuracy", "rater": "p_lea", "formIds": ["form_kpi"] }

"ratings": { "a_report": { "score": 92, "comment": "Target figure did not match the source report.", "by": "p_lea", "at": "2026-10-17T10:15" } }

{ "formId": "form_kpi", "rows": ["t_dept","t_assignee"], "columns": ["t_task"],
  "filters": { "t_dept": ["Finance"] },
  "value": "m_complete", "dateGrouping": "week", "chartType": "table" }
```

The live versions of these objects are shown in the app under **Visualize → For developers**.

## Suggested production build

- **Tables:**
  - `employees` (id, name, department, active)
  - `forms` (id, title, description, schema JSONB, created_by)
  - `schedules` (id, name, description, end_at timestamptz, reviewer_id, created_by)
  - `schedule_assignees` (schedule_id, employee_id)
  - `schedule_forms` (schedule_id, form_id)
  - `responses` (id, schedule_id, form_id, employee_id, submitted_at timestamptz, is_late, hours_late, values JSONB)
  - `accuracy_setups` (id, kpi, rater_id), `accuracy_setup_ratees` (setup_id, employee_id), `accuracy_setup_forms` (setup_id, form_id)
  - `ratings` (id, response_id, setup_id, rater_id, score numeric CHECK (score BETWEEN 0 AND 100), comment, rated_at), unique (response_id, setup_id)
  - `views` (id, owner_id, form_id, pivot_config JSONB, chart_type)
- **Completeness of one form by assignee and task:**

  ```sql
  WITH required AS (
    SELECT s.id AS schedule_id, s.name AS task, a.employee_id
    FROM schedules s
    JOIN schedule_forms     f ON f.schedule_id = s.id AND f.form_id = $1
    JOIN schedule_assignees a ON a.schedule_id = s.id
  ), submitted AS (
    SELECT DISTINCT schedule_id, employee_id FROM responses WHERE form_id = $1
  )
  SELECT e.name, r.task,
         COUNT(*)                AS required,
         COUNT(sb.employee_id)   AS submitted,
         ROUND(100.0 * COUNT(sb.employee_id) / COUNT(*)) AS completeness_pct
  FROM required r
  JOIN employees e ON e.id = r.employee_id
  LEFT JOIN submitted sb USING (schedule_id, employee_id)
  GROUP BY ROLLUP (e.name, r.task);
  ```
