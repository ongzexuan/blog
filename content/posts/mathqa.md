---
title: "Math Question Answering"
date: 2021-10-21T18:00:00-00:00
author: Ze Xuan Ong
draft: false
feature_image: /mathqa/feature.png
tags: ["dev"]
summary: "Describing my interest in MathQA, and collecting good papers and resources in a single place."
---

One holdover academic interest from my time at Miao Technologies was the idea that we could build a thinking machine that solve math word problems. Not the equation kind, but the kind where Billy has twenty more apples than Ali, and you're supposed to figure out how many fruits they have in the aggregate.

## Significance

Before delving into the complexities of how this can be done, we should consider first the implications that the existence of such a system might mean for us, particularly future generations of students (clearly in this regard I've outgrown that, but I'm sure my children will have to deal with this eventually). Suppose there was such a machine that could, given a math problem embedded in a natural language context, magically tell you what the numerical answer was. If we allow ourselves a little more imagination, perhaps it could even show you the steps it took to derive the said answer. 






## Schools of Thought

### Slot Filling

In the earlier days where compute was expensive and phones had tails, there was a small but growing interest in this topic. The approach they took to this problem is akin to slot-filling at a casino. That is, it presumes that math problems come in a variety of templates, and if we could just figure out which template a question was, we could apply a formula to the variables and get an answer. Consider:

```
Question 1: Ali has 20 apples. Billy has 30 more apples than Ali. How many do they have altogether?

Question 2: Ali has 40 apples. Billy has 20 more apples than Ali. How many do they have altogether?
```

The answer to both questions is: more than they can carry, but you get the point. The solution to any question that comes in such a form (template), is to take the first number and add it to the second number. Hence the idea of 'slots'.

Thus, early research focused on a few questions:
* How can we accurrately determine the correct template to use?
* How can we make such methods of matching robust to changes in the language?

The main problem with these approaches that was sort of elided away was the sheer lack of generalizability that these templates have in the real world. There are an infinite number of possible ways to ask a math question, so it naturally follows that there are an infinite number of templates. Here is where I try to be clever and assert that, while the previous statement is theoretically true, in practice the set of mathematical knowledge and forms of language experssions that are permissible to be inflicted on K-12 students in curriculum globally is a constrained set that grows slowly. Sure it is large, and probably the template frequency chart probably has an infinitely long tail, but it would also mean that you can cover a lot of ground with a large finite number of templates. After all, we never made guarantees that our magical machine will always give the right answer, but it will try its best and give the right answer most of the time. Given these relaxations, I think a slot-filling approach can still get you very far, as long as you have a large number of templates. The million dollar question here is what is that number look like? 

### Equation Representation

Implicit in the approach of slot filling is that there exists a single formula to solve every 'type' of question. From personal experience, I find this to be true, in that for K-12 math problems there is almost always a peferred canonical approach to solving the problem. How else can one cram for exams if there is no right answer? Perhaps we can give up the idea of


### World Representation

I've spent some time looking at the various solutions to date, and none of them have been particularly exciting or satisfactory. The part that nags at me especially is the part where it is difficult to deterrmine if a model has actually learned anything about math, or has it just leant to memorize answers?

A recent anti-dataset paper tried to prove this point. It took math questions and made a bunch of transformations to them - asking different questions, rearranging the parts so that it was still the same question but was just grammatically different, and even went as far as to make some questions non-answerable. The conclusion of the paper is that pretty much all models failed to really learn anything. So this is kinda disappointing.

This has led me to reconsider how we might improve our approach to solving this problem. One idea that I've thought about quite a bit (I haven't really seen anyone work on this) is the idea of state representation, or world representation. In typical problems, every line except the last one helps the student to build up a context of the problem. This context could really be anything, from two trains moving towards each other at some speed, to more abstract things like entangled shapes with shaded area that unsurprisingly needs to be determined. Many things can be asked about this rich context, not just the particular one that was asked in the question itself. It just happens that teachers and curricula have arbitarily defined it that way.

I think the more intuitive and humanistic way to think about solving math problems is as a two-stage process. In the first stage, we parse the question and try to build up an accurate representation of the world. In students, many things can go wrong here. From not noticing that a particular line has the same length as that of the other line on the opposite side, to not realizing that horses have 4 legs but ducks only have 2 (and that somehow a few hundred of them can fit in a barn where you can count legs but no bodies). Without an accurate state representation of the world as described in the problem context, the math problem necessarily cannot be solved correctly. In the second stage, we then take this accurate representation of the world, together with the actual question prompy, and apply our mathematical knowledge to convert both of these inputs into a numerical answer, hopefully correct.

The problem with contemporary approaches to this problem is that they try to do both of these separate steps in a big leap. Sure, BERT and co are powerful tools, but it's not obvious to me that we can combine both language representation and mathematical reasoning ability into a single model and expect to do robustly well. In this regard, I think we could be better served by treating this as two separate problems to be solved by some ensemble of heterogeneous approaches.

#### Problems with Representation

One outstanding issue with this that I have not quite yet figured out is the difficulty of reprresenting this world state. How exactly should we encode this information? And what kind of granularity does it need to be? I'm not exactly sure, neither am I sure how that can be stuffed in a GPU. Let's not even mention the difficulty of encoding external persumed information about the world e.g. that apples are fruits, trains do travel, horses have 4 legs and snakes have none etc.

Perhaps






Sure, the numbers are different, but the underlying question and grammatical structure of the problem are identical.



Accessing the sheet and making edits is really simple with this package:

```
import gspread

# Get access to worksheet
gc = gspread.service_account(filename=CREDENTIALS_FILE)
spreadsheet = gc.open("My Spreadsheet Name")
worksheet = spreadsheet.worksheet("My Worksheet Name")

# Append data
my_data = ["something"]
worksheet.append_row(my_data)

```

I'm quite happy with this package. My minor unimportant complaint is that without looking under the hood it's not immediately obvious how many API calls are made to Google's server for each function call. This has led to me exhausting the API rate limit when running backfill tasks too fast, but that's probably on me.


### Backfilling With New Tasks

Previously we had 4 single independent tasks, so there was nothing DAG-gy about it. Now we finally add a dependency - we export our data from the database to GSheets once we finished collecting and transforming the data from all sources.

{{< figure src="/airflow-etl-gsheets/dag.png" caption="Our now very complex DAG." >}}

This transition from 4 tasks to 5 tasks is handled fine by Airflow. If we now go back and look at old DAG runs, we'll notice that there is now a new downstream dependent task just as in the image, but because it was a past DAG run, that task was never run. If we wish to we can run a backfill specifically on this task. One minor downside is that I had assumed that because I was appending only to my spreadsheet, the data on the spreadsheet will be sorted by date naturally. With backfills running multiple task instances in parallel however, the finish times are non-deterministic, so the output is appended with imperfect sorting. This is a small problem that can be fixed by sorting the sheet, since this is a one-time problem.

After performing our backfill to the start of the year, we can have a look at the worksheet in the GSheet we've been exporting to.

{{< figure src="/airflow-etl-gsheets/gsheet.png" caption="The GSheet now chock full of data." >}}

### Slapping Slack on it

This is probably a good point in time to add some monitoring to the Unraid server we've been running this thing on. We're running a number of containers and VMs, some of which do things on a regular basis, so it'd be good to know if they succeeded or failed without having to login and check. Slack is probably the best option, so we setup a Slack workspace for our server, added an Airflow bot user, and passed the webhook URL to a Airflow Connection named `SLACK_CONN_ID`.

I borrowed this [nice pair of slack operators](https://github.com/ongzexuan/airflow-transaction-dags/blob/master/slack_operator.py) - there's one for success and one for failure. To integrate this into our DAG, we invoke the `on_success_callback` and `on_failure_callback` option for the `Operator`s. Specifically, at the very last task, we add the two respective calls into the keyword arguments as follows:

```
# Task: Export data to GSheets
export_gsheet_task = PythonOperator(task_id="export_to_gsheet_task",
                                    python_callable=export_to_gsheet,
                                    provide_context=True,
                                    on_success_callback=task_success_slack_alert,
                                    on_failure_callback=task_fail_slack_alert)
```

Then, when we do a test run of the DAG, we get some results.

{{< figure src="/airflow-etl-gsheets/slack.png" caption="Slack notifications for the local Airflow instance. More notifications coming soon!" >}}





