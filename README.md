# lab2
## Task1
- STRING_AGG: -- For each post, combines all its tags into a single string, separated by commas. STRING_AGG(expression, separator)  
- JOIN: joins rows where the values in post.postid and posttag.postid match.  
- GROUP BY: Groups all rows belonging to the same post title, so STRING_AGG() can aggregate the tags correctly.  
- ORDER BY: Sorts the final output alphabetically by the post title.  
<pre>
SELECT
    post.title,
    string_agg(posttag.tag, ', ' ORDER BY posttag.tag) AS tags
FROM
    post
JOIN
    posttag ON post.postid = posttag.postid
GROUP BY
    post.title
ORDER BY	
    post.title;
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
        post.postid,
        post.title,
        DENSE_RANK() OVER (ORDER BY COUNT(likes.postid) DESC) AS rank
    FROM post
    JOIN posttag ON post.postid = posttag.postid
    LEFT JOIN likes ON post.postid = likes.postid
    WHERE posttag.tag = '#leadership'
    GROUP BY post.postid, post.title
) AS ranked
WHERE
    rank <= 5
ORDER BY
    rank;
</pre>

## Task3
- generate_series creates weeks 1–30.   
- First subquery: counts users whose first subscription occurred in that week.    
- Second subquery: counts later subscription events (kept customers) in that week.    
- Third subquery: counts posts made in that week.    
- date_part('week', ...) extracts week numbers for comparison.  

<pre>
WITH first_sub AS (
    SELECT
        userid,
        MIN(date) AS first_date
    FROM subscription
    GROUP BY userid
)
SELECT
    week,
    (
        SELECT COUNT(*)
        FROM first_sub
        WHERE date_part('week', first_date) = week
    ) AS new_customers,
    (
        SELECT COUNT(*)
        FROM first_sub
        JOIN subscription ON subscription.userid = first_sub.userid
        WHERE subscription.date > first_sub.first_date
          AND date_part('week', subscription.date) = week
    ) AS kept_customers,
    (
        SELECT COUNT(*)
        FROM post
        WHERE date_part('week', post.date) = week
    ) AS activity
FROM generate_series(1, 30) AS week
ORDER BY week;
</pre>

## Task4
- first_reg: earliest subscription per user.  
- jan_regs: users whose first registration was in January.  
- LEFT JOIN friend keeps users even with no friends.  
- EVERY(...) checks if all joined rows have a non-NULL friendid.  
- GROUP BY aggregates per user.

<pre>
WITH first_reg AS (
  SELECT userid, MIN(date) AS registration_date
  FROM subscription
  GROUP BY userid
),
jan_regs AS (
  SELECT users.userid, users.name, first_reg.registration_date
  FROM users
  JOIN first_reg USING (userid)
  WHERE date_part('month', first_reg.registration_date) = 1
)
SELECT
  jan_regs.name,
  EVERY(friend.friendid IS NOT NULL) AS has_friend,
  jan_regs.registration_date AS registration_date
FROM jan_regs
LEFT JOIN friend ON friend.userid = jan_regs.userid
GROUP BY jan_regs.name, jan_regs.registration_date
ORDER BY jan_regs.name;
</pre>

## Task5
- Recursive CTE starting from userid = 20.  
- Base case selects all direct friends of user 20.  
- Recursive step follows the chain by matching friendid → next userid.  
- NOT EXISTS prevents reusing the same edge.  
- Final SELECT outputs the entire chain.

<pre>
WITH RECURSIVE friend_chain(name, user_id, friend_id) AS (
    SELECT
        users.name,
        users.userid,
        friend.friendid
    FROM users
    LEFT JOIN friend ON friend.userid = users.userid
    WHERE users.userid = 20

    UNION ALL

    SELECT
        users.name,
        users.userid,
        friend.friendid
    FROM friend_chain
    JOIN users ON users.userid = friend_chain.friend_id
    LEFT JOIN friend ON friend.userid = users.userid
    WHERE friend_chain.friend_id IS NOT NULL
)
SELECT
    name,
    user_id,
    friend_id
FROM friend_chain;
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
SELECT postid, userid, date
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
