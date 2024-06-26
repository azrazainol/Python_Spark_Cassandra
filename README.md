# STQD6324_Assignment_04

This assignment uses uses PySpark which connects Python to Spark and Cassandra. The purpose is to extract insights from the MovieLens 100k Dataset.

## Prepare Tables in Cassandra

```sql
cqlsh


CREATE KEYSPACE movielens WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '1'} AND durable_writes = true;


USE movielens;


CREATE TABLE ratings (
    user_id int,
    movie_id int,
    rating int,
    time int,
    PRIMARY KEY (user_id, movie_id)
);


CREATE TABLE names (
    movie_id int PRIMARY KEY,
    title text,
    release_date text,
    vid_release_date text,
    url text,
    unknown int,
    action int,
    adventure int,
    animation int,
    children int,
    comedy int,
    crime int,
    documentary int,
    drama int,
    fantasy int,
    film_noir int,
    horror int,
    musical int,
    mystery int,
    romance int,
    sci_fi int,
    thriller int,
    war int,
    western int
);


DESCRIBE TABLE ratings;
DESCRIBE TABLE names;


exit
```


## Add .py Script in PuTTy

### Load Libraries

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions
from pyspark.sql import Row

# Initialize Spark session
spark = SparkSession.builder \
    .appName("MovieLens Analysis") \
    .config("spark.cassandra.connection.host", "127.0.0.1") \
    .getOrCreate()

# Step 1: Parse the u.user file
def parse_user(line):
    fields = line.split('|')
    return Row(user_id=int(fields[0]), age=int(fields[1]), gender=fields[2], occupation=fields[3], zip=fields[4])

def parse_data(line):
    fields = line.split("\t")
    return Row(user_id=int(fields[0]), movie_id=int(fields[1]), rating=int(fields[2]), time=int(fields[3]))

def parse_item(line):
    fields = line.split("|")
    return Row(movie_id=int(fields[0]), title=fields[1], release_date=fields[2], vid_release_date=fields[3], url=fields[4],
               unknown=int(fields[5]), action=int(fields[6]), adventure=int(fields[7]), animation=int(fields[8]),
               children=int(fields[9]), comedy=int(fields[10]), crime=int(fields[11]), documentary=int(fields[12]),
               drama=int(fields[13]), fantasy=int(fields[14]), film_noir=int(fields[15]), horror=int(fields[16]),
               musical=int(fields[17]), mystery=int(fields[18]), romance=int(fields[19]), sci_fi=int(fields[20]),
               thriller=int(fields[21]), war=int(fields[22]), western=int(fields[23]))

if __name__ == "__main__":
    # Create a SparkSession
    spark = SparkSession.builder.appName("MovieLens Analysis").config("spark.cassandra.connection.host", "127.0.0.1").getOrCreate()

    # Parse data files
    lines1 = spark.sparkContext.textFile("hdfs:///user/maria_dev/ml-100k/u.user")
    users = lines1.map(parse_user)

    lines2 = spark.sparkContext.textFile("hdfs:///user/maria_dev/ml-100k/u.data")
    ratings = lines2.map(parse_data)

    lines3 = spark.sparkContext.textFile("hdfs:///user/maria_dev/ml-100k/u.item")
    names = lines3.map(parse_item)

    # Convert to DataFrames
    usersDataset = spark.createDataFrame(users)
    ratingsDataset = spark.createDataFrame(ratings)
    namesDataset = spark.createDataFrame(names)

    # Write to Cassandra
    usersDataset.write \
        .format("org.apache.spark.sql.cassandra") \
        .mode('append') \
        .options(table="users", keyspace="movielens") \
        .save()

    ratingsDataset.write \
        .format("org.apache.spark.sql.cassandra") \
        .mode('append') \
        .options(table="ratings", keyspace="movielens") \
        .save()

    namesDataset.write \
        .format("org.apache.spark.sql.cassandra") \
        .mode('append') \
        .options(table="names", keyspace="movielens") \
        .save()

    # Read from Cassandra into DataFrames
    readUsers = spark.read \
        .format("org.apache.spark.sql.cassandra") \
        .options(table="users", keyspace="movielens").load()

    readRatings = spark.read \
        .format("org.apache.spark.sql.cassandra") \
        .options(table="ratings", keyspace="movielens").load()

    readNames = spark.read \
        .format("org.apache.spark.sql.cassandra") \
        .options(table="names", keyspace="movielens").load()

    # Create temporary views for SQL querying
    readUsers.createOrReplaceTempView("users")
    readRatings.createOrReplaceTempView("ratings")
    readNames.createOrReplaceTempView("names")

    # Execute SQL queries

    # i) Calculate the average rating for each movie (top ten results)
    sql_query_i = spark.sql("""
    SELECT movie_id, AVG(rating) AS avg_rating
    FROM ratings
    GROUP BY movie_id
    ORDER BY avg_rating DESC
    LIMIT 10
    """)
    print("Average Rating by Movie")
    sql_query_i.show()

    # ii) Identify the top ten movies with the highest average ratings
    sql_query_ii = """
    SELECT n.title, AVG(r.rating) AS avg_rating
    FROM ratings r
    JOIN names n ON r.movie_id = n.movie_id
    GROUP BY n.title
    ORDER BY avg_rating DESC
    LIMIT 10
    """
    print("Movies with Highest Average Ratings")
    spark.sql(sql_query_ii).show()

    # iii) Find the users who have rated at least 50 movies and identify their favourite movie genres
    sql_query_iii = """
    SELECT u.user_id, u.age, u.occupation, u.gender, g.genre, COUNT(*) AS count_ratings
    FROM users u
    JOIN ratings r ON u.user_id = r.user_id
    JOIN names n ON r.movie_id = n.movie_id
    LATERAL VIEW explode(array("unknown", "action", "adventure", "animation", "children", "comedy", "crime", "documentary",
        "drama", "fantasy", "film_noir", "horror", "musical", "mystery", "romance", "sci_fi", "thriller", "war", "western")) g AS genre
    GROUP BY u.user_id, u.age, u.occupation, u.gender, g.genre
    HAVING COUNT(*) >= 50
    ORDER BY u.user_id, count_ratings DESC
    """
    print("Favourite Genres for Frequent Users")
    spark.sql(sql_query_iii).show(10)

    # iv) Find all the users with age that is less than 20 years old
    sql_query_iv = """
    SELECT *
    FROM users
    WHERE age < 20
    """
    print("Users Under 20 Years Old")
    spark.sql(sql_query_iv).show(10)

    # v) Find all the users who have the occupation “scientist” and their age is between 30 and 40 years old
    sql_query_v = """
    SELECT *
    FROM users
    WHERE occupation = 'scientist' AND age BETWEEN 30 AND 40
    """
    print("Scientists between 30-40 Years Old")
    spark.sql(sql_query_v).show(10)

    # Stop the Spark session
    spark.stop()

```






