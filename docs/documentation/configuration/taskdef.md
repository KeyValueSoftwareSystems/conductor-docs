# Task Definition

Tasks are the building blocks of workflow in Conductor. A task can be an operator, system task or a custom worker.

A typical Conductor workflow is a list of tasks that are executed until completion or the termination of the workflow.

## Workers
A worker is responsible for executing a task.  Operator and System tasks are handled by the Conductor server, while user 
defined tasks needs to have a worker created that awaits the work to be scheduled by the server for it to be executed.
Workers can be implemented in any language, and Conductor provides support for Java, Golang and Python worker framework that provides features such as 
polling threads, metrics and server communication that makes creating workers easy.

Each worker embodies Microservice design pattern and follows certain basic principles:

1. Workers are stateless and do not implement a workflow specific logic.  
2. Each worker executes a very specific task and produces well defined output given specific inputs.
3. Workers are meant to be idempotent (or should handle cases where the task that partially executed gets rescheduled due to timeouts etc.)
4. Workers do not implement the logic to handle retries etc, that is taken care by the Conductor server.
 
Conductor maintains a registry of worker tasks.  A task MUST be registered before being used in a workflow.

## Schema
**Example**
``` json
{
  "name": "encode_task",
  "retryCount": 3,
  "timeoutSeconds": 1200,
  "inputKeys": [
    "sourceRequestId",
    "qcElementType"
  ],
  "outputKeys": [
    "state",
    "skipped",
    "result"
  ],
  "timeoutPolicy": "TIME_OUT_WF",
  "retryLogic": "FIXED",
  "retryDelaySeconds": 600,
  "responseTimeoutSeconds": 3600,
  "pollTimeoutSeconds": 3600,
  "concurrentExecLimit": 100,
  "rateLimitFrequencyInSeconds": 60,
  "rateLimitPerFrequency": 50,
  "ownerEmail": "foo@bar.com",
  "description": "Sample Encoding task"
}
```

| Field                       | Type                       | Description                                                                                                                                                                                                    | Notes                                                                                                                                                 |
|:----------------------------|:---------------------------|:---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:------------------------------------------------------------------------------------------------------------------------------------------------------|
| name                        | string                     | Task Type. Unique name of the Task that resonates with its function.                                                                                                                                           | unique                                                                                                                                                |
| description                 | string                     | Description of the task                                                                                                                                                                                        | optional                                                                                                                                              |
| retryCount                  | number                     | No. of retries to attempt when a Task is marked as failure                                                                                                                                                     | defaults to 3 with maximum allowed capped at 10                                                                                                       |
| retryLogic                  | string (restricted values) | Mechanism for the retries                                                                                                                                                                                      | see possible values below                                                                                                                             |
| retryDelaySeconds           | number                     | Time to wait before retries                                                                                                                                                                                    | defaults to 60 seconds                                                                                                                                |
| timeoutPolicy               | string (restricted values) | Task's timeout policy                                                                                                                                                                                          | defaults to `TIME_OUT_WF`; see [Timeout Policy](#timeout-policy)                                                                                                                            |
| timeoutSeconds              | number                     | Time in seconds, after which the task is marked as `TIMED_OUT` if not completed after transitioning to `IN_PROGRESS` status for the first time                                                                 | no timeouts if set to 0                                                                                                                               |
| responseTimeoutSeconds      | number                     | If greater than 0, the task is rescheduled if not updated with a status after this time (heartbeat mechanism). Useful when the worker polls for the task but fails to complete due to errors/network failure.  | defaults to 3600; see [Response timeout best practices](#response-timeout) |
| pollTimeoutSeconds          | number                     | Time in seconds, after which the task is marked as `TIMED_OUT` if not polled by a worker                                                                                                                       | no timeouts if set to 0                                                                                                                               |
| inputKeys                   | array of string(s)         | Array of keys of task's expected input. Used for documenting task's input. See [Using inputKeys and outputKeys](#using-inputkeys-and-outputkeys).                                                              | optional                                                                                                                                              |
| outputKeys                  | array of string(s)         | Array of keys of task's expected output. Used for documenting task's output.                                                                                                                                   | optional                                                                                                                                              |
| inputTemplate               | object                     | See [Using inputTemplate](#using-inputtemplate) below                                                                                                                                                          | optional                                                                                                                                              |
| concurrentExecLimit         | number                     | Number of tasks that can be executed at any given time                                                                                                                                                         | optional                                                                                                                                              |
| rateLimitFrequencyInSeconds | number                     | See [Task Rate limits](#task-rate-limits) below                                                                                                                                                                | optional                                                                                                                                              |
| rateLimitPerFrequency       | number                     | See [Task Rate limits](#task-rate-limits) below                                                                                                                                                                | optional                                                                                                                                              |
| ownerEmail                  | string                     | Email address of the team that owns the task                                                                                                                                                                   | required                                                                                                                                              |

## Retry Logic

* FIXED: Reschedule the task after `retryDelaySeconds`
* EXPONENTIAL_BACKOFF: Reschedule the task after `retryDelaySeconds * (2 ^ attemptNumber)`
* LINEAR_BACKOFF: Reschedule after `retryDelaySeconds * backoffRate * attemptNumber`
 
## Timeout Policy

* RETRY: Retries the task again
* TIME_OUT_WF: Workflow is marked as TIMED_OUT and terminated. This is the default value.
* ALERT_ONLY: Registers a counter (task_timeout)

## Task Concurrent Execution Limits

* `concurrentExecLimit` limits the number of simultaneous Task executions at any point.
**Example:**  
If you have 1000 task executions waiting in the queue, and 1000 workers polling this queue for tasks, but if you have set `concurrentExecLimit` to 10, only 10 tasks would be given to workers (which would lead to starvation). If any of the workers finishes execution, a new task(s) will be removed from the queue, while still keeping the current execution count to 10.

### Task Rate limits

> Note: Rate limiting is only supported for the Redis-persistence module and is not available with other persistence layers.

* `rateLimitFrequencyInSeconds` and `rateLimitPerFrequency` should be used together.
* `rateLimitFrequencyInSeconds` sets the "frequency window", i.e the `duration` to be used in `events per duration`. Eg: 1s, 5s, 60s, 300s etc.
* `rateLimitPerFrequency`defines the number of Tasks that can be given to Workers per given "frequency window". No rate limit if set to 0.

**Example:**  
Let's set `rateLimitFrequencyInSeconds = 5`, and `rateLimitPerFrequency = 12`. This means our frequency window is of 5 seconds duration, and for each frequency window, Conductor would only give 12 tasks to workers. So, in a given minute, Conductor would only give 12*(60/5) = 144 tasks to workers irrespective of the number of workers that are polling for the task.  
Note that unlike `concurrentExecLimit`, rate limiting doesn't take into account tasks already in progress/completed. Even if all the previous tasks are executed within 1 sec, or would take a few days, the new tasks are still given to workers at configured frequency, 144 tasks per minute in above example.   


## Using inputKeys and outputKeys

* `inputKeys` and `outputKeys` can be considered as parameters and return values for the Task.
* Consider the task Definition as being represented by an interface: ```(value1, value2 .. valueN) someTaskDefinition(key1, key2 .. keyN);```
* However, these parameters are not strictly enforced at the moment. Both `inputKeys` and `outputKeys` act as a documentation for task re-use. The tasks in workflow need not define all of the keys in the task definition.
* In the future, this can be extended to be a strict template that all task implementations must adhere to, just like interfaces in programming languages.

## Using inputTemplate

* `inputTemplate` allows to define default values, which can be overridden by values provided in Workflow.
* Eg: In your Task Definition, you can define your inputTemplate as:

```json
"inputTemplate": {
    "url": "https://some_url:7004"
}
```

* Now, in your workflow Definition, when using above task, you can use the default `url` or override with something else in the task's `inputParameters`.

```json
"inputParameters": {
    "url": "${workflow.input.some_new_url}"
}
```