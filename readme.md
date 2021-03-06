# Fast Stellar Core Catch Up

CI status: [![<CircleCI>](https://circleci.com/gh/Lobstrco/stellar-core-parallel-catchup-py.svg?style=svg)](https://app.circleci.com/github/Lobstrco/stellar-core-parallel-catchup-py/pipelines)


## Description

Starting a [full stellar core validator](https://www.stellar.org/developers/stellar-core/software/admin.html) from scratch takes a while, as the node needs to download and process a lot of data during the "Catch up" phase.

The catching up phase usually takes more than a month.  
This script helps to make the process significantly shorter, as it allows to perform the catch up of multiple ledgers blocks in parallel.

On a powerful server the full catch up using this Python script can be done in less than a day.

Another implementation of the parallel catch up idea is a [shell script by SatoshiPay](https://github.com/satoshipay/stellar-core-parallel-catchup). 


## Preparation

Due to the parallel nature of the process, the sync time can be significantly decreased with bigger instance type and more CPU and RAM.

So it's recommended to scale up a server instance until the catch up process is complete.
On the instance similar to c5.12xlarge the full catch up can be completed within 24 hours (please feel free to share your specs and results).

The process also requires a lot of disk space, so we recommend to connect a temporary SSD disk around 2TB in size.
This temporary disk will be used to store copies of ledger chunks, merging them together in the database.

This script is also designed to publish history archives to a cloud archive like S3.


## Instructions


1. Make sure stellar core is not running:
```bash
sudo service stellar-core stop
```

2. Tweak default settings of postgresql for high performance:
```text
sudo nano /etc/postgresql/10/main/postgresql.conf
```

Change:
```text
max_locks_per_transaction 5000
max_pred_locks_per_transaction 5000
max_connections 1500
shared_buffers 10GB # (allocate about 25% of total RAM)
```

3. Grant superuser permissions to the user:
```bash
sudo -i -u postgres
psql -c "alter user <db_user> with superuser;"
```

4. Connect a large disk to the server (in early 2020 about 2TB would be enough for history and databases)
```bash
lsblk
# format & create partition if required

sudo mkdir /mnt/storage
sudo mount /dev/sdb1 /mnt/storage
```

5. Move postgresql database to the temporary disk:
```bash
sudo service postgresql stop
sudo mv sudo mv /var/lib/postgresql/10/main /mnt/storage/
sudo ln -s /mnt/storage/main /var/lib/postgresql/10/main
sudo service postgresql start
```

6. Clone repository:
```bash
git clone https://github.com/Lobstrco/stellar-core-parallel-catchup-py.git /mnt/storage/core-parallel-catchup
sudo chown stellar /mnt/storage/core-parallel-catchup
```

7. Login as stellar user:
```bash
sudo -i -u stellar
cd src
```

8. Install python script requirements from provided requirements.txt
```bash
pip install -r requirements.txt 
```

9. Generate your secret key - it will be needed for next step:
```bash
stellar-core gen-seed
```

10. Initialize folders structure, daemonize workers monitor and merge process. This will take a while:
```bash
export DB_HOST=localhost
export DB_PORT=5432
export DB_USER=<db_user>
export DB_PASSWORD=<db_password>
export NODE_SECRET_KEY=<secret_key>
python cli.py initialize
nohup python cli.py monitor > monitor.log &
nohup python cli.py merge > merge.log &
```

11. Great! You're almost done. Now rename database:
```sql
ALTER DATABASE "catchup-stellar-result" RENAME TO "stellar-core";
```

12. Move folders to their real destinations:
```bash
sudo mv result/data/buckets /var/lib/stellar/
sudo service postgresql stop
sudo rm /var/lib/postgresql/10/main
sudo mv /mnt/storage/main /var/lib/postgresql/10/
sudo service postgresql start
```

13. You're ready to start stellar core node
```bash
sudo service stellar-core start
```

14. (Optional) If you're not going to use stellar horizon, remove HORIZON cursor from core database to re-enable db maintenance
```bash
curl localhost:11626/dropcursor?id=HORIZON
``` 

15. Publish history archives to the cloud using [stellar-archivist](https://github.com/stellar/go/tree/master/tools/stellar-archivist):
```text
sudo apt-get install stellar-archivist
sudo service stellar-core stop

sudo -i -u stellar
/bin/bash
export $(cat /etc/stellar/stellar-core | xargs) && stellar-core new-hist s3 --conf /etc/stellar/stellar-core.cfg
exit

sudo service stellar-core start

sudo -i -u stellar
/bin/bash
export $(cat /etc/stellar/stellar-core | xargs) && nohup stellar-archivist repair file:///mnt/storage/parallel-catchup/result/vs/ s3://<aws_s3_history_bucket_name> --s3region=$AWS_DEFAULT_REGION > archivist-repair.out &
exit
```

16. It's all done! Now you can detach the temporary disk from the instance, and scale it down to the usual size.
