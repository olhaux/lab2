# lab2
## Task1
<pre>
SELECT
    p.title,
    string_agg(pt.tag, ', ' ORDER BY pt.tag) AS tags
FROM
    post p
JOIN
    posttag pt ON p.postid = pt.postid
GROUP BY
    p.title
ORDER BY	
    p.title;
</pre>

## Task2
<pre>
  SELECT
    postid,
    title,
    rank
FROM (
    SELECT
        p.postid,
        p.title,
        DENSE_RANK() OVER (ORDER BY COUNT(l.postid) DESC) AS rank
    FROM
        post p
    JOIN
        posttag pt ON p.postid = pt.postid
    LEFT JOIN
        likes l ON p.postid = l.postid
    WHERE
        lower(pt.tag) = '#leadership'
    GROUP BY
        p.postid, p.title
) AS ranked
WHERE
    rank <= 5
ORDER BY
    rank,
    postid;
</pre>

## Task3
<pre>
  SELECT
  w.week,

  (
    SELECT COUNT(*)
    FROM (
      SELECT MIN(date) AS first_date
      FROM subscription
      GROUP BY userid
    ) s
    WHERE date_part('week', s.first_date) = w.week
  ) AS new_customers,

  (
    SELECT COUNT(*)
    FROM subscription s
    JOIN (
      SELECT userid, MIN(date) AS first_date
      FROM subscription
      GROUP BY userid
    ) f USING (userid)
    WHERE s.date > f.first_date
      AND date_part('week', s.date) = w.week
  ) AS kept_customers,

  (
    SELECT COUNT(*)
    FROM post p
    WHERE date_part('week', p.date) = w.week
  ) AS activity

FROM generate_series(1, 30) AS w(week)
ORDER BY w.week;
</pre>

## Task4
<pre>WITH first_reg AS (
  SELECT userid, MIN(date) AS registration_date
  FROM subscription
  GROUP BY userid
),
jan_regs AS (
  SELECT u.userid, u.name, f.registration_date
  FROM users u
  JOIN first_reg f USING (userid)
  WHERE date_part('month', f.registration_date) = 1
)
SELECT
  jr.name,
  EVERY(fr.friendid IS NOT NULL) AS has_friend,
  jr.registration_date AS registration_date
FROM jan_regs jr
LEFT JOIN friend fr ON fr.userid = jr.userid
GROUP BY jr.name, jr.registration_date
ORDER BY jr.name;
</pre>

## Task5
<pre>
  WITH RECURSIVE friend_chain AS (
  -- Step 1: start with Anas
  SELECT 
    u.name,
    f.userid,
    f.friendid
  FROM friend f
  JOIN users u ON u.userid = f.userid
  WHERE f.userid = 20

  UNION ALL

  -- Step 2: keep finding the next friend in the chain
  SELECT 
    u.name,
    f.userid,
    f.friendid
  FROM friend f
  JOIN users u ON u.userid = f.userid
  JOIN friend_chain c ON f.userid = c.friendid      -- continue chain
  WHERE NOT EXISTS (                                -- make sure X is NOT friend with Z
    SELECT 1
    FROM friend skip
    WHERE skip.userid = c.userid
      AND skip.friendid = f.friendid
  )
)

SELECT * FROM friend_chain;

</pre>
