# POLITICAL SENTIMENT ANALYSIS
*Angelo Turri*
_
*angelo.turri@gmail.com*

## Stakeholder & Problem
The stakeholder is a social media communications team working for a political candidate, Donald Trump. They have requested that you analyze a body of social media posts from their voter base and extract meaningful insights on their base's attitudes.


## Data Origin & Description

#### Price trend including 2008 market crash
![Price trend including 2008 market crash](/figures/trend_before.png)

All steps taken in this section were performed in [the "data_prep" notebook](https://github.com/Jellohub/political_sentiment_analysis/blob/master/notebooks/data_prep.ipynb).

### Data: Origin
Data is taken from the former reddit titled ***r/the_donald***. This reddit has been archived along with 20,000 others on [the-eye.eu](https://the-eye.eu/redarcs/). If you want to download it yourself, you just need to type "the_donald" in the search bar on this website and download the "Comments" link provided there. [The "data_prep" notebook](notebooks/data_prep.ipynb) uses the following URL to download (if you click it, the file will start downloading on your computer):

- https://the-eye.eu/redarcs/files/The_Donald_comments.zst

Fair warning - if you are about to explore on this website, be cautious. I looked at some of the archived reddits and will never be the same again.

### Data: Statistics
The compressed file is sizeable at 3.8GB, but this is in .zst format. Once converted to a .txt file, it takes up a whopping 37.48 GB of space, containing data on approximately 48 million posts. Due to the sheer amount of data and the limitations of my machine, I was unwilling to analyze all 48 million posts. I wanted to take 2 million posts, so I kept every 23rd post from this file.

Each post is recorded as a dictionary. Only some of the keys were relevant to our analysis:
- Raw text
- Post score (upvotes - downvotes)
- Author
- Date posted

After extraction, our initial dataframe had 2.1 million total entries ranging from August of 2015 to April of 2020, for a total of 1710 days – approximately 4.5 years. There are 178,308 unique authors.

## Preprocessing

All steps taken in this section were performed in [the preprocessing notebook](https://github.com/Jellohub/political_sentiment_analysis/blob/master/notebooks/preprocessing.ipynb).

### Removing hyperlinks and extraneous characters

- I used a regex to eliminate hyperlinks. The regex wasn't perfect, but it was very good. It reduced the number of posts with hyperlinks from 108,228 to just 229. That's a 99.8% reduction in posts with hyperlinks.
- There were several other combinations of characters we replaced, such as "&gt;" that repeated many times. These are longhand notations of certain special characters like ">" or "<". They can be looked up [on this website](https://www.htmlhelp.com/reference/html40/entities/special.html).

### Tokenizing posts
- Tokenizing strings is the process of separating them into lists, where each element is an individual word in that sring. I used another regex that identified words and eliminated special characters such as punctuation.
- Tokenization enables easy stopword removal and the creation of bigrams and trigrams.

### Stopword removal
- Stopwords are commonly used words that significantly increase the burden on models while contributing very little. We removed stopwords contained in nltk's stopword list. This resulted in the removal of 22.3 million words, about half of all original words.

### Calculating post lengths
- Enables the removal of insanely long posts and create more reasonably distributed data.

### Creating bigrams and trigrams
- Individual words are one means of extracting insights from this data, but bigrams and trigrams carry more information than individual words. By extracting bigrams and trigrams, we have two more options for analysis and the potential to give more meaningful recommendations to our stakeholder.

### Removing bots
- All posts from the universal reddit moderator, AutoModerator, were removed.
- Posts with the words "bot made by u/reddit_username" were removed.
- Authors with the word "bot" or "Bot" in their name were removed.
- Posts with the words "I'm a bot" were removed.

These measures weren't perfect - they probably didn't remove all the bots, and they did remove some posts by real people. However, the vast majority of posts removed by these measures were made by bots.

If you're skeptical about the effectiveness of these measures, or you think they might have been too extensive, head to the Appendix at the end of this notebook. It shows you the kinds of posts each measure removed. These queries take up quite a bit of space, which is why I kept them at the end of the notebook.

### Capping post length at 200 words
- Looking at the distribution of post length, it was extremely skewed to the right. Capping post length at 200 words results in a much more reasonable distribution.

### Giving post score a hard boundary of (-25, 100)
- Looking at the distribution of post scores, it had long tails to both the right and left. A lower boundary of -25 and an upper boundary of 100 returned the distribution to a more reasonable state.

### Spam removal
- Removing posts longer than 200 words eliminated some spam, but not all.
- I used the most common unigrams, bigrams, and trigrams to locate any other spam.
    - Posts containing a unigram four or more times in a row were removed.
    - Posts containing a bigram three or more times in a row were removed.
    - Posts containing a trigram two or more times in a row were removed.
- Even after all these measures, trigrams such as "source script protect" continued to appear in the most common phrases. I manually removed these.
- All combined spam from this notebook is stored externally in a single dataframe, "spam.parquet."

# Feature Enginnering

Technically, we did minor feature engineering in [the preprocessing notebook](https://github.com/Jellohub/political_sentiment_analysis/blob/master/notebooks/preprocessing.ipynb) with bigrams and trigrams. All major feature engineering occurs in [the feature engineering notebook](notebooks/Technically, we did minor feature engineering in [the preprocessing notebook](https://github.com/Jellohub/political_sentiment_analysis/blob/master/notebooks/preprocessing.ipynb) with bigrams and trigrams. All major feature engineering occurs in [the feature engineering notebook](notebooks/https://github.com/Jellohub/political_sentiment_analysis/blob/master/notebooks/feature_engineering.ipynb).
).

### Sentiment analysis

Sentiment analysis was done with VADER (Valence Aware Dictionary sEntiment Reasoner). I discovered VADER through [this video by Rob Mulla](https://www.youtube.com/watch?v=QpzMWQvxXWk). Vader uses [a lexicon of about 7,500 words](https://github.com/cjhutto/vaderSentiment/blob/master/vaderSentiment/vader_lexicon.txt) to calculate the probability of a piece of text being postive, negative or neutral. It also gives a compound score of sentiment based on these three probabilities, which is what we use to score sentiment.

[Neptune.ai](https://neptune.ai/blog/sentiment-analysis-python-textblob-vs-vader-vs-flair#:~:text=Valence%20aware%20dictionary%20for%20sentiment,to%20calculate%20the%20text%20sentiment.&text=Positive%2C%20negative%2C%20and%20neutral.) describes VADER as being "optimized for social media data and can yield good results when used with data from twitter, facebook, etc." Since Reddit counts as social media, I believe VADER is a good choice for this project.

### Bag of words features

We have both of the target variables we need - score and sentiment. We are unable to use models meaningfully on our data as it currently stands. To create proper training data:

- I limited my vocabulary to the top 100 unigrams, bigrams, and trigrams. This was necessary because keeping all the words would have resulted in too many features.
- For each word/phrase in our vocabulary, I went through each post and determined whether the word/phrase was present.
- The result was three sparse matrices for unigrams, bigrams and trigrams.
- Due to limiting our vocabulary, many of the rows in these sparse matrices consisted entirely of 0's (that is, none of the words in our vocabulary were present in the given post). These rows do nothing but dilute the data and impeded the model's ability to make predictions, so all these rows were removed.

## Metrics
I use the MAE (Mean Absolute Error), which averages the number of times someone made an error.
I also use the Rsquared; the R-squared value tells you much of the actual values' variance a model can explain by using its odwn model.

# Modeling

All modeling was done in [the modeling notebook](https://github.com/Jellohub/political_sentiment_analysis/blob/master/notebooks/modeling.ipynb). I only include the best models and their resultsAll other models are left in [the modeling notebook](https://github.com/Jellohub/political_sentiment_analysis/blob/master/notebooks/modeling.ipynb).

The model of choice was Linear Regression. The advantages of this model for this project are:

- It provides coefficients for all independent variables, making it interpretable;
- The coefficients can be positive or negative, so we know whether the independent variable positively or negatively impacted the target

We fit one model for each set of features, so we could analyze the model coaefficients separately for single-, two-, and three-word phrases.

### First Iteration

We started by using the "score" target variable. The results were very poor, with R-squared values of <0.1 for each model. These are so abysmal that I am not going to go into details for any of these models.

### Second Iteration

The second iteration of modeling used the "vader" target variable. The results were much better:

- Test R-squared for unigrams model: 0.245
- Test R-squared for bigrams model: 0.166
- Test R-squared for trigrams model: 0.326

Mean Absolute Error results:

- Test MAE for unigrams model: 0.33
- Test MAE for bigrams model: 0.433
- Test MAE for trigrams model: 0.363

### Third Iteration

The third iteration of modeling also used the "vader" target variable, but I eliminated all neutral scores from the dataset. The results were marginally better for R^2 but the MAE scores got slightly worse:

- Test R-squared for unigrams model: 0.249 (+0.003)
- Test R-squared for bigrams model: 0.168 (+0.002)
- Test R-squared for trigrams model: 0.344 (+0.018)

Mean Absolute Error results:

- Test MAE for unigrams model: 0.408 (+0.078)
- Test MAE for bigrams model: 0.468 (+0.035)
- Test MAE for trigrams model: 0.406 (+0.043)

### Fourth Iteration

The fourth iteration of modeling performed a hyperparameter grid search with the much faster SGDRegressor (also a linear regression model – just faster than the OLS version, and without significance values for the coefficients). The same data in the previous iteration was used for this one (with neutral VADER sentiment values removed). This iteration did not yield any improvements. All values were the same or slightly worse.

- Test R-squared for unigrams model: 0.249 (+0.00)
- Test R-squared for bigrams model: 0.168 (+0.00)
- Test R-squared for trigrams model: 0.333 (-0.011)

Mean Absolute Error results:

- Test MAE for unigrams model: 0.408 (+0.00)
- Test MAE for bigrams model: 0.469 (+0.001)
- Test MAE for trigrams model: 0.424 (+0.018)

## Methods Justification & Value to Stakeholder

The Linear Regression that we use provides a series of positive and negative coefficientsth that say how well thye are able to control / dominate territory.
    
## Limitations

We only use one social media platform: Reddit.
Furthermore, we only use one subreddit.
So I would say our data could be considered limited in scope.

## Recommendations

## Repository Structure
- **[data](https://github.com/Jellohub/political_sentiment_analysis/tree/master/data)**: folder containing data files, if any.
    - [**training_data**]: the target variables and features required for the linear regression models.
- **[notebooks](https://github.com/Jellohub/political_sentiment_analysis/tree/master/notebooks)**: folder containing all four secondary notebooks. These are only there for anyone who wants technical details.
- **[visualizations](https://github.com/Jellohub/political_sentiment_analysis/tree/master/visualizations)**: folder containing all visualizations used in the presentation and README.
- **[final notebook](https://github.com/Jellohub/political_sentiment_analysis/blob/master/notebook.ipynb)**: project notebook containing all data imports, preprocessing, feature engineering, models, and visualizations.
- **[presentation.pdf](https://github.com/Jellohub/political_sentiment_analysis/blob/master/presentation.pdf)**: pdf file with all presentation slides.
- **[presentation pdf](https://github.com/Jellohub/political_sentiment_analysis/blob/master/presentation.pdf)**: PDF version of presentation slides.