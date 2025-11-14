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
  -- start with Anas
  SELECT 
    u.name,
    f.userid,
    f.friendid
  FROM friend f
  JOIN users u ON u.userid = f.userid
  WHERE f.userid = 20

  UNION ALL

  -- keep finding the next friend in the chain
  SELECT 
    u.name,
    f.userid,
    f.friendid
  FROM friend f
  JOIN users u ON u.userid = f.userid
  JOIN friend_chain c ON f.userid = c.friendid
  WHERE NOT EXISTS (
    SELECT 1
    FROM friend skip
    WHERE skip.userid = c.userid
      AND skip.friendid = f.friendid
  )
)

SELECT * FROM friend_chain;

</pre>

## P+ Assignment
<pre>
WITH march_posts AS (
  SELECT p.userid, p.postid
  FROM post p
  WHERE date_part('month', p.date) = 3
),
authors AS (
  SELECT DISTINCT u.userid, u.name
  FROM users u
  JOIN march_posts mp USING (userid)
),
likes_totals AS (
  -- count all likes on those March posts (regardless of when the like occurred)
  SELECT mp.userid, COUNT(l.postid) AS total_likes
  FROM march_posts mp
  LEFT JOIN likes l ON l.postid = mp.postid
  GROUP BY mp.userid
)
SELECT
  a.name AS user,
  CASE WHEN COALESCE(lt.total_likes, 0) >= 50 THEN 't' ELSE 'f' END AS t
FROM authors a
LEFT JOIN likes_totals lt USING (userid)
ORDER BY a.name;
</pre>
