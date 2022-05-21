---
layout: post
title: "Twitter sentiment analysis in Python"
subtitle: "Combining the power of Selenium and RoBERTa"
background: '/img/posts/twitter_analysis/jeremy-bezanger-unsplash.jpg'
---

Social media are now the main source of information for many people. In fact as of February 2021, 38% of adults use social media as a source of news in France, and 76% in Kenya. Regardless of the veracity of the news in twitter, it is a great place to get general news, and especially what people think about it.\
So today I asked myself : what is going on in the world, and are the news great or not. To answer these questions, we need 2 things : 
- a way to collect tweets from tweeter
- a way to determine if the tweets are conveying a general positive sentiment or not. 

You guessed it, in today's post we are going to use both a twitter scraper and a sentiment analysis tool. In our case, the scraper will be hand-made (you will see why), and as for the sentiment analysis model we will use RoBERTa-Large.

## 1 - Collecting tweets 

Several solutions exist to collect information from twitter, however some of them are just awful. Forget about manual collection, because it will just take to much time 😅.
\
A free twitter API does exist, and in fact it may usable for this project in particular, but it also comes with annoying limitations such as a limit on the number of request, and very few settings regarding the time period of tweet collection. \
Then, multiple twitter scrappers written in Python exist. I tried 3 of them : twitterscraper, twitter_scrapper, and twint. Well guess what : None of them worked, at least for me. Is it my fault? Maybe, but in the meantime I was left with 0 tweets in my possession. \
Of course, commercial solutions also exist, but I was not ready to spend 100€ for this little project (some advanced solutions cost up to 500€/month 😯). \
\
I was running out of ideas, when I remembered that what I wanted was not that complicated : I wanted a scrapper that could extract text from a page and do basic interaction with it. In fact, I know a library that is made for this... Please say hello to ... *Selenium* once again ! 
\
\
We already did some selenium in a previous post, so the syntax should be familiar now 😁 \
Notice that for this tutorial, a [Chrome WebDriver](https://chromedriver.chromium.org/) needs to be installed

### Extracting trends


```
from selenium import webdriver
from selenium.webdriver.common.keys import Keys
import time
import re
import clipboard

driver_path = 'C:\Program Files (x86)\chromedriver.exe'
base_url = 'https://mobile.twitter.com/i/trends'

options = webdriver.ChromeOptions()
options.add_argument("--start-maximized")

driver = webdriver.Chrome(executable_path=driver_path, options=options)
driver.get(base_url)

time.sleep(3)
trends = driver.find_elements_by_xpath("//div[@data-testid='cellInnerDiv']")
for trend in trends:
    txt_ = trend.get_attribute('innerHTML')
    extracted_trends = re.findall(">(.{1,20})</span>",txt_)
    if len(extracted_trends)==3:
        subject = extracted_trends[1]
        print(subject)
```

    #TheVoice</span>
    Qatar
    Parc
    Vélodrome
    Zidane
    Loris
    #TPMPPeople</span>
    Auteuil
    Ben Yedder
    Harit
    Milik
    Fred Hermel
    Moussa
    Triangle of Sadness
    Bondy
    

Here we extract the trends from the [twitter trends](https://mobile.twitter.com/i/trends). Altough the procedure is very similar what we did in the previous post, the overall html file is so messy ! So many nested divs and elements with seemingly random class names... \
So instead of trying to locate precisely the trend text, I decided to use regular expressions. They are a very efficient and useful way of finding patterns in text [Regex 101](https://regex101.com/) seems to be a great website to build regular expressions. \
Here we notice that there is a problem when the trend starts with #, but we can easily fix it if we want in a post process.

### Extracting tweets


```
def send_new_subject(driver, subject):
    text1 = driver.find_element_by_xpath("//label[@data-testid='SearchBox_Search_Input_label']")
    text1.click()
    time.sleep(1)
    clear_button = driver.find_element_by_xpath("//div[@data-testid='clearButton']").click()
    input_area = text1.find_element_by_xpath(".//input")
    clipboard.copy(subject)
    input_area.clear()
    input_area.send_keys(Keys.SHIFT, Keys.INSERT, Keys.ENTER)
    return driver

def append_tweets(tweets, tweets_list):
    for tweet in tweets:
        elts = tweet.find_elements_by_xpath(".//span")
        tweet_text = ""
        for elt in elts:
            txt = elt.get_attribute('innerHTML') 
            if len(txt)>20 and len(re.findall("</|@",txt))==0 and len(txt)<150:
                tweet_text += txt+" "
        if len(tweet_text)>0 and tweet_text not in tweets_list:
            tweets_list.append(tweet_text)
    return tweets_list

N_TWEETS = 200
i = 0
dict_ = {}
for trend in trends:
    txt_ = trend.get_attribute('innerHTML')
    extracted_trends = re.findall(">(.{1,20})</span>",txt_)
    if len(extracted_trends)==3:
        subject = extracted_trends[1]
        print(subject)
        if i==0:
            trend.click()
        else:
            send_new_subject(driver, subject)
        time.sleep(2)
        tweets_list = []
        while len(tweets_list)<N_TWEETS:
            nb_tries = 0
            while nb_tries<3 and len(tweets_list)<N_TWEETS:
                try:
                    tweets = driver.find_elements_by_xpath("//div[@aria-label='Timeline: Search timeline']/div/div")
                    print(len(tweets), len(tweets_list))
                    tweets_list = append_tweets(tweets, tweets_list)
                    driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
                    time.sleep(2)
                except:
                    print('retrying')
                    time.sleep(1)
                    nb_tries+=1
        dict_[subject] = tweets_list
        break
        i += 1
```

    TheVoice
    4 0
    [...]
    25 187
    

Now we collect the tweets from the trends. The scrapping process requires a bit of trial and error. The code is designed to collect all the tweets in an automatic manner altough we could do some things manually. I used a Selenium command to scroll the twitter page and load more tweets, otherwise we would not be able to collect enough tweets. For time purposes I chose to collect the 200 top tweets from each trend (we could collect much more).

## 2 - Sentiment analysis

Collecting the tweets was not an easy task but it was only one part of the project. We want to use the tweets to know whether a news is positive or not.
\
We can use basic NLP associated with classical machine learning to perform sentiment analysis by using algorithms such as ,Linear Regression, Naive Bayes or Support Vector Machines, but now deep learning has proven to be a better option when it comes to dealing with unstructured data (such as text). \
BERT (**B**idirectional **E**ncoder **R**epresentations from **T**ransformers) is such a model, that is pre-trained on a very large corpus in a semi-supervised manner and has the advantage of using the context given by a sentence to get better result. 
\
RoBERTa is trained in a slightly more robust way and on much more data, which explains why it outperforms BERT in NLP tasks. We are going to use HuggingFace for the sentiment analysis task.


```
from transformers import AutoTokenizer, AutoModelForSequenceClassification
from scipy.special import softmax
import numpy as np

roberta = 'cardiffnlp/twitter-roberta-base-sentiment'
model = AutoModelForSequenceClassification.from_pretrained(roberta)
tokenizer = AutoTokenizer.from_pretrained(roberta)
labels = ['Negative','Neutral','Positive']

def process_tweet(tweet):
    tweet_words = []
    for word in tweet.split(' '):
        if word.startswith('@') and len(word) > 1:
            word = '@user'
        elif word.startswith('http'):
            word = "http"
        tweet_words.append(word)
    tweet_proc = " ".join(tweet_words)
    return tweet_proc

def analyse_tweet(tweet_proc):
    encoded_tweet = tokenizer(tweet_proc, return_tensors='pt')
    output = model(**encoded_tweet)
    return softmax(output[0][0].detach().numpy())

for trend in dict_.keys():
    sentiments = {0:0,1:0,2:0}
    for i in range(len(tweet_list)):
        tweet = tweet_list[i]
        processed_tweet = process_tweet(tweet)
        scores = analyse_tweet(processed_tweet)
        sentiments[np.argmax(scores)]+=1
    print(trend)
    for i in range(3):
        print(f"{labels[i]} : {sentiment(i)} %")
```

    TheVoice 
    Negative : 8 
    Neutral : 176 
    Positive : 24

The sentiment analysis may take quite some time. As we can see through this one example, most tweets are neutral, which was to be expected. If we want a more precise prediction, we must collect more tweets.

## Conclusion

In this post, we saw how to use selenium to collect trends and tweets from twitter, and then perform sentiment analysis using state of the art NLP algorithms. In a more advanced project, we could add topic modelling to try to find why people may be happy or not about a certain subject, and we could try to find if links exist between different trends.


Credits : Photo by Jeremy Bezanger on [Unsplash](https://https://unsplash.com/)