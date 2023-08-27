# BigQuery Queries for Google Analytics

A running list of useful queries for use with Google Analytics 4 data stored in a connected BigQuery project. If you want to avoid data thresholding, these will provide precise analytics numbers.

Documentation and other details for these queries can be found at [Kevinleary.net > Google Analytics Queries for BigQuery](https://www.kevinleary.net/blog/google-analytics-queries-for-bigquery/). If you have any recommendations or want to make suggestions/additions feel free to submit a [pull request](https://github.com/Kevinlearynet/bigquery-google-analytics-queries/pulls).

## Daily Pageviews by Traffic Source

```.language-sql
SELECT
  event_date,
  traffic_source.source as traffic_source,
  COUNT(*) as pageviews
FROM `{BIGQUERY-PROJECT-ID}.analytics_{GA4-ID}.events_*`
WHERE event_name = 'page_view'
AND _TABLE_SUFFIX BETWEEN
  FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)) AND
  FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY))
GROUP BY 1, 2
ORDER BY 1 ASC, 3 DESC
```

## Daily Pageviews by URL

```.language-sql
SELECT
  event_date,
  LOWER( (
    SELECT REGEXP_EXTRACT_ALL(value.string_value, r"([^?]+)")[
    OFFSET (0)]
    FROM UNNEST(event_params)
    WHERE key = 'page_location'
  ) ) AS url,
  COUNT(*) AS pageviews
FROM `{BIGQUERY-PROJECT-ID}.analytics_{GA4-ID}.events_*`
WHERE
  _TABLE_SUFFIX BETWEEN
  FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)) AND
  FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY))
GROUP BY 1, 2
ORDER BY 1 DESC, 3 DESC
```

## Unique Pageviews by Day

```.language-sql
WITH pageviews AS (
  SELECT
    event_date,
    (
      SELECT REGEXP_EXTRACT_ALL(value.string_value, r"([^?]+)")[
      OFFSET (0)]
      FROM UNNEST(event_params)
      WHERE key = 'page_location'
    ) as url,
    (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS ga_session_id
  FROM `{BIGQUERY-PROJECT-ID}.analytics_{GA4-ID}.events_*`
  WHERE
    _TABLE_SUFFIX BETWEEN
    FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)) AND
    FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY))
  GROUP BY 1, 2, 3
)

SELECT
  event_date,
  url,
  COUNT(ga_session_id) as unique_pageviews,
FROM pageviews
GROUP BY 1, 2
ORDER BY 1 ASC, 3 DESC
```

## Daily Users & New User Counts

```.language-sql
WITH users AS (
  SELECT
    event_date,
    user_pseudo_id,
    MAX(IF(event_name IN ('first_visit', 'first_open'), 1, 0)) AS is_new_user
  FROM `{BIGQUERY-PROJECT-ID}.analytics_{GA4-ID}.events_*`
  WHERE
    _TABLE_SUFFIX BETWEEN
    FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)) AND
    FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY))
  GROUP BY 1, 2
)

SELECT
event_date,
  COUNT(users) AS all_users,
  SUM(is_new_user) AS new_users,
FROM users
GROUP BY 1
ORDER BY 1 ASC
```

## Daily Sessions

This query will give you a single and column containing the total number of unique sessions that occurred during the given time range.

```.language-sql
WITH sessions AS (
  SELECT
    event_date,
    (
      SELECT value.int_value
      FROM UNNEST(event_params)
      WHERE key = 'ga_session_id'
    ) AS ga_session_id,
  FROM `{BIGQUERY-PROJECT-ID}.analytics_{GA4-ID}.events_*`
  WHERE
    _TABLE_SUFFIX BETWEEN
    FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)) AND
    FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY))
  GROUP BY 1, 2
)

SELECT
  event_date,
  COUNT(sessions) AS sessions
FROM sessions
GROUP BY 1
ORDER BY 1 ASC
```

## Timestamped Conversion Events

```.language-sql
SELECT
  event_timestamp,
  (
    SELECT value.string_value
    FROM UNNEST(event_params)
    WHERE key = 'form_name'
  ) AS form_name,
  (
    SELECT value.string_value
    FROM UNNEST(event_params)
    WHERE key = 'form_destination'
  ) AS form_destination,
  (
    SELECT value.string_value
    FROM UNNEST(event_params)
    WHERE key = 'form_id'
  ) AS form_id,
  LOWER( (
    SELECT REGEXP_EXTRACT_ALL(value.string_value, r"([^?]+)")[
    OFFSET (0)]
    FROM UNNEST(event_params)
    WHERE key = 'page_location'
  ) ) AS url,
FROM `{BIGQUERY-PROJECT-ID}.analytics_{GA4-ID}.events_*`
WHERE
  event_name = 'form_submit'
  AND _TABLE_SUFFIX BETWEEN
    FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)) AND
    FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY))
```

## Select Normalized URL's

```.language-sql
(
  SELECT REGEXP_EXTRACT_ALL(value.string_value, r"([^?]+)")[
  OFFSET (0)]
  FROM UNNEST(event_params)
  WHERE key = 'page_location'
) as url,
```

## Date Ranges

### Between Two Dates

```.language-sql
WHERE _TABLE_SUFFIX BETWEEN '20230801' AND '20230818'
```

### Last 7 Days

```.language-sql
WHERE
  _TABLE_SUFFIX BETWEEN
  FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)) AND
  FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY))
```

### Last 14 Days

```.language-sql
WHERE
  _TABLE_SUFFIX BETWEEN
  FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 14 DAY)) AND
  FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY))
```

### Last 30 Days

```.language-sql
WHERE
  _TABLE_SUFFIX BETWEEN
  FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)) AND
  FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY))
```

### Any Number of Days (X = Days)

```.language-sql
WHERE
  _TABLE_SUFFIX BETWEEN
  FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL X DAY)) AND
  FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY))
```
