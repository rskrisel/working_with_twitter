# Working with Twitter Data

In this workshop, we will learn how to wrangle, clean, analyze, and visualize Twitter data. As of writing this in March 2023, Twitter is no longer accepting applications for the research API. Thus, we will not spend time on learning how to collect data from the Twitter API. 

## The Dataset
The data for this workshop was retrieved from the API Twitter and focuses on Tweets related to the Biden Administration's student loan forginess program. The data was collected using twarc2, a command line tool and Python library for archiving Twitter JSON data. Each tweet is represented as a JSON object that was returned from the Twitter API. You still need access tokens from Twitter in order to use twarc2 to interact with the Twitter API.

To collect Tweets related to the student loan forgiveness program, I made the following query:

```
twarc2 search "(studentloanforgiveness OR \"student loan forgiveness\" OR cancelstudentdebt OR \"cancel student debt\" OR studentdebtrelief OR \"student debt relief\") -is:retweet lang:en" --start-time "2022-08-01" --end-time "2023-03-01" --archive student_loan_json.jsonl

```
In this command, I am searching for a variety of hashtags (without including the '#') and keyword phrases relating to the issue of student loan forgiveness. I specified that I don't want to include retweets (-is:retweet) and I also specified to only pick up Tweets that are in English. I selected the timeframe to be between August 2022 and March 2023, encompassing when the announcement was first made until the current Supreme Court review.

You can download the dataset [here](https://gc-cuny-edu.zoom.us/j/87281241758?pwd=QnRpamMycFliU3VuL2JRWWVuckVEQT09). 



```python
# !twarc2 search "(studentloanforgiveness OR \"student loan forgiveness\" OR cancelstudentdebt OR \"cancel student debt\" OR studentdebtrelief OR \"student debt relief\") -is:retweet lang:en" --start-time "2022-08-01" --end-time "2023-03-15" --archive student_loan_3_15.jsonl
```

## Reading in the JSON File

We will use Pandas to bring our JSON data into our Python environment and use the read_json() method to import our file. We also specify a few parameters:
- orient = split
- convert_dates = True, to convert any column with dates into datetime format
- keep_default_dates = ['created_at'], to specify which column we want to be treated as datetime



```python
import pandas as pd
```


```python
tweets_df = pd.read_json('student_loan_json.jsonl', orient='split', convert_dates = True,
                       keep_default_dates = ['created_at'])
```

## Cleaning the Data

Let's take a look at our Dataframe:


```python
tweets_df
```

As you can see, our dataset is quite large, including 83 columns. Let's take a closer look at these columns:


```python
tweets_df.columns
```

Let's clean up these columns by renaming those that might be most useful:


```python
tweets_df.rename(columns={'created_at': 'date',
                          'public_metrics.retweet_count': 'retweets', 
                          'author.username': 'username', 
                          'author.name': 'name',
                          'author.verified': 'verified', 
                          'public_metrics.like_count': 'likes', 
                          'public_metrics.quote_count': 'quotes', 
                          'public_metrics.reply_count': 'replies',
                           'author.description': 'user_bio'},
                            inplace=True)
```

Now, let's select just the columns that are most useful to us:


```python
tweets_df = tweets_df[['date', 'username', 'name', 'verified', 'text', 'retweets',
           'likes', 'replies',  'quotes', 'user_bio']]
```


```python
tweets_df
```

Finally, let's inspect our datatypes:


```python
tweets_df.dtypes
```

## Sorting the data

With just 10 columns, now we have a dataset that is much more manageable! 
</br>
Let's take a closer look at our data. First, let's sort by retweets in descending order:


```python
tweets_df.sort_values(by='retweets', ascending=False)[:5]
```

Now, we can sort our data by time in ascending order (meaning earliest to latest)


```python
tweets_df.sort_values(by='date', ascending=True)[:5]
```

## Ploting Tweets Over Time

To visualize our Tweets over time, we can add a column with a 1 for every row and then create another column adding the single count with the number of retweets. This will allow us to capture the total number of tweets, which we can then use to count how many tweets were published per day, week, month, or year


```python
tweets_df = tweets_df.assign(count=1)
```


```python
tweets_df['total_count']= tweets_df['count'] + tweets_df['retweets']
```

We can use the sum() method to count all the values in the total_count column:


```python
sum_tweets = tweets_df['total_count'].sum()
print(sum_tweets)
```


```python
tweets_df.sort_values(by='total_count', ascending=False)[:5]
```

Let's set the date column to the index so we can do some special date manipulations:


```python
tweets_df = tweets_df.set_index('date')
tweets_df
```

Since our index is a datetime value, we can use the special Pandas method .resample() to group the tweets by month, add them up, and plot them over time.


```python
tweets_df['total_count'].resample('M').sum()\
.plot(title='Student Debt Relief Tweets by Month');
```


```python
tweets_df['total_count'].resample('D').sum()\
.plot(title='Student Debt Relief Tweets by Day');
```

## Extracting Hashtags

To extract the hashtags from our collection of tweets, we are going to define a new function,  extract_hashtags_list, that will seek words starting with the "#" key and add those words without the "#" to an empty list, hashtag_list:


```python
def extract_hashtags_list(text):
    hashtag_list = []
    for word in text.split():
        if word[0] == '#':
            hashtag_list.append(word[1:])
    return hashtag_list
```

Let's apply our extract_hashtags_list function to the "text" column in our tweets_df dataframe and save the results in a new column, hashtags_list:


```python
tweets_df['hashtags_list'] = tweets_df['text'].apply(extract_hashtags_list)
```


```python
tweets_df.sort_values(by='hashtags_list', ascending=False)[:15]
```

### Cleaning our hashtag results and preparing the dataframe for visualization

Right now, the hashtags in the our hashtags_list column are written in list format, with multiple hashtags in a cell in some cases. In general, we want to have just a single value per cell. To do this, we can use the "explode" method, which will ensure we have just a single hashtag per row while maintaining the rest of the corresponding data. 


```python
tweets_df=tweets_df.explode("hashtags_list")
```

Let's take a look at our dataframe:


```python
tweets_df
```

Looks like we have many empty cells in the hashtags_list column. Let's drop these rows and set our dataframe equal to a new variable, hashtags_df.


```python
hashtags_df=tweets_df.dropna(subset=['hashtags_list'])
```

Let's take a look at our hashtags_df dataframe, sorted by the total number of tweets:


```python
hashtags_df.sort_values(by='total_count', ascending=False)[:10]
```

As you can see, our index is still our "date" column. Let's reset the index:


```python
hashtags_df=hashtags_df.reset_index()
```

Next, let's just select the columns we need:


```python
hashtags_df=hashtags_df[['date', 'total_count', 'hashtags_list']]
```

We can take a look at our dataframe, sorted by the total number of tweets:


```python
hashtags_df.sort_values(by='total_count', ascending=False)[:10]
```

### Visualizing the most commonly used hashtags
To start, let's use the groupby function to group data based on the "hashtags_list" column:


```python
hashtags_df.groupby('hashtags_list')
```

Next, we add the "sum" function to add all instances of the occurence of each hashtag across our entire dataset and we see our data sorted in descending order:


```python
hashtags_df.groupby('hashtags_list').sum().sort_values(by='total_count', ascending=False)[:20]
```

Let's save these results to our hashtags_df and stack on an additional command to reset the index:


```python
hashtags_df=hashtags_df.groupby('hashtags_list').sum().sort_values(by='total_count', ascending=False).reset_index()[:10]
```


```python
hashtags_df
```

Finally, let's bring in our Plotly library to visualize our results in an interactive pie chart:


```python
import plotly.express as px
```


```python
fig = px.pie(hashtags_df, values='total_count', names='hashtags_list', 
             title='Hashtags', color_discrete_sequence=px.colors.sequential.RdBu)
fig.show()
```


```python

```
