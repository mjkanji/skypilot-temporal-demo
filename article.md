# TODOs

- Finalize how env vars are set (`export` or `source .env`).
- Should the example live in a separate repo?

# Durable Execution of AI Workloads with Temporal and SkyPilot

AI is eating the world — and your org’s budget. As demand for AI-powered features surges across industries, teams are 
under pressure to deliver and iterate faster than ever.

In this competitive landscape, the ability to iterate and learn faster provides a meaningful advantage. Temporal addresses 
this by allowing AI/ML developers to [orchestrate complex workflows](https://temporal.io/blog/ai-ml-and-data-engineering-workflows-with-temporal) 
with ease, enabling them to focus on business logic rather than building complex state machines or writing error-handling code.

But while Temporal tackles orchestration, challenges like GPU scarcity, rising hardware costs, and infrastructure 
complexity are still hampering developers' ability to innovate and deliver new products effectively.

Using multiple cloud providers to improve GPU availability and reduce costs seems like an obvious fix, but the 
operational overhead from managing a multi-cloud setup can be rather significant, often requiring a dedicated DevOps 
team and more than one Kubernetes ninja.

Surely, there's a better way to do this?

## Enter SkyPilot: Run AI on Any Infra

SkyPilot abstracts the complexities of multi-cloud setups by presenting all of your cloud infrastructure as a unified 
compute pool — the Sky to your clouds. 

It offers a simple, unified, and declarative interface for defining and provisioning infrastructure across this unified 
compute pool. This enables AI teams to quickly and easily launch AI training, inference, and batch workloads across cloud 
providers without needing deep expertise in infrastructure or DevOps.

Say you want to run an AI training job on a GPU-accelerated machine with 4 vCPUs, 16GB of RAM, and an A100. With 
SkyPilot, it's as simple as defining your requirements in a YAML file:

```yaml 
resources:
  cloud: aws
  region: us-east-1
  cpu: 4
  memory: 16
  accelerators: A100

setup: |
  pip install my-dependencies

run: |
  python run my-training-job.py
```

Then, run the following from your terminal:

```bash
sky launch -c mycluster training_job.yaml
```

SkyPilot provisions the VM, runs the job, then shuts down the machine.

If the GPU configuration isn't available in the selected region, simply remove the `region:` line and SkyPilot will 
search across all regions.

Better yet, instead of specifying a cloud provider, you can let SkyPilot choose one for you. It'll automatically look 
through the catalog of the 16+ providers it currently supports and choose the cheapest available option.

SkyPilot also supports managed job queues with spot instances (with automatic recovery from preemption), hyperparameter 
search across thousands of instances on multiple clouds, and reproducible cloud development environments. When it's 
time to deploy, it can serve your LLM and scale automatically across clouds and regions.

## Temporal + SkyPilot: Durable Execution across Clouds

AI pipelines typically consist of many interdependent steps that can fail and require retries. Temporal is ideal for 
orchestrating such workflows, and using SkyPilot and Temporal together can supercharge your AI team's productivity by 
abstracting both infrastructure management and process orchestration, allowing AI developers to focus on optimizing 
business logic and model performance.

Below, we show an example of using Temporal to orchestrate SkyPilot tasks. 

### Setup

Start by setting up a SkyPilot API server, which handles infrastructure provisioning and state. Follow the 
[API server deployment guide](https://docs.skypilot.co/en/latest/reference/api-server/api-server-admin-deploy.html).

Then, install SkyPilot on your Temporal workers and log in to the API server:

```bash
$ pip install -U skypilot-nightly
$ sky api login
```

For this example, we'll work with a mock workflow consisting of data preprocessing, training, and evaluation steps 
[defined as SkyPilot tasks](https://github.com/skypilot-org/mock-train-workflow/tree/clientserver_example). These tasks 
are wrapped in Temporal Activities and composed into an end-to-end ML workflow. 

You can find the full code for the demo here [TODO: Add link.].

### Explanation

The demo code provides Temporal wrappers for the following SkyPilot commands:

- `sky launch`: Provisions infrastructure and runs a task
- `sky exec`: Runs a task on existing infrastructure
- `sky down`: Terminates infrastructure

These operations are wrapped in Temporal activities, making the entire workflow durable and resilient to failures.

The `SkyPilotWorkflow` below orchestrates an end-to-end ML pipeline through six activities:

1\. `run_git_clone` clones the `mock-train-workflow` repo from GitHub onto the Temporal Worker, so it has access to the code that
the SkyPilot job must execute.

```python
clone_result = await workflow.execute_activity(
    run_git_clone,
    GitCloneInput(repo_url=repo_url,
                    clone_path=clone_path,
                    yaml_file_paths=yaml_paths,
                    branch=branch,
                    api_server_endpoint=api_server_endpoint),
    start_to_close_timeout=timedelta(minutes=5),
    retry_policy=retry_policy,
    task_queue=WORKER_TASK_QUEUE,
)
```

2\. The second step uses the `run_sky_launch` activity to launch a SkyPilot cluster
to execute the task defined in `data_preprocessing.yaml`.
```python
preprocess_result = await workflow.execute_activity(
    run_sky_launch,
    SkyLaunchCommand(cluster_name=cluster_name,
                    yaml_content=preprocess_yaml,
                    launch_kwargs=launch_kwargs,
                    envs_override=input.envs_override,
                    api_server_endpoint=api_server_endpoint),
    start_to_close_timeout=timedelta(minutes=30),
    retry_policy=retry_policy,
    task_queue=WORKER_TASK_QUEUE,
)
```

3\. Next, we use the `run_sky_down` activity to terminate the preprocessing cluster
from the last step.

```python
down_result = await workflow.execute_activity(
    run_sky_down,
    SkyDownCommand(cluster_name=cluster_name,
                    api_server_endpoint=api_server_endpoint),
    start_to_close_timeout=timedelta(minutes=10),
    retry_policy=retry_policy,
    task_queue=WORKER_TASK_QUEUE,
)
```

4. The fourth step uses the `run_sky_launch` activity to create a new cluster
for the model training job, as defined in `train.yaml`.

```python
train_result = await workflow.execute_activity(
    run_sky_launch,
    SkyLaunchCommand(cluster_name=cluster_name,
                    yaml_content=train_yaml,
                    launch_kwargs=launch_kwargs,
                    envs_override=input.envs_override,
                    api_server_endpoint=api_server_endpoint),
    start_to_close_timeout=timedelta(hours=1),
    retry_policy=retry_policy,
    task_queue=WORKER_TASK_QUEUE,
)
```

5\. Next, we evaluate the model. Because training and inference often have
similar computational requirements, we re-use the training cluster through the
`run_sky_exec` activity.

```python
eval_result = await workflow.execute_activity(
    run_sky_exec,
    SkyExecCommand(cluster_name=cluster_name,
                    yaml_content=eval_yaml,
                    exec_kwargs=exec_kwargs,
                    envs_override=input.envs_override,
                    api_server_endpoint=api_server_endpoint),
    start_to_close_timeout=timedelta(minutes=30),
    retry_policy=retry_policy,
    task_queue=WORKER_TASK_QUEUE,
)
```

6. Finally, we tear down the training cluster at the end of the workflow.

```python
down_result = await workflow.execute_activity(
    run_sky_down,
    SkyDownCommand(cluster_name=cluster_name,
                    api_server_endpoint=api_server_endpoint),
    start_to_close_timeout=timedelta(minutes=10),
    retry_policy=retry_policy,
    task_queue=WORKER_TASK_QUEUE,
)
```

### Demo

Next, let's try running the example using Temporal. To do so, you'll need to have Temporal SDK for Python and the 
Temporal CLI installed on your machine. Instructions on how to do so can be found 
[here](https://learn.temporal.io/getting_started/python/dev_environment/).

With your development environment set up, start a local instance of the Temporal service:

```bash
temporal server start-dev
```

This will start the Temporal Service (usually at port 7233) and Temporal Web UI (usually at port 8233).

Next, open a new terminal to schedule the workflow for execution:

```bash
# In a new terminal, set the env vars and run the workflow
source .env
python run_workflow.py
```

If you go to the Temporal Web UI (normally at `http://localhost:8233`), you can see the Workflow listed.

![](assets/temporal-ui-1.png)

Click on `skypilot-workflow-id` to see more details about the execution of the workflow.

![](assets/temporal-ui-2.png)

In this case, we see that the Workflow is queued but has not yet started execution because there are no available 
Workers. To fix this, start a local Worker to execute the workflow from a new terminal:

```bash
# In a new terminal, set the env vars and start the worker
source .env
python run_worker.py
```

When you go back to the Temporal Web UI, you should see the event history and logs from the workflow streaming in. When 
the workflow has successfully executed, you should see the different activities, their status, and retry attempts (if any) laid out in the Event History:

![](assets/temporal-ui-3.png)

## Wrapping Up

In this example, we demonstrated how Temporal and SkyPilot can help your AI team move faster by simplifying 
infrastructure management and process orchestration, allowing them to focus on optimizing business logic and model performance.

If you're a platform engineer, this might be particularly interesting to you because using SkyPilot with Temporal 
allows you to quickly onboard data scientists' experiments into production settings. 

During R&D, your data team can rapidly iterate and provision resources using SkyPilot's declarative API independently, 
without needing to learn about how Temporal works. And when it's time to deploy the trained models to production, 
they can bring over their SkyPilot task specifications to Temporal by reusing a generic set of Temporal Activities 
(like the ones we used in this example) to run any SkyPilot task, minimizing any overhead for the platform engineering 
team during deployment.
