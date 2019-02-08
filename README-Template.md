# BigQuery Encryption and Crypto Delete

This document describes the encryption, decryption, and keyset-related functions that BigQuery supports. 

BigQuery keeps your data safe by using encryption at rest. BigQuery also provides support for customer managed encryption keys (CMEK), which enables you to encrypt tables using specific encryption keys. In some cases, however, you may want to encrypt individual values within a table. For example:

You want to keep data for all of your own customers in a common table, and you want to encrypt each of your customers’ data using a different key.
You have data spread across multiple tables that you want to be able to “crypto-delete”. Crypto-deletion, or crypto-shredding, is the process of deleting an encryption key to make data encrypted using that key unreadable.


## Getting Started

The following queries were run on a customers dataset in BiqQuery. The schema for the table is below:
```
[
  {
    "mode": "NULLABLE", 
    "name": "c_cust_key", 
    "type": "STRING"
  }, 
  {
    "mode": "NULLABLE", 
    "name": "c_email", 
    "type": "STRING"
  }, 
  {
    "mode": "NULLABLE", 
    "name": "c_name", 
    "type": "STRING"
  }, 
  {
    "mode": "NULLABLE", 
    "name": "c_phone", 
    "type": "STRING"
  }, 
  {
    "mode": "NULLABLE", 
    "name": "c_mktsegment", 
    "type": "STRING"
  }, 
  {
    "mode": "NULLABLE", 
    "name": "c_address", 
    "type": "STRING"
  }, 
  {
    "mode": "NULLABLE", 
    "name": "c_city", 
    "type": "STRING"
  }, 
  {
    "mode": "NULLABLE", 
    "name": "c_nation", 
    "type": "STRING"
  }, 
  {
    "mode": "NULLABLE", 
    "name": "c_region", 
    "type": "STRING"
  }
]
```

| Table         | Table Size    | No. of Rows|
| ------------- |:-------------:| ----------:|
| Customers     | 1.28 GB       | 10,100,000 |
| Customers_keys| 112.81 KB     | 1000       |
| Customers_enc | 2.42 GC       | 10,100,000 |

Note: Customers_keys and customers_enc tables are created using the following scripts.

### Creating keysets

The following query creates a keyset in the Customers_keys table for each c_cust_id in the customers table, which can subsequently be used to encrypt data. Each keyset contains a single encryption key with randomly-generated key data.

```
CREATE TABLE
  `pso-ox-data-pocs.loading_dataset.customers_keys` AS (
  SELECT
    c_cust_key,
    KEYS.NEW_KEYSET('AEAD_AES_GCM_256') AS keyset
  FROM (
    SELECT
      DISTINCT(c_cust_key)
    FROM
      `pso-ox-data-pocs.loading_dataset.customers` ) )
```

### Using keysets to encrypt data

The following query uses the keysets in the Customers_keys table to encrypt data in the customers table, storing the results in the customers_enc table.

```
CREATE TABLE
  `pso-ox-data-pocs.loading_dataset.customers_enc` AS (
  SELECT
    pcd.c_cust_key,
    AEAD.ENCRYPT( (
      SELECT
        keyset
      FROM
        `pso-ox-data-pocs.loading_dataset.customers_keys` AS ck
      WHERE
        ck.c_cust_key = pcd.c_cust_key),
      pcd.c_email,
      CAST(pcd.c_cust_key AS STRING) ) AS encrypted_email,
    AEAD.ENCRYPT( (
      SELECT
        keyset
      FROM
        `pso-ox-data-pocs.loading_dataset.customers_keys` AS ck
      WHERE
        ck.c_cust_key = pcd.c_cust_key),
      pcd.c_name,
      CAST(pcd.c_cust_key AS STRING) ) AS encrypted_name,
    AEAD.ENCRYPT( (
      SELECT
        keyset
      FROM
        `pso-ox-data-pocs.loading_dataset.customers_keys` AS ck
      WHERE
        ck.c_cust_key = pcd.c_cust_key),
      pcd.c_phone,
      CAST(pcd.c_cust_key AS STRING) ) AS encrypted_phone,
    pcd.c_mktsegment,
    AEAD.ENCRYPT( (
      SELECT
        keyset
      FROM
        `pso-ox-data-pocs.loading_dataset.customers_keys` AS ck
      WHERE
        ck.c_cust_key = pcd.c_cust_key),
      pcd.c_address,
      CAST(pcd.c_cust_key AS STRING) ) AS encrypted_address,
    c_city,
    c_nation,
    c_region
  FROM
    `pso-ox-data-pocs.loading_dataset.customers` AS pcd )
```

### Using keysets to decrypt data

The following query uses the keysets in the Customers_keys table to decrypt data in the customers_enc table.

```
SELECT
  ecd.c_cust_key,
  AEAD.DECRYPT_STRING(
    (SELECT keyset
     FROM `pso-ox-data-pocs.loading_dataset.customers_keys` AS ck
     WHERE ck.c_cust_key = ecd.c_cust_key),
    ecd.encrypted_email,
    CAST(ecd.c_cust_key AS STRING)
  ) AS decrypted_email
FROM `pso-ox-data-pocs.loading_dataset.customers_enc` AS ecd;
```


### Crypto-deletion

Crypto-deletion is the process of rendering data unrecoverable by deleting the key used to encrypt it. The following statement performs crypto-deletion for a list of c_cust_keys by deleting the keysets for each of them:


```
DELETE
  `pso-ox-data-pocs.loading_dataset.customers_keys`
WHERE
  c_cust_key IN ('0033',
    '0171',
    '0219');
```

```
SELECT
  ecd.c_cust_key,
  AEAD.DECRYPT_STRING( (
    SELECT
      keyset
    FROM
      `pso-ox-data-pocs.loading_dataset.customers_keys` AS ck
    WHERE
      ck.c_cust_key = ecd.c_cust_key),
    ecd.encrypted_email,
    CAST(ecd.c_cust_key AS STRING) ) AS decrypted_email
FROM
  `pso-ox-data-pocs.loading_dataset.customers_enc` AS ecd
WHERE
  c_cust_key IN ('0033',
    '0171',
    '0219');
```

### Using an encrypted column in the WHERE clause

The following statements implement an encrypted column in the WHERE clause of the query.

```
SELECT
  c_name,
  c_cust_key,
  c_email,
  c_phone,
  c_mktsegment,
  c_address
FROM
  `pso-ox-data-pocs.loading_dataset.customers`
WHERE
  c_name IN ('Sarah Wagner',
    'Brittany Rangel',
    'Darren Taylor')
ORDER BY
  c_name
```
Elapsed time: 11.202 secs.
Slot time consumed: 25.606 secs.
Bytes shuffled: 0 B.
Bytes spilled to disk: 0 B.


```
SELECT
  AEAD.DECRYPT_STRING( (
    SELECT
      keyset
    FROM
      `pso-ox-data-pocs.loading_dataset.customers_keys` AS ck
    WHERE
      ck.c_cust_key = ecd.c_cust_key),
    ecd.encrypted_name,
    CAST(ecd.c_cust_key AS STRING) ) AS decrypted_name,
  c_cust_key,
  (
  SELECT
    keyset
  FROM
    `pso-ox-data-pocs.loading_dataset.customers_keys` AS ck
  WHERE
    ck.c_cust_key = ecd.c_cust_key),
  ecd.encrypted_email,
  CAST(ecd.c_cust_key AS STRING) AS decrypted_email,
  (
  SELECT
    keyset
  FROM
    `pso-ox-data-pocs.loading_dataset.customers_keys` AS ck
  WHERE
    ck.c_cust_key = ecd.c_cust_key),
  ecd.encrypted_phone,
  CAST(ecd.c_cust_key AS STRING) AS decrypted_phone,
  c_mktsegment,
  (
  SELECT
    keyset
  FROM
    `pso-ox-data-pocs.loading_dataset.customers_keys` AS ck
  WHERE
    ck.c_cust_key = ecd.c_cust_key),
  ecd.encrypted_address,
  CAST(ecd.c_cust_key AS STRING) AS decrypted_address
FROM
  `pso-ox-data-pocs.loading_dataset.customers_enc` AS ecd
WHERE
  AEAD.DECRYPT_STRING( (
    SELECT
      keyset
    FROM
      `pso-ox-data-pocs.loading_dataset.customers_keys` AS ck
    WHERE
      ck.c_cust_key = ecd.c_cust_key),
    ecd.encrypted_name,
    CAST(ecd.c_cust_key AS STRING) ) IN ('Sarah Wagner',
    'Brittany Rangel',
    'Darren Taylor')
ORDER BY
  decrypted_name
```
Elapsed time: 20.718 secs.
Slot time consumed: 3 m 9.189 secs.
Bytes shuffled: 0 B.
Bytes spilled to disk: 0 B.

### COUNT function on an ecrypted column

The following two queries implement a COUNT function on an encrypted column.

```
SELECT
  c_cust_key,
  COUNT( c_name )
FROM
  `pso-ox-data-pocs.loading_dataset.customers`
GROUP BY
  c_cust_key
```
Elapsed time: 2.314 secs.
Slot time consumed: 3.436 secs.
Bytes shuffled: 87.89 KB.
Bytes spilled to disk: 0 B.

```
SELECT
  c_cust_key,
  COUNT( encrypted_name )
FROM
  `pso-ox-data-pocs.loading_dataset.customers_enc`
GROUP BY
  c_cust_key
```
Elapsed time: 3.298 secs.
Slot time consumed: 9.006 secs.
Bytes shuffled: 87.89 KB.
Bytes spilled to disk: 0 B.

## Authors

* **Google PSO Team** (OpenX-PSO-Data@google.com)

## License
Copyright 2019 Google Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.


