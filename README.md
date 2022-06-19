

### Run the DB in a container. 
Note: Nothing will be save when container shuts down. To save, set up volume mount:

`docker run -d --name my_postgres -p 54320:5432 -e POSTGRES_PASSWORD=my_password postgres:13`

### DB Setup
* `docker exec -it my_postgres psql -U postgres -c "create database my_database"`
* `docker exec -it my_postgres /bin/sh`
* `psql -U postgres`
* `\c my_database`

Docs on how to navigate session: https://tomcam.github.io/postgres/#opening-a-connection-locally

### Create stuff

```
CREATE TABLE if not exists history (
    walletId                 VARCHAR(40) NOT NULL,
    nftId                    VARCHAR(40) NOT NULL,
    txId                     VARCHAR(40) NOT NULL,
    transferIn               timestamp(3) with time zone NOT NULL,
    transferOut              timestamp(3) with time zone,
    PRIMARY KEY(walletId, nftId, txId)
);
```
#### Minting
```
INSERT INTO history VALUES ('Brian', 'BAYC', 1, to_timestamp(0));
```

```
INSERT INTO history VALUES ('Brian', 'AZUKI', 2, to_timestamp(500000));
```

```
INSERT INTO history VALUES ('Brian', 'CRYPTOPUNK', 3, to_timestamp(700000));
```

#### Transfers

```
BEGIN;

UPDATE history SET transferOut = to_timestamp(800000)
WHERE walletId = 'Brian' AND nftId = 'CRYPTOPUNK' AND transferOut IS NULL;

INSERT INTO history VALUES ('Isabella', 'CRYPTOPUNK', 4, to_timestamp(800000));

COMMIT;
``` 

```
BEGIN;

UPDATE history SET transferOut = to_timestamp(900000)
WHERE walletId = 'Brian' AND nftId = 'BAYC' AND transferOut IS NULL;

INSERT INTO history VALUES ('Isabella', 'BAYC', 5, to_timestamp(900000));

COMMIT;
``` 

```
BEGIN;

UPDATE history SET transferOut = to_timestamp(1500000)
WHERE walletId = 'Isabella' AND nftId = 'BAYC' AND transferOut IS NULL;

INSERT INTO history VALUES ('Brian', 'BAYC', 6, to_timestamp(1500000));

COMMIT;
```

Now see what we got:
```
my_database=# SELECT * from history;
 walletid |   nftid    | txid |       transferin       |      transferout
----------+------------+------+------------------------+------------------------
 Brian    | AZUKI      | 2    | 1970-01-06 18:53:20+00 |
 Brian    | CRYPTOPUNK | 3    | 1970-01-09 02:26:40+00 | 1970-01-10 06:13:20+00
 Isabella | CRYPTOPUNK | 4    | 1970-01-10 06:13:20+00 |
 Brian    | BAYC       | 1    | 1970-01-01 00:00:00+00 | 1970-01-11 10:00:00+00
 Isabella | BAYC       | 5    | 1970-01-11 10:00:00+00 | 1970-01-18 08:40:00+00
 Brian    | BAYC       | 6    | 1970-01-18 08:40:00+00 |
(6 rows)
```
### Queries
#### Get me all NFTs that Brian owns

```
my_database=# SELECT nftid FROM history
WHERE walletid = 'Brian' AND transferout IS NULL;
 nftid
-------
 AZUKI
 BAYC
(2 rows)
```


#### Get me NFTs that Brian owned at a certain time
```
my_database=# SELECT nftid, transferIn FROM history
WHERE walletid = 'Brian'
    AND transferin <= to_timestamp(700000)
    AND (transferout > to_timestamp(700000)
        OR transferout IS NULL);
   nftid    |       transferin
------------+------------------------
 AZUKI      | 1970-01-06 18:53:20+00
 BAYC       | 1970-01-01 00:00:00+00
 CRYPTOPUNK | 1970-01-09 02:26:40+00
(3 rows)
```

```
my_database=# SELECT nftid, transferIn FROM history
WHERE walletid = 'Brian'
    AND transferin <= to_timestamp(1100000)
    AND (transferout > to_timestamp(1100000)
        OR transferout IS NULL);
 nftid |       transferin
-------+------------------------
 AZUKI | 1970-01-06 18:53:20+00
(1 row)
```

```
my_database=# SELECT nftid, transferIn FROM history
WHERE walletid = 'Brian'
    AND transferin <= to_timestamp(1600000)
    AND (transferout > to_timestamp(1600000)
        OR transferout IS NULL);
 nftid |       transferin
-------+------------------------
 AZUKI | 1970-01-06 18:53:20+00
 BAYC  | 1970-01-18 08:40:00+00
```

##### Another way 
```
my_database=# SELECT *
FROM (
         SELECT DISTINCT ON (nftId) nftId, walletId, transferIn
         FROM history
         WHERE transferin <= to_timestamp(1600000)
         ORDER BY nftId, transferin DESC
     ) as groupedByWhenNftWasMostRecentlyTransferedRegardlessOfWallet
WHERE walletId = 'Brian';
 nftid | walletid |       transferin
-------+----------+------------------------
 AZUKI | Brian    | 1970-01-06 18:53:20+00
 BAYC  | Brian    | 1970-01-18 08:40:00+00
(2 rows)
```
