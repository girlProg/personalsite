---
title: "AI Assisted Workflows"
date: 2026-06-01
categories: [ai, claude code, dashboards, kobotoolbox, agile, world bank]
---

# Workflow Between 2023-2024 Using Matplotlib

KoboToolbox provides a fairly informative dashboard to visualise the data collected by your field workers for any form. My primary use case is for student enrolment for the [AGILE (Adolescent Girls Initiative for Learning and Empowerment)](https://documents.worldbank.org/en/publication/documents-reports/documentdetail/099081425093029294) World Bank Programme.

We collect student data across 2 states independent of each other. During the last verification exercise of the enrolled students in one of the states, I found myself wondering about the statistics of the live data we were collecting. Although yet to be cleaned, it would nevertheless still be useful to know the estimated quality of the data and know ahead of time if we were meeting our goals. What makes this specifically ideal for a verification exercise is because you know ahead of time how many students, how many schools, etc you want to verify. So for example, we are verifying if 40,000 students across one state are in the schools they were at the time of enrolment, and we know that all those 40,000 students' data was collected across 350 schools spread around the state.

In the early days of the project, I used to use Python scripts to check data collection status. I would connect to the KoboToolbox API and draw out graphs, output status documents and summaries for the field enumerators to answer to.

```python
fig, ax = plt.subplots()
bar_container = ax.bar(percentages.keys(), percentages.values())
ax.set_ylabel('Percentage')
ax.set_title(f'Cummulative Submission Progress as at {datetime.now().strftime("%H:%M on %d/%m/%Y")}')
ax.set_ylim(0,140)
plt.xticks(rotation = 25)
ax.bar_label(bar_container)
plt.show()
```

And this would give me a graph with percentage progress for each local government.

![state graph](/assets/images/state.jpg)

One state having 122% let me know that I was dealing with duplicate data, which I then had to reconcile and clean after the data collection exercise. Here's another from the next year which let me know I had no alarming levels of duplicate data.

![](/assets/images/state2.jpg)

Building a dashboard was always going to be overkill for this process because the data collection period always ranged from a few days to 1 week, and the data wouldn't need to be visualised again for a long time. This was my process from 2023 till 2024.

# Workflow in 2026 Using Claude Code

Now to where AI comes in — building a dashboard became a lot easier than in the past. I could easily draw out live statistics on the data as it was being collected without needing to run a script every few hours. So I went straight into vibe coding to see what I could build. I was essentially looking to build a throw-away dashboard. Here are a few requirements I had when I started:

* View the number of schools visited
* Percentage of active enumerators (we have had cases in the past where only one enumerator was working in a local government area, but they both would get paid — we wanted to curb that)
* Percentage of schools visited
* A visual graph of locations visited using GPS data collected
* How many submissions submitted per local government
* How many submissions submitted per enumerator

These were the starting requirements. I gave Claude Code my KoboToolbox form, which informed it on the structure of my data and how it would need to scaffold the project. I had my token in an `.env` file, and the first draft was disappointing — unfortunately I did not save screenshots. This is what I had at the end of about a 3 hour session of back and forth.

![dashboard statistics](/assets/images/dashboard.png)

It was amazingly useful and I kept adding features as we went, like daily rate of submissions and who was making those submissions.

![dashboard daily](/assets/images/dashboard-daily.png)

I love being able to see live statistics about the data as it's being collected, and the client loves it even more. After the week of data collection, I clean up the data and follow my ETL Pipeline for adding it to the MIS Dashboard for managing student attendance and disbursements.