> [!IMPORTANT]
> **This project is now on its own org at (https://github.com/luna-hq/luna/).**

---

[![main](https://github.com/flowerinthenight/luna/actions/workflows/main.yml/badge.svg)](https://github.com/flowerinthenight/luna/actions/workflows/main.yml)

## Overview

**Luna** is an **in-memory**, columnar (OLAP) SQL server for object storage data. Built on [DuckDB](https://duckdb.org/) and [Apache Arrow](https://arrow.apache.org/), it supports  several data sources ([S3](https://aws.amazon.com/s3/), [GCS](https://cloud.google.com/storage?hl=en), [Azure Blob Storage](https://azure.microsoft.com/en-us/products/storage/blobs), local filesystem, etc.), and various data types (CSV, JSON, Parquet, Iceberg, Excel, etc.). It's in-memory by default but it supports spilling to disk for larger-than-memory workloads.

> [!NOTE]
> - Luna's development is supported (and funded) by [Alphaus, Inc.](https://www.alphaus.cloud/en/) as it's also being used internally.
> - At the moment, Luna is tested (and expected) to run on a single machine, although support for distributed clusters is part of the goal in the (near) future.

## API specs

A Luna process maintains a single, in-memory database that can be configured through its TCP-based API which is exposed, by default, on port `7688` (can be changed through the `--api-host-port` flag). Requests use a variant of Redis' [RESP](https://redis.io/docs/latest/develop/reference/protocol-spec/) spec, specifically, the [Bulk strings](https://redis.io/docs/latest/develop/reference/protocol-spec/#bulk-strings) representation. Responses, on the other hand, consist of a stream of `RecordBatch` messages, utilizing Arrow's [IPC format](https://arrow.apache.org/docs/format/Columnar.html#format-ipc). This also applies to error messages.

#### [Request]

Requests are encoded as follows (excluding [`AUTH`](#authentication)):

```
$<length>\r\n<data>\r\n
```

- The dollar sign (`$`) as the first byte.
- One or more decimal digits (0..9) as the string's length, in bytes, as an unsigned, base-10 value.
- The `CRLF` terminator.
- The data.
- A final `CRLF`.

The `<data>` section is further broken down into the following:

- First 2 bytes - prefix as command type. It can either be `x:` for execute, or `q:` for query statements.
- Remaining bytes - the actual command, mostly in SQL form.

For example, to load CSV files from cloud storage, we will have the following requests:

```sql
-- Setup credentials for GCS access:
$79\r\nx:CREATE OR REPLACE SECRET (TYPE gcs, KEY_ID 'some-key', SECRET 'some-secret');\r\n

-- or S3 access:
$116\r\nx:CREATE OR REPLACE SECRET (
TYPE s3, PROVIDER config, KEY_ID 'some-key', SECRET 'some-secret', REGION 'us-east-1');\r\n

-- Then import some CSV files from GCS:
$138\r\nx:CREATE TABLE tmpcur AS FROM read_csv('gs://bucket/987368816909_2025-08*.csv',
header = true,
union_by_name = true,
files_to_sniff = -1);\r\n

-- or from S3:
$138\r\nx:CREATE TABLE tmpcur AS FROM read_csv('s3://bucket/987368816909_2025-08*.csv',
header = true,
union_by_name = true,
files_to_sniff = -1);\r\n

-- Describe the created table:
$18\r\nq:DESCRIBE tmpcur;\r\n

-- Query data:
$39\r\nq:SELECT uuid, date, payer FROM tmpcur;\r\n
```

#### [Response]

- Error messages will have two fields/columns: `input`, and `error`, with row values containing the input and the resulting error.

```
schema:
  fields: 2
    - input: type=utf8
    - error: type=utf8

record:
  schema:
  fields: 2
    - input: type=utf8
    - error: type=utf8
  rows: 1
  col[0][input]: ["DESCRIBE me;"]
  col[1][error]: ["Catalog Error: Table with name me does not exist!..."]

+--------------+--------------------------------------------------------+
| input        | error                                                  |
+--------------+--------------------------------------------------------+
| DESCRIBE me; | Catalog Error: Table with name me does not exist!      |
|              | Did you mean \"pg_namespace\"?\n\nLINE 1: DESCRIBE me; |
+--------------+--------------------------------------------------------+
```

- Success messages use the same format with `OK` as the value.

```
+---------------------------------------------------------------+-------+
| input                                                         | error |
+---------------------------------------------------------------+-------+
| CREATE TABLE customers AS FROM read_csv('s3://bucket/custo... | OK    |
+---------------------------------------------------------------+-------+
```

- Otherwise, the result will depend on the query result itself.

```
SELECT CustomerId, LastName, FirstName, Email FROM customers;

schema:
  fields: 4
    - CustomerId: type=utf8, nullable
    - LastName: type=utf8, nullable
    - FirstName: type=utf8, nullable
    - Email: type=utf8, nullable

record:
  schema:
  fields: 4
    - CustomerId: type=utf8, nullable
    - LastName: type=utf8, nullable
    - FirstName: type=utf8, nullable
    - Email: type=utf8, nullable
  rows: 1000
  col[0][CustomerId]: ["dE014d010c7ab0c" "2B54172c8b65eC3" ...]
  col[1][LastName]: ["Goodman" "Lane" ...]
  col[2][FirstName]: ["Andrew" "Alvin" ...]
  col[3][Email]: ["marieyates@gomez-spencer.info" "alexandra86@mccoy.com" ...]

+-----------------+----------+-----------+-------------------------------+
| CustomerId      | LastName | FirstName | Email                         |
+-----------------+----------+-----------+-------------------------------+
| dE014d010c7ab0c | Goodman  | Andrew    | marieyates@gomez-spencer.info |
| 2B54172c8b65eC3 | Lane     | Alvin     | alexandra86@mccoy.com         |
| ...             | ...      | ...       | ...                           |
+-----------------+----------+-----------+-------------------------------+
```

## Security

Luna is designed to be accessed by trusted clients inside trusted environments. Exposing Luna directly to the internet or to an environment where untrusted clients can directly access Luna's TCP port is discouraged.

#### [Authentication]

Luna provides instance level authentication by setting the `--passwd` flag. When enabled, Luna will refuse any query by unauthenticated clients. A client can authenticate itself by sending the `AUTH` command followed by the password, then the CRLF terminator.

```
AUTH <password>\r\n
```

Luna's authentication is per connection. Once authenticated, a client can continue sending commands to Luna for the duration of the connection. An invalid authentication will terminate the connection.

The `AUTH` command is sent unencrypted. It does not protect against an attacker who has enough access to the network to perform eavesdropping.

## Build and run (local)

The following guide uses [`lunactl`](https://github.com/flowerinthenight/lunactl/), a test cmdline for Luna. A [sample CSV](./luna/tests/) (copied from [sample-csv-files](https://github.com/datablist/sample-csv-files)) is also included to test filesystem-based data imports.

```sh
# Build binary.
$ cargo build

# Run on default port 7688.
$ RUST_LOG=info ./target/debug/luna

# or if password is to be required.
# $ RUST_LOG=info ./target/debug/luna --passwd K6K8bvyD2u

# Install lunactl for testing.
$ brew install flowerinthenight/tap/lunactl

# If password was enabled, add the --pass flag. For ex:
# $ lunactl -p "DESCRIBE customers;" --pass K6K8bvyD2u

# --- Sample for CSV ---

# Import local CSV to luna (newline is for readability only).
$ lunactl -x -p "CREATE TABLE customers AS FROM read_csv(
'{luna-root}/luna/tests/customers-1000.csv', header = true);"

# Describe our newly-created table.
$ lunactl -p "DESCRIBE customers;"

# Query some data.
$ lunactl -p "SELECT CustomerId, Email FROM customers LIMIT 5;"

# --- Sample for Parquet ---

# Import local parquet to luna (newline is for readability only).
$ lunactl -x -p "CREATE TABLE users AS FROM read_parquet(
'{luna-root}/luna/tests/users-1000.parquet');"

# Describe the users table.
$ lunactl -p "DESCRIBE users;"

# Query some data.
$ lunactl -p "SELECT id, email FROM users LIMIT 5;"
```

## Running on a GCP MIG

A sample cloud-init [startup script](./luna/scripts/startup-gcp-mig.sh) is provided for spinning up a [Managed Instance Group](https://cloud.google.com/compute/docs/instance-groups#managed_instance_groups) with Luna running as a systemd service.

```bash
# Create a launch template. Update --service-account with
# your own service account. You can update the --machine-type.
$ cd luna/scripts/
$ gcloud compute instance-templates create luna-tmpl \
  --machine-type e2-micro \
  --service-account=name@project.iam.gserviceaccount.com \
  --scopes=cloud-platform \
  --metadata=startup-script=''"$(cat startup-gcp-mig.sh)"''

# Create the MIG. Update {target-region} with actual value.
$ gcloud compute instance-groups managed create luna-mig \
  --template luna-tmpl --size 1 --region {target-region}

# Let's use 'https://github.com/flowerinthenight/g-ssh-cmd'
# to tail the VM logs. This is optional.
$ brew install flowerinthenight/tap/g-ssh-cmd

# Assuming your 'gcloud' cmdline is configured properly.
$ g-ssh-cmd mig luna-mig 'journalctl -f' --project {project}
```

## Running on an AWS ASG

A sample cloud-init [startup script](./luna/scripts/startup-aws-asg.sh) is provided for spinning up an [Auto Scaling Group](https://docs.aws.amazon.com/autoscaling/ec2/userguide/auto-scaling-groups.html) with Luna running as a systemd service.

```sh
# Create a launch template. ImageId here is Amazon Linux, default VPC.
# (Added newlines for readability. Might not run when copied as is.)
# Replace 'keyName' with your own keypair; update 'InstanceType'.
$ cd luna/scripts/
$ aws ec2 create-launch-template \
  --launch-template-name luna-lt \
  --version-description version1 \
  --launch-template-data '
  {
    "UserData":"'"$(cat startup-aws-asg.sh | base64 -w 0)"'",
    "ImageId":"ami-08f0737412a47a5ed",
    "InstanceType":"t2.micro",
    "KeyName":"keyName"
  }'

# Create the single-zone ASG; update {target-zone} with actual value.
$ aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name luna-asg \
  --launch-template LaunchTemplateName=luna-lt,Version='1' \
  --min-size 1 \
  --max-size 1 \
  --tags Key=Name,Value=luna-asg \
  --availability-zones {target-zone}

# Let's use [https://github.com/flowerinthenight/g-ssh-cmd]
# to tail the VM logs. This is optional.
$ brew install flowerinthenight/tap/g-ssh-cmd

# Assuming your 'aws' cmdline is configured properly.
$ g-ssh-cmd asg luna-asg 'journalctl -f' --key keyName.pem
```

## Todo

- [ ] Client SDK (and samples) for mainstream programming languages
- [ ] A basic HTTP server for HTTP1.1-style requests
- [ ] A basic gRPC server for streaming style requests
- [ ] Authentication, users, and ACLs
  - [x] Simple, server-level password auth
- [ ] Preload data import(s) on startup
- [ ] Watch data source(s) changes
- [ ] Dedicated documentation site
- [ ] Deployment guides and samples
- [x] Release pipeline
- [ ] Proper testing codes
- [ ] Additional testdata for object storage, local, and different file/db types
- [ ] Distributed/cluster support
