

A **Kubernetes Job** ensures that a pod runs **to completion**. Unlike a **CronJob**, which schedules recurring tasks, a **Job** runs once and exits when complete. Jobs are useful for **batch processing, backups, database migrations, and data processing tasks**.

---

## **1. Job API Specification**
A Kubernetes `Job` is defined using `batch/v1` API and has several parameters:

        ```yaml
        apiVersion: batch/v1
        kind: Job
        metadata:
        name: my-job
        labels:
            app: job-example
        spec:
            parallelism: 2  # (1) Run 2 pods in parallel
            completions: 4  # (2) Complete 4 successful pods
            backoffLimit: 3 # (3) Retry failed jobs 3 times
            activeDeadlineSeconds: 300  # (4) Time limit before job termination
            ttlSecondsAfterFinished: 100  # (5) Automatically delete after 100 sec
            template:
                spec:
                    restartPolicy: OnFailure  # (6) Restart failed pods only
                    containers:
                        - name: my-container
                          image: busybox
                          args:
                            - /bin/sh
                            - -c
                            - "echo Hello from Kubernetes Job && sleep 30"
                ```

---

## **2. Explanation of All Job Attributes**
        | Attribute | Description | Example |
        |-----------|------------|---------|
        | **`parallelism`** | Defines the number of pods that can run in parallel. | `2` |
        | **`completions`** | Number of pods that must successfully complete before job is finished. | `4` |
        | **`backoffLimit`** | Number of retry attempts before marking job as failed. | `3` |
        | **`activeDeadlineSeconds`** | Total execution time before the job is forcibly stopped. | `300` (5 min) |
        | **`ttlSecondsAfterFinished`** | Deletes completed job after the specified time. | `100` |
        | **`restartPolicy`** | Defines restart behavior. Options: `Never`, `OnFailure`. | `OnFailure` |
        | **`containers`** | Defines the container and command to execute in the pod. | See YAML example above |

        ---

## **3. Job Completion Strategies**
### **A. Single Pod Job (Default)**
A single pod runs once and completes successfully.

    ```yaml
    spec:
        completions: 1
        parallelism: 1
    ```
✔️ **Use Case:** Running a short-lived **database migration** or **data processing** job.

---

### **B. Parallel Job (Fixed Completion)**
Runs multiple pods in parallel, completing after a total number of successful pods.

    ```yaml
    spec:
        parallelism: 2
        completions: 4
    ```
✔️ **Use Case:** **Data processing** where multiple jobs should complete before moving forward.

---

### **C. Parallel Job (Work Queue)**
- Runs an **infinite number** of pods to process a queue until externally stopped.
- Uses **parallelism** but without `completions`.

    ```yaml
    spec:
        parallelism: 3
        completions: 1  # Omit this line to process indefinitely
    ```
✔️ **Use Case:** **Processing Kafka events or job queues**.

---

### **D. Backoff Limit (Retry Failed Jobs)**
Defines the number of retries before a job is marked as failed.

        ```yaml
        spec:
         backoffLimit: 5
        ```
✔️ **Use Case:** **Retries network-related jobs** in case of temporary failures.

---

## **4. Example: Kubernetes Job for Backup**
This example **backs up a MySQL database** to an **AWS S3 bucket**.

### **Backup Job YAML**
        ```yaml
        apiVersion: batch/v1
        kind: Job
        metadata:
          name: mysql-backup-job
        spec:
          template:
             spec:
               restartPolicy: OnFailure
               containers:
                - name: mysql-backup
                  image: mysql:latest
                  command:
                    - /bin/sh
                    - -c
                    - "mysqldump -h mysql-host -u root -p$MYSQL_PASSWORD mydb | gzip > /backup/mydb.sql.gz && aws s3 cp /backup/mydb.sql.gz s3://my-backup-bucket/"
                   env:
                     - name: MYSQL_PASSWORD
                       valueFrom:
                         secretKeyRef:
                         name: mysql-secret
                         key: password
                   volumeMounts:
                    - name: backup-storage
                      mountPath: "/backup"
               volumes:
                - name: backup-storage
                  emptyDir: {}
        ```

### **Explanation:**
- **Takes a MySQL dump** (`mysqldump`).
- **Compresses** it (`gzip`).
- **Uploads** to AWS S3 (`aws s3 cp`).
- **Uses environment variables** to securely pass credentials.
- **`emptyDir` volume** stores the backup file temporarily.

✔️ **Use Case:** Scheduled **database backups** to cloud storage.

---

## **5. Monitoring Jobs**
### **Check Running Jobs**
    ```sh
    kubectl get jobs
    ```

### **Check Completed Jobs**
    ```sh
    kubectl get jobs --field-selector=status.successful=1
    ```

### **Check Job Details**
    ```sh
    kubectl describe job my-job
    ```

### **View Job Logs**
    ```sh
    kubectl logs $(kubectl get pods --selector=job-name=my-job -o jsonpath='{.items[0].metadata.name}')
    ```

### **Delete a Job**
    ```sh
    kubectl delete job my-job
    ```

---

## **6. Automatic Cleanup of Finished Jobs**
### **Option 1: TTL (Automatic Deletion)**
Use `ttlSecondsAfterFinished` to delete jobs after completion.
    ```yaml
    spec:
    ttlSecondsAfterFinished: 300  # Deletes job after 5 minutes
    ```

### **Option 2: Manual Cleanup**
    ```sh
    kubectl delete job --all
    ```

---

## **7. CronJob vs. Job: Key Differences**
    | Feature | **Job** | **CronJob** |
    |---------|--------|------------|
    | **Execution** | Runs **once** | Runs **repeatedly** |
    | **Trigger** | **Manual** or **on event** | **Scheduled** (cron syntax) |
    | **Use Case** | **One-time tasks** (e.g., backup, migration) | **Recurring tasks** (e.g., daily backups) |

✔️ **Use `Job` for:** **Data processing, one-time backups, database migrations.**  
✔️ **Use `CronJob` for:** **Recurring tasks like daily backups or log rotation.**

