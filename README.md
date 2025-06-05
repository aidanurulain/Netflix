# OVERVIEW
This project involves a comprehensive analysis of Netflix's movies and TV shows data using SQL. The goals is to extract valueable insights and answer business question based on dataset

# OBJECTIVE
- Analyze the distribution on content
- Identify the most common rating between movies and TV shows
- Analyze content based on release_years,countries and durations
- Explore and categorize content based on specific criteria and keywords

#DATA SET
- Dataset Link: https://www.kaggle.com/datasets/shivamb/netflix-shows?resource=download

#Business Problem and Solutions
## -- 1. Count the number of Movies vs TV Shows

SELECT DISTINCT type, COUNT(*) AS movie_types
FROM netflix_table
GROUP BY type;

Objective: Determine how much content on Netflix falls under each type (Movie vs TV Show)

## -- 2. Find the most common rating for movies and TV shows
SELECT *
FROM 
(
SELECT 
	rating, 
	COUNT(*) as rating_number, 
	RANK() OVER(PARTITION BY type ORDER BY COUNT(*) DESC) as ranking
FROM netflix_table
GROUP BY type,rating
) as T1
WHERE ranking = 1;

Objective: Identify the most frequently used content rating for each type 

## -- 3. List all movies released in a specific year (e.g., 2020)
SELECT title, release_year
FROM
(SELECT *
FROM netflix_table
WHERE type = 'movie'
) AS T2
WHERE release_year = 2020;

Objective: Retrieve all movie titles that were released in a specified year

## -- 4. Find the top 5 countries with the most content on Netflix
WITH RECURSIVE country_split AS (
    SELECT 
        TRIM(SUBSTRING_INDEX(country, ',', 1)) AS country,
        SUBSTRING_INDEX(country, ',', -1) AS remaining
    FROM netflix_table
    WHERE country IS NOT NULL

    UNION ALL

    SELECT 
        TRIM(SUBSTRING_INDEX(remaining, ',', 1)),
        SUBSTRING_INDEX(remaining, ',', -1)
    FROM country_split
    WHERE remaining LIKE '%,%'
)

SELECT country, COUNT(*) AS total_content
FROM country_split
WHERE country <> ''
GROUP BY country
ORDER BY total_content DESC
LIMIT 5;

Objective: To identify the top 5 countries that have produced the most content (movies and TV shows) available on Netflix

## -- 5. Identify the longest movie
        SELECT *
        FROM netflix_table
        WHERE type = 'movie' AND duration = (SELECT MAX(duration) FROM netflix_table)
        ORDER BY duration DESC;

Objective: Find the movie with the longest duration in terms of time

## -- 6. Find content added in the last 5 years
        SELECT *
        FROM netflix_table
        WHERE STR_TO_DATE(date_added,'%M %d, %Y') >= CURDATE() - INTERVAL 5 YEAR;

Objective: Get a list of content (movies or shows) that was added to Netflix in the last 5 years

## -- 7. Find all the movies/TV shows by director 'Rajiv Chilaka'!
        SELECT type,title,director
        FROM netflix_table
        WHERE LOWER(director) LIKE '%Rajiv Chilaka%';

Objective: List all content directed by Rajiv Chilaka


## -- 8. List all TV shows with more than 5 seasons
        SELECT title,duration
        FROM
        (SELECT type, title, duration,  SUBSTRING_INDEX(duration, ' ',1) AS number, SUBSTRING_INDEX(duration, ' ',-1) AS season
        FROM netflix_table
        WHERE type = 'TV Show' AND duration LIKE '%seasons%'
        ) as T2
        WHERE number > 5;

Objective: Identify TV shows that have more than 5 seasons


## -- 9. Count the number of content items in each genre
        WITH RECURSIVE genre_split AS (
          SELECT 
            TRIM(SUBSTRING_INDEX(listed_in, ',', 1)) AS genre,
            SUBSTRING_INDEX(listed_in, ',', -1) AS remaining,
            listed_in,
            1 AS level
          FROM netflix_table
          WHERE listed_in IS NOT NULL
        
          UNION ALL
        
          SELECT 
            TRIM(SUBSTRING_INDEX(remaining, ',', 1)) AS genre,
            SUBSTRING_INDEX(remaining, ',', -1) AS remaining,
            listed_in,
            level + 1
          FROM genre_split
          WHERE remaining LIKE '%,%'
        )
        
        SELECT genre, COUNT(*) AS total
        FROM genre_split
        WHERE genre <> ''
        GROUP BY genre
        ORDER BY total DESC;

Objective: Count how many pieces of content fall into each genre, even when multiple genres are listed per item


## -- 10.Find each year and the average numbers of content release in India on netflix. return top 5 year with highest avg content release!
        SELECT EXTRACT(YEAR FROM STR_TO_DATE(date_added,'%M %d, %Y')) AS year,
        	   ROUND((COUNT(*)/(SELECT COUNT(*) AS india_total FROM netflix_table WHERE LOWER(country) LIKE '%india%')),2) * 100 AS percentage
        FROM netflix_table
        WHERE LOWER(country) LIKE '%india%'
        GROUP BY year
        ORDER BY year;

Objective: Determine how content produced in India is distributed across years and find years with the highest percentage contribution


## -- 11. List all movies that are documentaries
        SELECT *
        FROM netflix_table
        WHERE LOWER(type) = 'Movie' AND listed_in LIKE '%Documentaries%';

Objective: Retrieve all movies that are categorized as documentaries


## -- 12. Find all content without a director
        SET SQL_SAFE_UPDATES = 0;
        
        UPDATE netflix_table
        SET director = NULL
        WHERE director = '';
        
        SET SQL_SAFE_UPDATES = 1;
        
        SELECT *
        FROM netflix_table
        WHERE director IS NULL;

Objective: Identify entries in the dataset that are missing a director

## -- 13. Find how many movies actor 'Salman Khan' appeared in last 10 years!
        SELECT COUNT(*) as salman_movie_count
        FROM
        (
        SELECT *
        FROM
        (
        SELECT *
        FROM netflix_table
        WHERE STR_TO_DATE(date_added,'%M %d, %Y') >= CURDATE() - INTERVAL 10 YEAR
        ) AS T2
        WHERE LOWER(cast) LIKE '%Salman Khan%' AND type = 'Movie'
        ) AS T1;

Objective: Count the number of movies featuring Salman Khan added to Netflix in the past 10 years


## -- 14. Find the top 10 actors who have appeared in the highest number of movies produced in India.
        WITH RECURSIVE split_actors AS (
            SELECT 
                TRIM(SUBSTRING_INDEX(cast, ',', 1)) AS actor,
                SUBSTRING_INDEX(cast, ',', -1) AS remaining,
                1 AS level
            FROM netflix_table
            WHERE country LIKE '%India%'
              AND cast IS NOT NULL
        
            UNION ALL
        
            SELECT 
                TRIM(SUBSTRING_INDEX(remaining, ',', 1)) AS actor,
                SUBSTRING_INDEX(remaining, ',', -1),
                level + 1
            FROM split_actors
            WHERE remaining LIKE '%,%'
        )
        SELECT actor, COUNT(*) AS total_content
        FROM split_actors
        WHERE actor <> ''
        GROUP BY actor
        ORDER BY total_content DESC
        LIMIT 10;

Objective: Identify the actors with the most appearances in Indian Netflix content


## -- 15.Categorize the content based on the presence of the keywords 'kill' and 'violence' in the description field. Label content containing these keywords as 'Bad' and all other content as 'Good'. Count how many items fall into each category.
        DROP TEMPORARY TABLE IF EXISTS new_table;
        CREATE TEMPORARY TABLE new_table AS
        SELECT *, 
        	CASE
        		WHEN LOWER(description) LIKE '%kill%' OR LOWER(description) LIKE '%violence%' THEN 'bad'
                ELSE 'good'
        	END AS content_lable
        FROM netflix_table;
        
        SELECT COUNT(*) as cateogary,content_lable
        FROM new_table
        GROUP BY 2;

Objective: Classify content as either "Good" or "Bad" based on presence of violent keywords and count how many items fall into each category

# CONCLUSION
This analysis provides a comprehensive view of Netflix's content and can help inform content strategy and decision-making
