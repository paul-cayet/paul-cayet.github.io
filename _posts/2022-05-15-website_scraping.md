---
layout: post
title: "Improving our vocabulary with Web Scraping "
subtitle: "Python and Selenium tutorial"
background: '/img/posts/web_scrapping/florian-olivo-unsplash.jpg'
---


Quite a few options are accessible to improve our vocabulary, but they may not be convenient. In this tutorial, we will be using some web scraping tools to build a database with the most frequent dictionnary request on Linguee. 

![url_example.png](\img\posts\web_scrapping\url_example.png)

## 1. Extracting vocabulary

We can access the most frequent Italian dictionary requests by clicking on the different numbers. For instance, let's try [the first one](https://www.linguee.com/italian-english/topitalian/1-200.html) (1-200). \
We can notice a few things. First of all, the URL consists of a base URL + some additionnal information about the frequency. \
Its simple structure will allow us to simply parse through the different pages. 

![first_voc.png](\img\posts\web_scrapping\first_voc.png)

Secondly, we can see that the proposed structure of vocabulary follows some sort of list or table. Let's go deeper by taking a look at the html code.

![voc_html.png](\img\posts\web_scrapping\voc_html.png)

By inspecting the first element, we get the corresponding part of the HTML code. The words are contained in a section with the class name "text top", and more specificaly in a table cell tag. \
Now that we obtained all those informations, it's time to build our scraper !

The web scraping tool of choice for this tutorial will be Selenium. It is simple to use and is extremely powerful when it comes to web automation. Other solutions exist (for instance BeautifulSoup) but they do not allow for interactive web scraping. \
\
Notice that for this tutorial, a [Chrome WebDriver](https://chromedriver.chromium.org/) needs to be installed.


```python
from selenium import webdriver

driver_path = 'C:\Program Files (x86)\chromedriver.exe'
base_url = 'https://www.linguee.com/italian-english/topitalian/'
frequency_url = '1-200'
driver = webdriver.Chrome(executable_path=driver_path)
driver.get(base_url + frequency_url + '.html')

first_word = driver.find_element_by_xpath("//div[@class='text top']/table/tbody/tr/td/a")
first_word.get_attribute('innerHTML')
```




    'no'



Fantastic ! (by the way it is interesting to see that the most frequent request in italian Linguee is "no" 😃 ) \
More seriously what we did here is quite simple : 
1. We launch an automated chrome tab with *webdriver.Chrome()*
2. Then we use the function *find_element_by_xpath* to go inside the structure of the HTML pages with the selenium function

The X_PATH is a simple way to locate an element. Here, we used the section with class name "text top" as the parent element, and went down from it. \
When the element is located, the work is almost done, we just extract the corresponding word. Here, we use *innerHTML* attribute to get the source of the content of the element. \
\
Now for the real part, we do this for every words in the page for the 5000 most frequent words :


```python
import time
import random
RTIMES = [2.1, 2, 2.5, 2.2, 1.8]

vocabulary_list = []
base_url = 'https://www.linguee.com/italian-english/topitalian/'
frequency_urls = ['1-200','201-1000','1001-2000','2001-3000','3001-4000','4001-5000']
driver = webdriver.Chrome(executable_path=driver_path)

for freq_url in frequency_urls:
    print(freq_url)
    driver.get(base_url + freq_url + '.html')
    words = driver.find_elements_by_xpath("//div[@class='text top']/table/tbody/tr/td/a")
    vocabulary_list.extend([word.get_attribute('innerHTML') for word in words])
    time.sleep(random.choice(RTIMES))
driver.close()
```

    1-200
    201-1000
    1001-2000
    2001-3000
    3001-4000
    4001-5000
    

We add a short random wait before loading the next page to reduce the risk of being detected as a bot.\
And the length of the vocabulary list is indeed 5000, which is what we wanted. We can save the contents of the list, and now we are going to translate those words, because here we only did half of the job.


```python
with open('vocab.txt', 'w') as f:
    for line in vocabulary_list:
        f.write(line)
        f.write('\n')
```

## 2. Translating the vocabulary

Now, let's translate the obtained vocuabulary. We can still use Linguee, but we will have to translate it one word at a time. In order to do this in a simpler way, we will use its translation counterpart, DeepL.


```python
driver = webdriver.Chrome(executable_path=driver_path)
driver.get("https://www.deepl.com/fr/translator#it/fr/")

sentence = 'no \nen \nlo \ncarne \ntesta'
input_area = driver.find_element_by_xpath("//textarea[@class='lmt__textarea lmt__source_textarea lmt__textarea_base_style']")
input_area.clear()
input_area.send_keys(sentence)
time.sleep(2)
translated_text = driver.find_element_by_xpath("//textarea[@class='lmt__textarea lmt__target_textarea lmt__textarea_base_style']")
translated_text.get_attribute('value')
```




    'pas de \nsur \nlo \nviande \ntête'



Once again the previous code shows how to translate a few words using DeepL translation. Here of course the base URL is different, and so are the located elements. Here is how we proceed :
1. We locate the textarea element (where we write the text to be translated)
2. We write the text using the *send_keys()* method
3. We locate the translated text element and retrieve the text using *get_attribute()*

Now we translate the whole previously found vocabulary using the same principles. We just need to keep in mind that there is a translation limit of 5000 characters within DeepL.


```python
with open('vocab.txt') as f:
    lines = f.readlines()
i = 0
Groups = []
while i<len(lines):
    L = []
    s = 0
    while s<4500 and i<len(lines):
        L.append(lines[i])
        s+=len(lines[i])
        i+=1
    Groups.append(L)
```

To overcome this limitation we just split the vocbulary into lengths that are compatible with the free version of DeepL


```python
from selenium.webdriver.common.keys import Keys
import clipboard

driver = webdriver.Chrome(executable_path=driver_path)
driver.get("https://www.deepl.com/fr/translator#it/fr/")
translated_vocabulary = ""

for words in Groups:
    sentence = ''.join(words)
    input_area = driver.find_element_by_xpath("//textarea[@class='lmt__textarea lmt__source_textarea lmt__textarea_base_style']")
    clipboard.copy(sentence)
    input_area.clear()
    input_area.send_keys(Keys.SHIFT, Keys.INSERT)
    nextTranslation = False
    while nextTranslation == False:
        translated_text = driver.find_element_by_xpath("//textarea[@class='lmt__textarea lmt__target_textarea lmt__textarea_base_style']")
        temp_text = translated_text.get_attribute('value')
        if(temp_text != '' and temp_text[0:7] != 'n\n[...]'):
            translated_vocabulary += temp_text
            nextTranslation = True
        else:
            time.sleep(1)
    time.sleep(random.choice(RTIMES))
```

The code is pretty similar to what was done for the simple translation. Notice that we didn't use the *send_keys* method in the same way than previously. In fact, send keys works just like a regular person typing on a keyboard. Using it for typing 5000 characters is slow, so instead we implement what a human would do : copy and paste. \
Then, translating 5000 characters with DeepL is not instantaneous. Therefore we need to implement a way to wait before copying the translated text. I used the fact that when DeepL is translating, the first characters in the text_area are "[...]". \
Finally, when all the words have been translated, we store them in a csv file with the original Italian words.


```python
import csv
tr_voc = translated_vocabulary.split('\n')
with open('vocabulary.csv', 'w', newline='') as csvfile:
    writer = csv.writer(csvfile)
    writer.writerow(['italian']+['french'])
    for i in range(len(lines)):
        writer.writerow([lines[i].strip('\n')]+[tr_voc[i]])
```

We finally did it ! 
Let's resume what we learned in this short tutorial :
1. We learned how to perform scraping on Linguee using Selenium
2. We learned how to interact with the DeepL web page to automatically translate some text

But this is not the end... In part 2 we will see how to use the vocabulary to build a Quizlet-like learning path to learn new vocabulary in an optimal way, so stay tuned !

Credits : Photo by Florian Olivo on [Unsplash](https://https://unsplash.com/)
