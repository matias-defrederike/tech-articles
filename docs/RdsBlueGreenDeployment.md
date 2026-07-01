---
layout: default
title: "AWS Relational Database Service: Blue Green Deployment"
parent: Articles
permalink: rds-blue-green-deployment
---

<p align="center">
<img src="{{ site.baseurl }}/assets/images/rds-blue-green-cover.avif" alt="Cover RDS Blue Green Article" width="800" style="border-radius: 8px;"/>
</p>

How many times have you postponed a database upgrade out of fear of what could go wrong? If the answer is "more than once," this article is for you.

---

Databases are usually where the most critical parts of an application reside. Any careless change, from dropping an index to a minor version upgrade, can cause service degradation or downtime.

I've had to execute this type of procedure on highly business-critical databases before. It wasn't a simple tweak: we were migrating from MySQL 5.7 to MySQL 8.0 in a production Aurora MySQL environment with zero tolerance for downtime.

Fortunately, if you use AWS, there is a robust solution called [RDS Blue/Green Deployments](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments.html) that allows you to perform this kind of operation with near-zero downtime. It creates a mirrored environment of your database so you can test, validate, and, when you are ready, safely perform the switchover.

It is worth noting: I initially used it on Aurora MySQL, but the feature is also available for MySQL Community.

## What is RDS Blue/Green Deployment? 🟦🟩

The idea is simple: you keep two environments running in parallel.

- **Blue**: the current environment, in production, receiving real traffic.
- **Green**: the new environment, with the applied changes (version upgrade, configuration changes, etc.), not yet receiving write traffic.

Once you are confident that the Green environment is healthy, you perform the **switchover**. AWS takes care of redirecting write operations to the new database in an automated and safe manner.

## Prerequisites and limitations ⚠️

Before creating the Green environment, ensure your database is ready:

- **Binary log (binlog) enabled** (`binlog_format = ROW`): This must be done at the Parameter Group level and will require a reboot, so if your database is critical, this must be taken into consideration. To verify that your database has Binlog active, run the command below.

    ```shell
    SHOW VARIABLES LIKE 'binlog_format';

    # Return example
    MySQL [matias]> SHOW VARIABLES LIKE 'binlog_format';
    +---------------+-------+
    | Variable_name | Value |
    +---------------+-------+
    | binlog_format | ROW   |
    +---------------+-------+
    1 row in set (0.006 sec)
    ```

- **Automated backups enabled**: These are the backups that need to be activated at the cluster level.
- **No active cross-region replicas**: This limitation does not apply to Aurora Global Cluster replicas, only to conventional cross-region replicas.

## Hands-On 🛠️

### 1. Blue environment (current production)

First step: the Aurora MySQL in production.

<p align="center">
<img src="{{ site.baseurl }}/assets/images/rds-blue-green-1.avif" alt="Aurora MySQL - Blue Environment" width="800" style="border-radius: 8px;"/>
</p>

### 2. Create the Green environment

With the prerequisites met, create the Green environment directly through the AWS console or via CLI. At this point, AWS provisions a copy of your database and starts continuous replication from the Blue environment.

<p align="center">
<img src="{{ site.baseurl }}/assets/images/rds-blue-green-2.avif" alt="Creating the Green environment" width="800" style="border-radius: 8px;"/>
</p>

### 3. Apply the upgrade to the Green environment

With the Green environment created, apply the version upgrade. This step also serves as validation: if there are inconsistencies or blockers that would prevent an in-place upgrade, they will be flagged here, before any impact on production.

<p align="center">
<img src="{{ site.baseurl }}/assets/images/rds-blue-green-3.avif" alt="Upgrade env Green" width="800" style="border-radius: 8px;"/>
</p>

### 4. Validate the Green environment

Point your applications (or testing tools) to the Green endpoint and validate its behavior. Since the Green environment is not yet receiving production writes, you can test safely.

### 5. Switchover

When you are confident, execute the switchover. AWS ensures that all pending transactions on the Blue environment are replicated to the Green environment. The downtime window is minimized as much as possible.

<p align="center">
<img src="{{ site.baseurl }}/assets/images/rds-blue-green-4.avif" alt="RDS Blue-Green" width="800" style="border-radius: 8px;"/>
</p>

### 6. After the Switchover

Next comes the clean-up process. You need to delete the "blue-green" resource that sits above the two clusters and then delete the old cluster. The old cluster receives an `-old` tag, making it easy to identify.

## Monitoring the Switchover with Python 🐍

To track the exact moment of the switchover — and measure the real impact on the database response time — I created a lightweight Python script that queries the database in a loop and logs the `hostname` and `server_id` returned by MySQL. When the switchover occurs, these values change, and the script automatically logs the event.

```python
# pip install mysql-connector-python python-dotenv

import mysql.connector
import logging
from dotenv import load_dotenv
import os
import time

load_dotenv()

logging.basicConfig(
    filename='db_connection.log',
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

def connect_and_query():
    db_config = {
        'host': os.getenv('DB_HOST'),
        'port': int(os.getenv('DB_PORT', 3306)),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME'),
        'connection_timeout': 5,
    }

    last_server_id = None

    try:
        while True:
            time.sleep(1)
            conn = None
            cursor = None

            try:
                conn = mysql.connector.connect(**db_config)
                cursor = conn.cursor()

                cursor.execute("SELECT NOW(), @@hostname, @@server_id;")
                now, hostname, server_id = cursor.fetchone()

                if server_id != last_server_id:
                    msg = f"[SWITCHOVER DETECTED] hostname={hostname}, server_id={server_id}"
                    print(msg)
                    logging.warning(msg)
                    last_server_id = server_id

                msg = f"time={now} | hostname={hostname} | server_id={server_id}"
                print(msg)
                logging.info(msg)

            except mysql.connector.Error as err:
                logging.error(f"Database error: {err}")
                print(f"Database error: {err}")

            except Exception as e:
                logging.error(f"Unexpected error: {e}")
                print(f"Unexpected error: {e}")

            finally:
                if cursor:
                    cursor.close()
                if conn:
                    conn.close()

    except KeyboardInterrupt:
        print("\nExecution interrupted by user.")
        logging.info("Script terminated by user.")

if __name__ == "__main__":
    connect_and_query()
```

The required environment variables are:

```env
DB_HOST=your-rds-endpoint
DB_PORT=3306
DB_USER=your-user
DB_PASSWORD=your-password
DB_NAME=your-database
```

The script uses `@@hostname` and `@@server_id`, native MySQL variables that identify the instance where the connection is established. During the switchover, these values change, and the log registers a `[SWITCHOVER DETECTED]` event with the exact timestamp.

## Final Thoughts 💡

RDS Blue/Green Deployment is a powerful tool for upgrades with near-zero downtime in critical environments. Like any solution, it has its limitations, and I've run into a few myself. Evaluate if it fits your scenario; the goal of this article is precisely to present an option that has actually helped me in practice.

But tell me: have you ever been in a tough spot during a database upgrade?

## References 📚

- [Creating an RDS Blue/Green Deployment](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-creating.html)
- [Limitations of RDS Blue/Green Deployments](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-considerations.html)
- [What is RDS Blue/Green Deployments?](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments.html)
