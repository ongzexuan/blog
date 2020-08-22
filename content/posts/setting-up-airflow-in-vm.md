---
title: "Setting Up Apache Airflow in a VM"
date: 2020-08-19T00:49:17-04:00
draft: false
feature_image: /setting-up-airflow-in-vm/airflow_interface.png
summary: "I want all my credit card transaction data in one place. I'm going to use Airflow to schedule jobs that use the Plaid API to extract and process this data on a daily basis."
---

I'm setting up an Airflow instance for my Unraid server. To be perfectly clear, I don't really need-need Airflow. For my workloads currently, a cron job with some logging capability is sufficient. But I've used Airflow before at an internship and it left a great impression on me, and arguably this would future-proof my future needs, in the same manner that my mom buys size XL clothes to future proof my growth (didn't work out). This weekend offers some respite between job training, which is just enough time to get this up and running.

On a more serious note, there are some features of Airflow that I particularly like. The first is the concept of backfills. Suppose I create a script that is intended to run daily. If I finish writing and deploying the script today, then it runs daily from today onwards, but what I also want is the result of running the script on all the days that came before, preferably from the first day of the year. Normally, this would entail writing some extra script that calls the first script with customized parameters. Depending on the complexity of the original script and how modularly it was written, this may get messy. Airflow resolves this problem by running each DAG (script) with a date, so it is as if the DAG was run on the particular day. 

The other concept that I like is the structuring of workflows as DAGs, which is sometimes necessary. For example, there is no point running the data export service if the data collection fails. The direct dependency relationship between the workflows is respected with Airflow. This also means that if some parts of my workflow fail, I can fix and re-run only those parts.

There's a ton of other great features that Airflow has that I don't personally use. For example, tasks are run in the order of the DAG sort, so you can do them in parallel and really fast. For myself I don't have enough data or threads to utilize that, so that's nice icing on the cake.

### Context and Task

What I'd like to do with this Airflow installation is to pull my credit card transaction data on a daily basis, process it according to some rules, and then put the data in a PostgreSQL database. I already have a [script](https://github.com/ongzexuan/airflow-dags) setup to do just that which uses the [Plaid API](https://plaid.com/), packaged as an Airflow DAG. Because Plaid already standardizes the data format for us from the different institutions, pulling data from each credit card (institution) is an identical process, only with different access tokens corresponding to each of my different card (institutions). We designate the pulling of data from each institution as a separate task.

### Installing Airflow

I'm planning to run Airflow and its database on a VM in Unraid. This way I can dedicate more resources to it on the Unraid interface (more cores and memory).

The [Airflow](https://airflow.apache.org/docs/stable/installation.html) installation page provides most required details for setting it up, so I won't elaborate too much here, it's the configuration settings that are more tricky. For my case I'm going to use Postgres for my database, so I'll install that first in the (Ubuntu) VM, before installing and initializing Airflow.

```
sudo apt-get install -y postgres postgres-contrib
```

Verify that Postgres is installed by logging in as the root user.

```
sudo -u postgres psql
```

Then (optionally) create a database for our working data and user for the database. Note this is not the same database that Airflow uses, we initialize that later. My [repo](https://github.com/ongzexuan/airflow-dags) has an example SQL script for that.

### Configuration Settings

Next we need to configure the relevant Airflow settings before running it.

In the `$AIRFLOW_HOME$` directory, which by default should be `~/airflow`, you should find the `airflow.cfg` file. This contains all the configurations for Airflow. There are a few key things we want to edit.

`sql_alchemy_conn` - change this to reflect the database connection string for the database you want to connect to.

`executor` - we use `LocalExecutor` here because we intend to execute our tasks locally on a single-node machine. In production you'd probably use something that can execute tasks in parallel such as `CeleryExecutor`.

`parallelism` - I reduced it to 4 because I only have a single core (two virtual cores) for this VM.

`max_active_dag_runs_per_dag` - I reduced it to 4 for similar reasons, I don't need a great amount of parallelism, and I don't need many tasks running at the same time.

`dag_concurrency` - I reduced it to 8 for similar reasons.

`load_examples` - I changed this to `False` to hide the examples.

You might need to change the `dags_folder` option to point to where your DAGs are stored. You may also need to export `$AIRFLOW_HOME` in your `.bashrc` file.

### Initialize Airflow

Once you have finished configuring the settings, in particular the database settings, we can go ahead and initialize the Airflow database, which will keep track of all our DAGs and their runs.

```
airflow initdb
```

### Systemd Script

We need to run both the Airflow webserver (for GUI access to logs and control), and the Airflow scheduler which schedules tasks (to be run). We could run both manually, Airflow's tutorial shows you how to do just that, but then managing it is a pain, so let's wrap it in a `systemd` script. There's a few online, mine is as follows, they're more or less identical except what you run.

{{<gist 4fc808b432e4af93d4a38a1fdaf9d367 >}}

{{<gist 3f6c4d7504e2d01473f99e7861364d31 >}}

The most annoying bit that took a long while to debug is that my `python` modules are installed in the `~/.local/bin` directory (Ubuntu LTS 18.04), I have to point the `systemd` script to where those modules are using the `ENVIRONMENT` flag. In particular, Airflow cannot run `gunicorn` without that.

### Start Airflow

Now we can start Airflow, both the webserver and the scheduler. The scheduler is needed to actually run the tasks.

```
sudo service airflow-webserver start
sudo service airflow-scheduler start
```

You should now see the Airflow UI on port 8080 of whichever host you are running this on!

{{<figure src="/setting-up-airflow-in-vm/airflow_interface.png" caption="Airflow UI showing the single DAG that I have, scheduled to run daily at 23:59." >}}

We can go ahead and fill in our environment variables in the Variables tab for our script to run.

### Backfilling

Now that we have Airflow up and running, we want to go ahead and backfill the tasks for the rest of the year that came before. In our case, we want to execute our data collection and processing script on every day before today in 2020. This is achieved as follows.

```
airflow backfill transaction_dag -B --delay_on_limit 5.0 --rerun_failed_tasks -s 2020-01-01 -e 2020-08-20
```

The [flags](https://airflow.apache.org/docs/stable/cli-ref#backfill) are mostly optional. The only really important ones are the start date and end date. Note that this does not override your DAG start date (in my case the DAG start date is 1st Jan 2020, so it wouldn't matter whether I specified the dates or not). We use the `-B` flag to run the backfill backwards in time from the end date, for no real reason, I just prefer the database inserts to work that way. We use the `--delay_on_limit` flag to add some latency between our max DAG runs, since we're using an external (free) API and don't want to accidentally trigger rate limits. We use the `--rerun_failed_tasks` to, obviously rerun failed tasks, since in our case we have 4 tasks per DAG, any of which can fail due to external API problems, and trying to go over the run log and rerunning specific tasks if they fail is labor intensive. Though really, no tasks should fail if the API works alright. You can add the `-dr` dry run flag before you really really run the backfill, and you can also run it with smaller date spans in case something blows up. In either case our script is also idempotent, so running it multiple times does the same thing as running it once, so in theory we could just backfill again if anything fails, depends on what you're running.

{{<figure src="/setting-up-airflow-in-vm/dag_runs.png" caption="Airflow UI showing backfilled DAG runs." >}}

The DAG runs succeed as you can see on the interface. As a final verification, let's take a look in the database using DataGrip.

{{<figure src="/setting-up-airflow-in-vm/datagrip.png" caption="Data exists in the database!" >}}

The data looks about right in the database. Note that negative amounts reflect refunds and credits and the like. And so we are done! Plaid provides its own categorization, and by and large it is mostly right. For accurate accounting though, I'd need it to be 100% right, and so the next thing I'd like to do is to create a categorization process during the processing step. This could simply be a set of hardcoded rules, since the variety of things I buy on my credit card is relatively small and slow growing. It will mostly be targeted and fixing the categorization errors made by Plaid on a small number of transactions.

### Some caveats

- We are entirely dependent on the well-functioning and generosity of the Plaid Development API for this to work. I'm allowed up to 100 Items (institutions) on this account. I doubt I'll ever try to open credit cards at more than 20 unique institutions in my lifetime, so that's all good.
- The goal of putting it in some database is so that I can export this data into a spreadsheet whenever I want to. Microsoft is actually working on a [Money in Excel](https://templates.office.com/en-us/money-in-excel-tm77948210) plugin that does exactly this using the Plaid API. It was announced for a long time but it took really long to actually deliver. I think it recently was released. In any case, I wanted more control of my data, so I was going to go ahead with this anyway.









