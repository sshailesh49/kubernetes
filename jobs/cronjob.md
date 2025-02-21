# **Kubernetes CronJob: Complete Explanation of All Attributes**  

A **Kubernetes CronJob** is used for running periodic tasks in a Kubernetes cluster. It works like a Linux cron job but runs inside the cluster using Kubernetes Jobs.

---

## **CronJob API Specification**
A `CronJob` is defined using the `batch/v1` API and has multiple attributes. Here is a breakdown of all available attributes:

        ```yaml
        apiVersion: batch/v1
        kind: CronJob
        metadata:
          name: my-cronjob
          labels:
            app: cronjob-example
        spec:
            schedule: "*/5 * * * *"   # (1) Schedule - Runs every 5 minutes
            timeZone: "UTC"           # (2) Optional - Set timezone
            startingDeadlineSeconds: 100  # (3) Time window to start missed jobs
            concurrencyPolicy: Forbid  # (4) Controls concurrent execution
            suspend: false            # (5) If true, CronJob is paused
            successfulJobsHistoryLimit: 3  # (6) Limits saved successful jobs
            failedJobsHistoryLimit: 1      # (7) Limits saved failed jobs

            jobTemplate:    # (8) Job definition
                    spec:
                        parallelism: 1   # (9) Number of pods that run in parallel
                        completions: 1   # (10) Total number of pods that must complete
                        backoffLimit: 3  # (11) Retry attempts for failed jobs
                        activeDeadlineSeconds: 200  # (12) Timeout for the job
                        template:
                            spec:
                            restartPolicy: OnFailure  # (13) Pod restart behavior
                            containers:
                               - name: cron-container
                                 image: busybox
                                 args:
                                    - /bin/sh
                                    - -c
                                    - "echo Hello from Kubernetes CronJob && sleep 30"
        ```

        ---

## **Explanation of All Attributes**
    | Attribute  | Description | Example |
    |------------|------------|---------|
    | `schedule`  | Cron expression defining when the job runs. | `"*/5 * * * *"` (every 5 minutes) |
    | `timeZone`  | Specifies the time zone for scheduling. | `"UTC"` or `"America/New_York"` |
    | `startingDeadlineSeconds` | Maximum time (in seconds) allowed to start a job if it misses its schedule. | `100` |
    | `concurrencyPolicy` | Defines how jobs should behave when multiple instances are scheduled at the same time. Options: `Allow`, `Forbid`, `Replace`. | `"Forbid"` (prevents overlap) |
    | `suspend` | If `true`, suspends job execution. | `false` |
    | `successfulJobsHistoryLimit` | Number of successful jobs to keep. | `3` |
    | `failedJobsHistoryLimit` | Number of failed jobs to keep. | `1` |
    | **`jobTemplate`** | Defines the job template to be executed. | **See below** |
    | `parallelism` | Number of pods running in parallel. | `1` |
    | `completions` | Number of pods that must complete before the job is successful. | `1` |
    | `backoffLimit` | Number of retry attempts before marking a job as failed. | `3` |
    | `activeDeadlineSeconds` | Time limit for a job to run before termination. | `200` |
    | `restartPolicy` | Controls pod restart behavior (`OnFailure`, `Never`). | `OnFailure` |
    | `containers` | Defines the containers and commands to run in the job. | See YAML example |

---

## **1. Cron Schedule Syntax**
The **schedule** uses standard **cron syntax**:

        ```
        ┌───────────── minute (0 - 59)
        │ ┌───────────── hour (0 - 23)
        │ │ ┌───────────── day of month (1 - 31)
        │ │ │ ┌───────────── month (1 - 12)
        │ │ │ │ ┌───────────── day of week (0 - 6) (Sunday = 0 or 7)
        │ │ │ │ │
        * * * * *  command_to_run
        ```

### **Example Schedules:**
    | Expression | Meaning |
    |------------|---------|
    | `*/5 * * * *` | Every 5 minutes |
    | `0 * * * *` | Every hour |
    | `0 12 * * *` | Every day at 12:00 PM |
    | `0 0 1 * *` | First day of the month at midnight |
    | `0 0 * * 0` | Every Sunday at midnight |

---

## **2. Handling Job Concurrency**
Kubernetes CronJobs allow control over job execution concurrency with `concurrencyPolicy`:
- **`Allow` (default):** Runs new jobs even if previous jobs are still running.
- **`Forbid`:** Prevents a new job from running if the previous job is still active.
- **`Replace`:** Kills the currently running job and starts a new one.

Example:
    ```yaml
    concurrencyPolicy: Forbid
    ```

---

## **3. Ensuring Jobs Run Even After Cluster Downtime**
- **Missed job handling**: If a job is scheduled but the cluster is down, it won’t run unless `startingDeadlineSeconds` is set.
- **Example:**
        ```yaml
        startingDeadlineSeconds: 100
        ```
If a job is missed, it has **100 seconds** to start once the cluster is back.

---

## **4. Controlling Job History**
To prevent excessive job history logs, configure:
    ```yaml
    successfulJobsHistoryLimit: 3
    failedJobsHistoryLimit: 1
    ```
- **Keeps only the last 3 successful jobs**.
- **Keeps only 1 failed job**.

---

## **5. Example: CronJob that Cleans Up Logs**
    ```yaml
    apiVersion: batch/v1
    kind: CronJob
    metadata:
    name: log-cleanup
    spec:
    schedule: "0 2 * * *"  # Runs every day at 2 AM
    jobTemplate:
        spec:
        template:
            spec:
            containers:
                - name: cleanup-container
                image: busybox
                args:
                    - /bin/sh
                    - -c
                    - "rm -rf /var/log/*.log"
            restartPolicy: OnFailure
    ```

---

## **6. Deploy and Monitor CronJobs**
### **Deploy a CronJob**
    ```sh
    kubectl apply -f cronjob.yaml
    ```

### **Check CronJobs**
    ```sh
    kubectl get cronjob
    ```

### **Check Running Jobs**
    ```sh
    kubectl get jobs
    ```

### **Check Logs of Latest Job**
    ```sh
    kubectl logs $(kubectl get pods --selector=job-name=<job-name> --output=jsonpath={.items..metadata.name})
    ```

### **Delete a CronJob**
    ```sh
    kubectl delete cronjob my-cronjob
