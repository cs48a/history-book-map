# History book map

The goal of this project is to show how frequently the (current) countries occure in history books.

The project uses basic NLP on the text of an official Hungarian history book (11th grade).

![History book map](https://cs48a.github.io/history-book-map/map.png "The colormap of the entities in the book")

An interactive plot is here: <https://cs48a.github.io/history-book-map/map.html>

## Problem statement

Given a text containing several named entities (countries, geographic areas, people, etc.) find a mapping between those entities and the names of currently existing countries. Assuming that each entitiy can be assigned to at least one currently existing country.
By applying the mapping create a map which colors the countries according to the number of occurances of each country in the text.

The expected result is that a Hungarian history book will be mostly centered around Hungary (and its historical area), but who knows maybe some interesting results will show out. Additionally, if the method can be applied without much effort to the 9th, 10th and 12th grade books as well, then it would show how the focus of the history shifts along centuries.

## Get the book

The book can be downloaded from here: <https://www.tankonyvkatalogus.hu/site/kiadvany/FI-504011101_1>

The book is 258 pages long and around 54MB.

## Extract the text from the book

Extracting the text requires a pdf converter which can extract textual data from a pdf. This is not trivial since the pdf standard is quite complex and was probably not constructed by having this kind of reverse engineering in mind.

I tried multiple tools but Pdfminer <https://github.com/pdfminer/pdfminer.six> seemed to be the most efficient/easy to use. It is able to read a document page by page and return a string of the whole text.

Possible issues here:
* Page- and linebreaks split words up
* Issues with special characters
* Issues with odd text placement and letter spacing (e.g., title over a map, etc.)

The resulting text file is 33659 lines long and contains around 927000 characters.

## Extract named entities

The point of interest is to identify each named entity in the text which can be assigned to a current country. For example Napoleon -> France, Prussia -> Germany.

Things to consider:
* Should historical people be included or only geographic entities?
* Should it be 1 to 1 mapping or 1 to multiple? (e.g., Prussia -> [Germany, Poland])

In order to extract the named entities from the text the HuSpaCy NLP library <https://github.com/huspacy/huspacy> was used. This is a quite recent library which has its sole focus on the Hungarian language. Hungarian is a so called agglutantive language <https://en.wikipedia.org/wiki/Agglutinative_language> which means that a word (even a named entity) can be extended with multiple suffixes depending on its role in a sentence (e.g., Austria is written as "Ausztria" in Hungarian, but "in Austria" is written as "Ausztriában" and to refer Austria as an object--as in "Austria (was attacked by France)"-- it is written as "Ausztriát (megtámadta Franciaország)".

So recognizing the named entities ("Ausztriában") is the first task and getting the lemma ("Ausztria") out of them is the second. Furtunately HuSpaCy can offer both. (It also says about itself that it is industrial grade and has CPU archicteture-specific optimization, but these were not the main aspects considered at the choice.)

### First check

With the dictionary of the counted lemmas a first check of the results can be done. For the sake of simplicity and further manipulation the dictionary is converted into a DataFrame.

Let's sort the DataFrame in a descending order and see how it looks, which are the most frequent entities. (Note that the book covers history between 1849-1945)

At first not much suprise here: lot of Hungary, Germany, France and Europe. Apparently Stalin ("Sztálin" in Hungarian) lost his last letter in the lemmatization and ended up as "Sztáli". Not what I would call historical retribution, but attention has to be paid to such details. The letter "R" as an independent lemma appears 32 times which is quite unfortunate.

An ascending sort can also give an insight: many entities only occure once and many of them are either meaningless (probably results of false tokenization/lemmatization) or they are not relevant (e.g., ISBN number and authors of the book etc.)

Plotting the occurances on a histogram also confirms this: it shows a very skewed graph with over 2000 entities which have only 1 occurance. Since the point of the whole project is to create a map showing countries, which the entities most frequently associated with, therefore it is reasonable to drop these singletons. At this point there is no intent to automate mapping of the entities to countries, so the less entities the better anyway.

![Entity histogram](https://cs48a.github.io/history-book-map/hist.png "Histogram of entity occurances")

## Clustering words before manual input

Unfortunately (to my knowledge) there is no obvious or out-of-the-box solution for mapping the entities to countries. So manual data input seems inevitable, but it could be helped by groupping the similar entities together. Since no additional context is used at this point, the similarity is strictly character-based. This is, however, could help group those entities together which are basically the same, but were 'broken' either by the original text (e.g., "Amerikai Egyesült Államok" /USA/ vs "Egyesült Államok" /US/) or during the text processing/lemmatization (e.g., "Amerikai Egyesült Állam" /United State of America/ (sic!) vs. "Amerikai Egyesült Államok" /United States of America/).

Using the rapidfuzz <https://github.com/maxbachmann/RapidFuzz> library the distance matrix for the remianing 817 entities was calculated. The applied metric was the partial similarity ratio. The partial similarity ratio "searches for the optimal alignment of the shorter string in the longer string and returns the fuzz.ratio for this alignment". This is suitable for handling the above mentioned cases where an entity is a partial case of another. (But it could lead to some funny cases where "DÁNIA" /DENMARK/ is groupped together with JORDÁNIA /JORDAN/.)

Using the distance matrix, entities were grouped together by hierarchical clustering <https://scikit-learn.org/stable/modules/generated/sklearn.cluster.AgglomerativeClustering.html>. As the overall number of clusters is not known beforhand, hierarihcal clustering with a properly selected distance threshold can give a good starting point.

## Manual input

The clusterint resulted in 635 cluster labels instead of the 817 different words. Sorting the dataframe according to assigned labels will help during the manual input of the assigned countries to each cluster since similar words are now grouped together.

For the manual input phase the following guidelines were followed

* Empires and larger unions will be represented by their most dominant state as well as occupied states won't be displayed (Ottoman Empire will only be displayed as Turkey)
    * British Empire: United Kingdom
    * Soviet Union: Russia
    * Habsburg Empire, Austro-Hungarian Monarchy: Austria
* Notable battlefields or historical places which are mostly known because of another countries actions will be represented by both countries:
    * Auschwitz, Sobibor, Majdanek : Germany, Poland
    * Midway: USA, Japan
    * Dunqurque: France, UK, Germany
* Geographical areas will be left out
    * Continents and sub-continents (e.g., Europe, South-East Asia)
    * Rivers (Danube, Nile, etc.)
    * Exceptions: Balkan peninsula
    
    
It was decided that historical people will be included and multiple countries (at most 10) will be allowed. The result of the manual input is in "countries.csv". It has the following structure:
(id, count, Entity, Country1, ... , Country10, Countries), where the last column (Countries) contain the allowed names of the countries.
    
### Possible issues

Some issues with the text processing already turned out:

* Wrong split during tokenization: e.g., Nagy-Britannia --> "Nagy", "Britannia"
* People with a family name that is also a city: e.g., "Váradi Péter" -> "Várad", "Péter"
* Multiple cities with the same name
* Mixture of hungarian and local names (Bécs vs Wien)
* Countries which did not exist or were part of another country (e.g., Czechia, Slovenia, etc)

## Display the data

A geojson file and Plotly's choropleth_mapbox feature was used for plotting <https://plotly.com/python/mapbox-county-choropleth/>.

The map uses logirathmic scale for the coloring, otherwise it would be less interesting with just a few countries shining bright.

## Conclusions

There is not much surprise in the results. The book is centered around Hungary and the map also refelects the dominant powers of the covered period (1849-1945): Germany, UK, Russia, USA.
 
It has to be mentioned that the period is somewhat arbitrary defined by the authors of the book and chapters focusing on solely on Hungary were not separated from international history. It might be worthy to try to perform the analysis on distinct chapters or on a period which is historically more meaningful.

A more interesting problem is the assignment of the current countries. Countries which were part of a larger Empire/country in the longer part of the discussed period (e.g., Slovenia) are necessarily under-represented (Belarus was not mentioned once). Even in this more recent period it is often debatable which country to assign to a given historical country or city. This could be even harder as more older periods are analyzed. For example should ancient Rome be assigned to all the current mediterranean countries or only to Italy?

It was interesting to do this experiment. There are really cool tools available, the manual input took around 2 hours.
