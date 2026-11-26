# HDFS Distributed Storage Lab

## 1. Project Overview

**Objective:** Set up a local Hadoop Distributed File System (HDFS) cluster using Docker and understand how distributed storage works.

This lab focuses on:

- Understanding HDFS architecture.
- Setting up a NameNode and multiple DataNodes.
- Creating directories in HDFS.
- Uploading files into HDFS.
- Understanding block splitting.
- Understanding block replication.
- Verifying uploaded files through the terminal and Web UI.
- Inspecting block locations and filesystem health.

### Technology Stack

| Technology | Purpose |
|---|---|
| Docker Desktop | Container environment |
| Docker Compose | Manage multiple Hadoop containers |
| Apache Hadoop 3.2.1 | Distributed storage |
| HDFS | Distributed file system |
| NameNode | Manage filesystem metadata |
| DataNode | Store actual data blocks |

---

## 2. HDFS Architecture

Our local cluster consists of three containers:

```text
                 HDFS CLIENT
              (hdfs dfs commands)
                       |
                       |
                 NameNode
                 Port: 9870
              Metadata Management
                       |
              -------------------
              |                 |
              |                 |
          DataNode 1        DataNode 2
          Port: 9864        Port: 9865
          Data Storage      Data Storage
```

### NameNode

The NameNode manages filesystem metadata, including:

- File and directory names.
- File-to-block mapping.
- Block IDs.
- DataNode locations.
- Replication information.

**Important:** The NameNode does not store the actual contents of normal user files.

### DataNode

DataNodes store the actual HDFS data blocks.

Their responsibilities include:

- Storing data blocks.
- Serving read and write requests.
- Sending heartbeats to the NameNode.
- Reporting the blocks they store.

### HDFS Client

The HDFS client is responsible for communicating with the NameNode to obtain block locations and transferring file data directly to DataNodes.

The NameNode coordinates the process but does not relay the actual file contents.

---

## 3. Docker Environment Setup

We used Docker Compose to run the Hadoop cluster.

### Containers

| Container | Role |
|---|---|
| namenode | HDFS metadata management |
| datanode1 | First storage node |
| datanode2 | Second storage node |

### Ports

| Service | Host Port | Purpose |
|---|---:|---|
| NameNode | 9870 | Web UI |
| DataNode 1 | 9864 | DataNode Web UI |
| DataNode 2 | 9865 | DataNode Web UI |

### Persistent Volumes

We configured named Docker volumes for persistent storage:

- `namenode_data`
- `datanode1_data`
- `datanode2_data`

This allows container data to persist when containers are recreated, as long as the volumes are not deleted.

### Hadoop Configuration

File: `hadoop.env`

```properties
CORE_CONF_fs_defaultFS=hdfs://namenode:9000
HDFS_CONF_dfs_replication=2
HDFS_CONF_dfs_permissions_enabled=false
HDFS_CONF_dfs_blocksize=1048576
```

Configuration explanation:

| Property | Value | Meaning |
|---|---|---|
| fs.defaultFS | hdfs://namenode:9000 | HDFS NameNode address |
| dfs.replication | 2 | Maintain two copies of each block |
| dfs.permissions.enabled | false | Disable HDFS permission checks for this lab |
| dfs.blocksize | 1048576 | Set block size to 1 MiB |

**Why did we change the block size?**

The default HDFS block size is 128 MiB. We changed it to 1 MiB so that a small 3 MiB test file would be divided into multiple blocks, making the process easier to observe.

### Start the cluster

Run these commands in Windows PowerShell from the project directory:

```powershell
docker compose up -d
```

Check running containers:

```powershell
docker ps
```

Check container status:

```powershell
docker compose ps
```

Recreate containers after changing the Hadoop configuration:

```powershell
docker compose up -d --force-recreate
```

**Important:** Avoid `docker compose down -v` during this lab because it deletes named volumes and their stored data.

---

## 4. Accessing the NameNode Container

We execute Hadoop commands from inside the NameNode container.

```powershell
docker exec -it namenode bash
```

We are now inside the Linux shell of the NameNode container.

Example:

```text
root@4b8a3a344577:/#
```

Check the configured HDFS block size:

```bash
hdfs getconf -confKey dfs.blocksize
```

Expected output:

```text
1048576
```

This confirms that the block size is 1 MiB.

---

## 5. Creating HDFS Directories

We created two directories in HDFS:

```bash
hdfs dfs -mkdir -p /data/raw
hdfs dfs -mkdir -p /data/processed
```

Directory structure:

```text
HDFS
 |
 +-- /data
      |
      +-- raw
      |
      +-- processed
```

Explanation:

- `/data/raw`: Store original, unprocessed data.
- `/data/processed`: Intended for storing transformed data later.

List HDFS directories:

```bash
hdfs dfs -ls /
```

List the data directory:

```bash
hdfs dfs -ls /data/
```

---

## 6. Creating a Sample CSV File

We created a small sample dataset inside the NameNode container.

Command:

```bash
cat > trips.csv <<'EOF'
trip_id,pickup,dropoff,fare
1,Dhaka Mirpur,Dhanmondi,250
2,Uttara,Gulshan,300
3,Banani,Mohakhali,150
4,Dhanmondi,Mirpur,200
EOF
```

Check the local file:

```bash
ls -lh trips.csv
```

**Important distinction:**

At this point, `trips.csv` existed only in the local filesystem of the NameNode container.

It was not yet stored in HDFS.

The local filesystem and HDFS are two different filesystems.

### Upload the CSV to HDFS

```bash
hdfs dfs -put trips.csv /data/raw/
```

Verify the upload:

```bash
hdfs dfs -ls /data/raw/
```

Read the CSV from HDFS:

```bash
hdfs dfs -cat /data/raw/trips.csv
```

---

## 7. Understanding HDFS Block Splitting

Next, we created a larger test file to observe how HDFS divides files into blocks.

Our configured block size:

```text
1 MiB = 1,048,576 bytes
```

Create a 3 MiB test file:

```bash
dd if=/dev/zero of=bigfile.bin bs=1M count=3
```

Explanation:

| Parameter | Meaning |
|---|---|
| `if=/dev/zero` | Generate zero-filled input |
| `of=bigfile.bin` | Output filename |
| `bs=1M` | Read/write in 1 MiB chunks |
| `count=3` | Write three chunks |

Check the file:

```bash
ls -lh bigfile.bin
```

Expected:

```text
-rw-r--r-- 1 root root 3.0M ... bigfile.bin
```

### Upload the file

```bash
hdfs dfs -put bigfile.bin /data/raw/
```

Now HDFS divides the 3 MiB file into three logical blocks:

```text
Original File: bigfile.bin (3 MiB)

       HDFS Block Splitting
               |
       -------------------
       |        |        |
       v        v        v
    Block 1  Block 2  Block 3
     1 MiB    1 MiB    1 MiB
```

---

## 8. Understanding Block Replication

Our replication factor is configured as:

```properties
HDFS_CONF_dfs_replication=2
```

This means HDFS maintains two copies of each logical block.

For our 3 MiB file:

```text
                 bigfile.bin
                     |
            -------------------
            |        |        |
            v        v        v
          Block 1  Block 2  Block 3
            |        |        |
         -------- -------- --------
         |      | |      | |      |
         v      v v      v v      v
        DN1    DN2 DN1   DN2 DN1   DN2
```

Storage result:

| Metric | Value |
|---|---:|
| Original file size | 3 MiB |
| Logical blocks | 3 |
| Replication factor | 2 |
| Total physical block copies | 6 |
| Approximate replicated storage | 6 MiB |

Each DataNode stores a copy of all three blocks in our two-DataNode cluster.

**Remember:** Replication does not mean dividing a block into two halves. Each replica contains the complete block.

---

## 9. Verifying HDFS Data Through the Terminal

### A. List uploaded files

```bash
hdfs dfs -ls /data/raw/
```

Our output:

```text
Found 2 items

-rw-r--r-- 2 root supergroup 3145728 ... /data/raw/bigfile.bin

-rw-r--r-- 2 root supergroup     243 ... /data/raw/trips.csv
```

The output shows that both files exist in HDFS.

The `2` in the permissions/replication column represents the replication factor for these files.

### B. Read the CSV file

```bash
hdfs dfs -cat /data/raw/trips.csv
```

This reads the file stored in HDFS, not the local container copy.

### C. Check filesystem health and block locations

```bash
hdfs fsck /data/raw/bigfile.bin -files -blocks -locations
```

Important output:

```text
/data/raw/bigfile.bin 3145728 bytes,
replicated: replication=2, 3 block(s): OK
```

Block details:

```text
Block 1: len=1048576 Live_repl=2

Block 2: len=1048576 Live_repl=2

Block 3: len=1048576 Live_repl=2
```

The output also showed two DataNode locations for each block.

Final status:

```text
Status: HEALTHY

Total blocks (validated): 3
Minimally replicated blocks: 3
Under-replicated blocks: 0
Missing blocks: 0
Corrupt blocks: 0
```

This confirms that:

- HDFS created three blocks.
- Every block has two replicas.
- Both DataNodes are storing the replicas.
- No blocks are missing or corrupt.

### D. Check DataNode status

```bash
hdfs dfsadmin -report
```

This command provides:

- Configured storage capacity.
- DFS used and remaining capacity.
- Number of live DataNodes.
- Number of dead DataNodes.
- Under-replicated blocks.
- Missing blocks.

We verified that our cluster had two live DataNodes.

---

## 10. Verifying Through the Web UI

We also explored the Hadoop NameNode Web UI.

### NameNode Dashboard

URL:

http://localhost:9870

The dashboard provides cluster information.

Useful sections:

| Section | Purpose |
|---|---|
| Overview | Cluster status and storage statistics |
| Datanodes | View live and dead DataNodes |
| Utilities | Access filesystem browsing |
| Browse the file system | Explore HDFS directories and files |

### Browse uploaded files

Follow these steps:

1. Open `http://localhost:9870`.
2. Find Utilities.
3. Select Browse the file system.
4. Open `/data`.
5. Open `/raw`.

You should find:

```text
/data/raw/
    |
    +-- bigfile.bin
    |
    +-- trips.csv
```

### DataNode Web UI

DataNode 1:

http://localhost:9864

DataNode 2:

http://localhost:9865

These interfaces provide information about individual DataNodes.

**Note:** The NameNode UI provides a convenient view of the filesystem and cluster. For detailed block replica locations and integrity checks, we used the `hdfs fsck` command.

---

## 11. Important HDFS Commands Reference

### Docker commands

| Command | Purpose |
|---|---|
| `docker compose up -d` | Start the cluster |
| `docker ps` | List running containers |
| `docker compose ps` | Check Compose services |
| `docker exec -it namenode bash` | Enter NameNode container |
| `docker logs namenode --tail 50` | Inspect NameNode logs |
| `docker stop datanode1` | Stop DataNode 1 |
| `docker start datanode1` | Start DataNode 1 |

### HDFS commands

| Command | Purpose |
|---|---|
| `hdfs dfs -mkdir -p /path` | Create HDFS directory |
| `hdfs dfs -ls /path` | List HDFS files |
| `hdfs dfs -put file /path` | Upload local file to HDFS |
| `hdfs dfs -get /path/file .` | Download from HDFS |
| `hdfs dfs -cat /path/file` | Read a file |
| `hdfs dfs -rm /path/file` | Delete an HDFS file |
| `hdfs dfs -du -h /path` | Check HDFS directory size |
| `hdfs dfsadmin -report` | View cluster and DataNode status |
| `hdfs fsck /path -files -blocks -locations` | Inspect blocks, replicas and health |
| `hdfs getconf -confKey dfs.blocksize` | Check configured block size |

---

## 12. Problems and Observations

### NotReplicatedYetException

During the first upload attempt of `bigfile.bin`, we encountered:

```text
NotReplicatedYetException:
Not replicated yet
```

Hadoop retried the operation.

We subsequently checked:

```bash
hdfs dfs -ls /data/raw/
```

The file existed successfully.

We then ran:

```bash
hdfs fsck /data/raw/bigfile.bin -files -blocks -locations
```

The result was:

```text
Status: HEALTHY
```

All three blocks had two live replicas.

**Learning:** A temporary error during an HDFS write does not necessarily mean the final upload failed. Always verify the final file and block health.

We did not establish the exact root cause of the temporary exception.

---

## 13. What I Learned

After completing this lab, I understand:

1. HDFS is a distributed filesystem designed to store large datasets.
2. NameNode manages metadata, not normal file contents.
3. DataNodes store actual data blocks.
4. The HDFS client communicates with the NameNode to obtain block locations.
5. Files are divided into blocks according to the configured block size.
6. HDFS replication maintains multiple copies of blocks.
7. `hdfs dfs -put` uploads local files into HDFS.
8. `hdfs fsck` helps verify block health and replica locations.
9. The NameNode Web UI provides a visual way to inspect the cluster and browse HDFS files.
10. Docker Compose allows us to simulate a distributed Hadoop cluster locally.

---

## 14. Current Lab Status

| Task | Status |
|---|---|
| Docker Hadoop cluster setup | Completed |
| NameNode setup | Completed |
| Two DataNodes setup | Completed |
| HDFS directory creation | Completed |
| CSV file creation and upload | Completed |
| Block size configuration | Completed |
| 3 MiB test file creation | Completed |
| Block splitting verification | Completed |
| Replication verification | Completed |
| FSCK health verification | Completed |
| NameNode Web UI exploration | Completed |
| DataNode failure and recovery experiment | Not yet completed |
| HDFS data ingestion pipeline | Not yet completed |

---

## 15. Next Learning Steps

The next planned experiments are:

1. Simulate DataNode failure and observe HDFS behavior.
2. Restore the DataNode and observe replica recovery.
3. Understand HDFS data ingestion using Python.
4. Explore how large datasets are organized in a data lake.
5. Move toward distributed data processing using Apache Spark.

**Final takeaway:** This lab introduced the fundamentals of distributed storage. The next goal is to use this infrastructure to solve actual data engineering problems rather than just execute Hadoop commands.