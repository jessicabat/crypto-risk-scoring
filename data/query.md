````sql
WITH wallet_features AS (
  SELECT
    address,
    COUNTIF(role = 'sender') AS n_sent_tx,
    COUNTIF(role = 'receiver') AS n_received_tx,
    -- ETH volumes (Wei -> ETH)
    SUM(CASE WHEN role = 'sender' THEN value_wei ELSE 0 END) / 1e18 AS total_sent_eth,
    SUM(CASE WHEN role = 'receiver' THEN value_wei ELSE 0 END) / 1e18 AS total_received_eth,
    COUNT(DISTINCT counterparty) AS n_unique_counterparties,
    -- Gas features (only meaningful for senders)
    AVG(CASE WHEN role = 'sender' THEN gas ELSE NULL END) AS avg_gas_used_sent,
    AVG(CASE WHEN role = 'sender' THEN gas_price / 1e9 ELSE NULL END) AS avg_gas_price_gwei_sent,
    MIN(block_timestamp) AS first_tx_time,
    MAX(block_timestamp) AS last_tx_time
  FROM (
    SELECT
      from_address AS address,
      to_address   AS counterparty,
      'sender'     AS role,
      CAST(value AS NUMERIC) AS value_wei,
      gas,
      gas_price,
      block_timestamp
    FROM `bigquery-public-data.crypto_ethereum.transactions`
    WHERE block_timestamp BETWEEN '2025-11-13' AND '2025-12-13'

    UNION ALL

    SELECT
      to_address   AS address,
      from_address AS counterparty,
      'receiver'   AS role,
      CAST(value AS NUMERIC) AS value_wei,
      gas,
      gas_price,
      block_timestamp
    FROM `bigquery-public-data.crypto_ethereum.transactions`
    WHERE block_timestamp BETWEEN '2025-11-13' AND '2025-12-13'
  )
  GROUP BY address
),

token_features AS (
  SELECT
    address,
    COUNTIF(role = 'token_sender')   AS n_token_sent,
    COUNTIF(role = 'token_receiver') AS n_token_received,
    COUNT(DISTINCT token_address)    AS n_unique_tokens_moved
  FROM (
    SELECT
      from_address AS address,
      'token_sender' AS role,
      token_address
    FROM `bigquery-public-data.crypto_ethereum.token_transfers`
    WHERE block_timestamp BETWEEN '2025-11-13' AND '2025-12-13'

    UNION ALL

    SELECT
      to_address AS address,
      'token_receiver' AS role,
      token_address
    FROM `bigquery-public-data.crypto_ethereum.token_transfers`
    WHERE block_timestamp BETWEEN '2025-11-13' AND '2025-12-13'
  )
  GROUP BY address
)

SELECT
  w.address,
  w.n_sent_tx,
  w.n_received_tx,
  w.total_sent_eth,
  w.total_received_eth,
  w.n_unique_counterparties,
  w.avg_gas_used_sent,
  w.avg_gas_price_gwei_sent,
  w.first_tx_time,
  w.last_tx_time,
  TIMESTAMP_DIFF(w.last_tx_time, w.first_tx_time, DAY) AS days_active_in_window,
  IFNULL(t.n_token_sent, 0)         AS n_token_sent,
  IFNULL(t.n_token_received, 0)     AS n_token_received,
  IFNULL(t.n_unique_tokens_moved, 0) AS n_unique_tokens_moved
FROM wallet_features w
LEFT JOIN token_features t
  ON w.address = t.address
WHERE (w.n_sent_tx > 0 OR w.total_received_eth > 0.01)
LIMIT 1200000;
````