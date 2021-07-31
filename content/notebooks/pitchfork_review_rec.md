---
title: 'Recommendations Based on Pitchfork Written Reviews'
date: 2020-07-23
draft: true
tags:
  - data science
  - data analytics
  - music discovery
  - recommender systems
---
# Introduction


## Setup
First, let's make sure we're in the correct working directory and have all the necessary packages imported.
We'll also read in the dataset--It's been cleaned beforehand on my local machine; a simple process to get it into a csv rather than the SQLLite db I had downloaded.
Finally, we'll set a flag to indicate which vectorizer we are using in the interest of testing both and comparing (A/B testing).


```python
%cd "drive/My Drive/Colab Notebooks"
!pwd
!pip3 install sklearn pandas rake_nltk
```

    /Users/tejassiripurapu/Notebooks
    Defaulting to user installation because normal site-packages is not writeable
    Requirement already satisfied: sklearn in /Users/tejassiripurapu/Library/Python/3.7/lib/python/site-packages (0.0)
    Requirement already satisfied: pandas in /Users/tejassiripurapu/Library/Python/3.7/lib/python/site-packages (1.0.5)
    Requirement already satisfied: rake_nltk in /Users/tejassiripurapu/Library/Python/3.7/lib/python/site-packages (1.0.4)
    Requirement already satisfied: scikit-learn in /Users/tejassiripurapu/Library/Python/3.7/lib/python/site-packages (from sklearn) (0.23.1)
    Requirement already satisfied: python-dateutil>=2.6.1 in /Users/tejassiripurapu/Library/Python/3.7/lib/python/site-packages (from pandas) (2.8.1)
    Requirement already satisfied: pytz>=2017.2 in /Users/tejassiripurapu/Library/Python/3.7/lib/python/site-packages (from pandas) (2020.1)
    Requirement already satisfied: numpy>=1.13.3 in /Users/tejassiripurapu/Library/Python/3.7/lib/python/site-packages (from pandas) (1.18.5)
    Requirement already satisfied: nltk in /Users/tejassiripurapu/Library/Python/3.7/lib/python/site-packages (from rake_nltk) (3.5)
    Requirement already satisfied: scipy>=0.19.1 in /Users/tejassiripurapu/Library/Python/3.7/lib/python/site-packages (from scikit-learn->sklearn) (1.4.1)
    Requirement already satisfied: threadpoolctl>=2.0.0 in /Users/tejassiripurapu/Library/Python/3.7/lib/python/site-packages (from scikit-learn->sklearn) (2.1.0)
    Requirement already satisfied: joblib>=0.11 in /Users/tejassiripurapu/Library/Python/3.7/lib/python/site-packages (from scikit-learn->sklearn) (0.15.1)
    Requirement already satisfied: six>=1.5 in /Applications/Xcode.app/Contents/Developer/Library/Frameworks/Python3.framework/Versions/3.7/lib/python3.7/site-packages (from python-dateutil>=2.6.1->pandas) (1.12.0)
    Requirement already satisfied: click in /Users/tejassiripurapu/Library/Python/3.7/lib/python/site-packages (from nltk->rake_nltk) (7.1.2)
    Requirement already satisfied: tqdm in /Users/tejassiripurapu/Library/Python/3.7/lib/python/site-packages (from nltk->rake_nltk) (4.47.0)
    Requirement already satisfied: regex in /Users/tejassiripurapu/Library/Python/3.7/lib/python/site-packages (from nltk->rake_nltk) (2020.6.8)



```python
import pandas as pd
from rake_nltk import Rake
from sklearn.metrics.pairwise import cosine_similarity, euclidean_distances
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.feature_extraction.text import TfidfVectorizer
```


```python
df = pd.read_csv("data/pitchfork_reviews_cleaned.csv")
print(df.head())

tfidf = True # if True: use TfidfVectorizer, else: CountVectorizer
```

                 artist                                       album       genre  \
    0       David Byrne  “…The Best Live Show of All Time” — NME EP        Rock   
    1         DJ Healer           Lost Lovesongs / Lostsongs Vol. 2  Electronic   
    2       Jorge Velez                                 Roman Birds  Electronic   
    3           Chandra                          Transportation EPs        Rock   
    4  The Chainsmokers                                    Sick Boy  Electronic   

       score             author                                             review  
    0    5.5          Andy Beta  Viva Brother, Terris, Mansun, the Twang, Joe L...  
    1    6.2        Chal Ravens  The Prince of Denmark—that is, the proper prin...  
    2    7.9   Philip Sherburne  Jorge Velez has long been prolific, but that’s...  
    3    7.8          Andy Beta  When the Avalanches returned in 2016 after an ...  
    4    3.1  Larry Fitzmaurice  We’re going to be stuck with the Chainsmokers ...  


## A Little Preprocessing
Now that our data is loaded into a dataframe, let's separate out the written review and use the album title as the index.

NOTE: The set we're working with may be too big to handle atm (session is crashing when trying to vectorize). Randomly sample 8000 reviews to begin with


```python
working_df = df[['artist', 'album', 'review']].copy()
# working_df.set_index('album', inplace=True)
print(working_df.head())
print(len(working_df))
working_df.dropna(inplace=True, how='any')
print(len(working_df)) # How many na rows were there? 5 (which ones?)
sample_df = working_df.sample(8000)
```

                 artist                                       album  \
    0       David Byrne  “…The Best Live Show of All Time” — NME EP   
    1         DJ Healer           Lost Lovesongs / Lostsongs Vol. 2   
    2       Jorge Velez                                 Roman Birds   
    3           Chandra                          Transportation EPs   
    4  The Chainsmokers                                    Sick Boy   

                                                  review  
    0  Viva Brother, Terris, Mansun, the Twang, Joe L...  
    1  The Prince of Denmark—that is, the proper prin...  
    2  Jorge Velez has long been prolific, but that’s...  
    3  When the Avalanches returned in 2016 after an ...  
    4  We’re going to be stuck with the Chainsmokers ...  
    20873
    20867


## Keyword Extraction

Use the nltk Rake function (from rake-nltk Python package) to extract important keywords from the reviews. This improves the vectorization by removing punctuation and allows for larger batch size by decreasing size of corpus and individual docs (reviews).

Furthermore, as a new Rake object is instantiated for every row in the dataframe, I've added the artist name as a stopword when extracting keywords for any given review of an artist. This is because the artist name is likely used frequently in a review of their music, so our similarity scores (and therefore recommendations) will be skewed towards albums by that artist. We want more variety to the recommendation, a user has likely already heard of other albums by the same artist--or can simply Google it.






```python
artists = sample_df['artist']

sample_df['keywords'] = ""

for idx, row in sample_df.iterrows():
  r = Rake(stopwords=artists[idx])
  r.extract_keywords_from_text(row['review'])
  keywords = list(r.get_word_degrees().keys())
  row['keywords'] = " ".join(keywords)

corpus = sample_df['keywords']
```

## Vectorization
The working dataframe is set up containing album titles (as index) and the full written review. We're going to vectorize that review column using Tfidf. This will give us a vector of word frequencies across the whole corpus. Think of this as mapping every review onto an n-dimensional space, where n is the number of distinct words in the corpus. Then we can measure similarity between the vectors using cosine similarity.
Admittedly, Tfidf is a little different as it measures frequency relative to how often that word appears in different documents, but this is a good way to conceptualize.



```python
if tfidf:
  tv = TfidfVectorizer()
  tv_counts = tv.fit_transform(corpus)
  # tv_counts = tv.transform(df['review'])
  tv_counts = tv_counts.toarray()
  print(len(tv_counts[0])) # How many words are being counted?
```

    89524



```python
if not tfidf:
    cv = CountVectorizer()
    cv_counts = cv.fit_transform(corpus)
    cv_counts = cv_counts.toarray()
    print(len(cv_counts[0]))
```


```python
labels = []
albums = sample_df['album']
for i in range(len(tv_counts[0])):
  labels.append("word" + str(i))
counts_df = pd.DataFrame(tv_counts, columns=labels, index=albums)
print(counts_df.head())
```

                                     word0  word1  word2  word3  word4  word5  \
    album                                                                       
    Pimps Don't Pay Taxes              0.0    0.0    0.0    0.0    0.0    0.0   
    Exmilitary                         0.0    0.0    0.0    0.0    0.0    0.0   
    Like Someone In Love EP            0.0    0.0    0.0    0.0    0.0    0.0   
    Expressions (2012 A.U.)            0.0    0.0    0.0    0.0    0.0    0.0   
    Songs from the Hermetic Theatre    0.0    0.0    0.0    0.0    0.0    0.0   

                                     word6  word7  word8  word9  ...  word89514  \
    album                                                        ...              
    Pimps Don't Pay Taxes              0.0    0.0    0.0    0.0  ...        0.0   
    Exmilitary                         0.0    0.0    0.0    0.0  ...        0.0   
    Like Someone In Love EP            0.0    0.0    0.0    0.0  ...        0.0   
    Expressions (2012 A.U.)            0.0    0.0    0.0    0.0  ...        0.0   
    Songs from the Hermetic Theatre    0.0    0.0    0.0    0.0  ...        0.0   

                                     word89515  word89516  word89517  word89518  \
    album                                                                         
    Pimps Don't Pay Taxes                  0.0        0.0        0.0        0.0   
    Exmilitary                             0.0        0.0        0.0        0.0   
    Like Someone In Love EP                0.0        0.0        0.0        0.0   
    Expressions (2012 A.U.)                0.0        0.0        0.0        0.0   
    Songs from the Hermetic Theatre        0.0        0.0        0.0        0.0   

                                     word89519  word89520  word89521  word89522  \
    album                                                                         
    Pimps Don't Pay Taxes                  0.0        0.0        0.0        0.0   
    Exmilitary                             0.0        0.0        0.0        0.0   
    Like Someone In Love EP                0.0        0.0        0.0        0.0   
    Expressions (2012 A.U.)                0.0        0.0        0.0        0.0   
    Songs from the Hermetic Theatre        0.0        0.0        0.0        0.0   

                                     word89523  
    album                                       
    Pimps Don't Pay Taxes                  0.0  
    Exmilitary                             0.0  
    Like Someone In Love EP                0.0  
    Expressions (2012 A.U.)                0.0  
    Songs from the Hermetic Theatre        0.0  

    [5 rows x 89524 columns]


## (Cosine) Similarity
Now we have a dataframe containing the word frequencies and we can compare each album using cosine similarity. Doing this in a pairwise manner (across every row in the dataframe) allows us to measure the similarity between two albums based on their review.


```python
similarity = cosine_similarity(counts_df)
print(similarity[:5])
```

God bless, everything ran and nothing has crashed yet (hopefully). Now let's make sense of the similarity matrix. Each album gets a row in the matrix (i.e. album[0] has a corresponding vector at similarity[0]).


## Recommendation and User Testing
Every value in the album vector represents how similar that album is with another. So for album[i], there is a corresponding album vector such that the value at index j in that vector is showing how similar album[i] is with album[j]. If i == j we should see a value of 1, meaning that the two albums are exactly alike (as they should be, it's the same album).

So to find recommendations for a given album, sort the corresponding similarity vector (while maintaining the index). Using the index of the highest value which isn't 1 will reference the album most similar to the given album based on the written Pitchfork review.

The process to recommend is as follows: Given an input album title -> get review corresponding to album, vectorize using Tfidf, measure similarity against sample, return top 5 (or 10?) most similar albums.


```python
working_df.set_index('album', inplace=True)
```


```python
search_album = "My Beautiful Dark Twisted Fantasy"
review = working_df.loc[search_album]['review']

vec = tv.transform([review]).toarray()
print(len(vec[0]))
search_sim = cosine_similarity(vec, counts_df)
print(search_sim)
```


```python
search_df = pd.DataFrame(search_sim[0])
top_5 = search_df.nlargest(5, 0)
# print(top_5.index)
for idx in top_5.index:
  # print(idx)
  print(albums.iloc[idx])
```
