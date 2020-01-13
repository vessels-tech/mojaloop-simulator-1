# Testing Redis Disk Usage

with redis.conf settings:

```
maxmemory 2mb
maxmemory-policy allkeys-lru
```


## Up And Running:
```bash
cd ./src
docker-compose -f docker-compose-sim-dfsp.yml up
```

## Test Requests

```bash
curl -X POST \
  "http://localhost:3003/scenarios" \
  -H 'Content-Type: application/json' \
  -d '[
    {
        "name": "scenario3",
        "operation": "postTransfers",
        "body": {
            "from": {
                "displayName": "Frank Bush",
                "idType": "MSISDN",
                "idValue": "123456789"
            },
            "to": {
                "idType": "MSISDN",
                "idValue": "987654321"
            },
            "amountType": "SEND",
            "currency": "USD",
            "amount": "100",
            "transactionType": "TRANSFER",
            "note": "test payment",
            "homeTransactionId": "AAA"
        }
    }
]'
```


## Ad-hoc Load Tests
```bash
artillery run redis_test.yml
```


## Findings

1. This log from redis instances:
```
redis-sim_1              | 1:M 19 Dec 2019 11:56:23.119 * 100 changes in 300 seconds. Saving...
redis-dfsp_1             | 1:M 19 Dec 2019 11:56:23.115 * 100 changes in 300 seconds. Saving...
redis-sim_1              | 1:M 19 Dec 2019 11:56:23.127 * Background saving started by pid 13
redis-dfsp_1             | 1:M 19 Dec 2019 11:56:23.128 * Background saving started by pid 16
redis-sim_1              | 13:C 19 Dec 2019 11:56:23.166 * DB saved on disk
redis-dfsp_1             | 16:C 19 Dec 2019 11:56:23.166 * DB saved on disk
redis-sim_1              | 13:C 19 Dec 2019 11:56:23.167 * RDB: 0 MB of memory used by copy-on-write
redis-dfsp_1             | 16:C 19 Dec 2019 11:56:23.168 * RDB: 0 MB of memory used by copy-on-write
redis-sim_1              | 1:M 19 Dec 2019 11:56:23.236 * Background saving terminated with success
redis-dfsp_1             | 1:M 19 Dec 2019 11:56:23.235 * Background saving terminated with success
```

implies that if there are many changes to the cache, then it will be written to the `/data/dump.rdb` file.

2. Check this file size with:
```bash
$ docker exec -it src_redis-sim_1 sh -c "ls -lah"
total 1108
drwxr-xr-x    2 redis    redis       4.0K Dec 19 11:56 .
drwxr-xr-x    1 root     root        4.0K Dec 19 11:00 ..
-rw-r--r--    1 redis    redis       1.1M Dec 19 11:56 dump.rdb
```


3. After another 300 requests:
```
-rw-r--r--    1 redis    redis       1.2M Dec 19 12:17 dump.rdb
```



Summary:
- every 600 seconds, a new dump file is saved
- After 300 new requests, this dump file grew 0.1 MB
  - so we can estimate 1 request = 0.000333333 MB
  - and 1,000,000 Requests ~= 300 MB