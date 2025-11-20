# lab2
## Task1
- STRING_AGG: -- For each post, combines all its tags into a single string, separated by commas. STRING_AGG(expression, separator)  
- JOIN: joins rows where the values in post.postid and posttag.postid match.  
- GROUP BY: Groups all rows belonging to the same post title, so STRING_AGG() can aggregate the tags correctly.  
- ORDER BY: Sorts the final output alphabetically by the post title.  
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
- We join likes, posttag and post where the postid is the same  
- Only include entries WHERE posttag.tag = '#leadership'  
- COUNT(likes.postid) counts how many times each postid appears in the likes table  
- Ranks every post based on the number of likes it has.  
- DESC means decsending order, ascending order is the default (ORDER BY rank = ORDER by rank ASC)  
- Subquery since we cant use WHERE before the rank column is built  
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
- generate_series creates weeks 1–30.   
- First subquery: counts users whose first subscription occurred in that week.    
- Second subquery: counts later subscription events (kept customers) in that week.    
- Third subquery: counts posts made in that week.    
- date_part('week', ...) extracts week numbers for comparison.  

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
- first_reg: earliest subscription per user.  
- jan_regs: users whose first registration was in January.  
- LEFT JOIN friend keeps users even with no friends.  
- EVERY(...) checks if all joined rows have a non-NULL friendid.  
- GROUP BY aggregates per user.

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
- Recursive CTE starting from userid = 20.  
- Base case selects all direct friends of user 20.  
- Recursive step follows the chain by matching friendid → next userid.  
- NOT EXISTS prevents reusing the same edge.  
- Final SELECT outputs the entire chain.

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
- We first use a CTE to filter out posts that have been created in March using WHERE date_part('month', post.date) = 3, this table includes postID and userID  
- Our LEFT JOIN takes the postID column from likes, only when that postID refers to a post that is posted in march, giving us one row per like for each March post.  
- We then GROUP BY users.userid, this 'collects' all March posts for a user into one group.  
- Within that group, COUNT(likes.postid) totals every like row across every March post the user has.  
- By using (COUNT(likes.postid) >= 50), we get the value outputted as a boolean, that is true if the total likes is greater or equal to 50  
- Finally, we also join users (by userID) to get the names, and we order by the names to get the output in alphabetical order  

<pre>
WITH march_post_cte AS (
SELECT
    postid,
    userid,
    date
FROM post
WHERE date_part('month', post.date) = 3
)
SELECT
    users.name,
    (COUNT(likes.postid) >= 50) AS received_likes
    FROM march_post_cte
    LEFT JOIN likes ON likes.postid = march_post_cte.postid
    JOIN users ON users.userid = march_post_cte.userid
    GROUP BY users.userid
    ORDER BY users.name;
</pre>
