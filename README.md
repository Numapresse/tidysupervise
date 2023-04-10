# TidySupervise

The TidySupervise project aims to fill a gap. Unsupervised text classification is already well-represented in the R text mining ecosystem with tools like stm, that have few equivalents in other programming languages. Supervised methods are not as prevalent and most existing packages, like RTextTools, are not readily compatible with the tidyverse.

Supervised models can still be very useful. Although they require some extensive data and corpora preparations before hands, this trade-off comes with significant advantages:
* Models are fixed and transferable.
* Model results do not require lengthy interpretation, since categories have been already defined.
* Applying the model is very quick and scale well with big corpora.

TidySupervise is still very much in the alpha stage: supervised classification is currently limited to SVM models.

You can install the package using:

```
devtools::install_github("Numapresse/TidySupervise")
```

This introduction vignette gives a thorough tour of all the functions, using a labelled dataset of French newspaper from the 1920s and the 1930s.

## Create a training corpus

Supervised models are fully appropriate when your are dealing with a lot of texts and want to go beyond an "exploratory" stage. TidySupervise was initially developed to classify millions of French newspaper articles published before 1945 and to extract corpora revolving a specific news genre (political news, stock exchange section, serial novels, weather report, and so forth…).

Training a text model is more an art than a science. You'll have to make a lot of subjective calls: whether a document belongs to a specific labelling category, whether this category should really exist at all, whether the document is classifiable in the first place. This is more akin to the work a literary critic than to a systematic scientific investigation.

The most practical way to label the data remains to put the training corpus in a spreadsheet file. If the corpus is already stored in a directory, you can use TidySupervise function tds_generate to create the corpus.

```{r eval=FALSE}
tds_generate(dir = "training_corpus")
```

The output will be a csv file with the same name as the directory, that can be opened by most spreadsheet applications. The name of each file will be reproduced in the "document" column and their content in the "text" column. An empty "label" column waits for your classification in the middle.

If the corpus is big (or use big-sized documents), you may find it more practical to only store the metadata in the csv file. Setting metadata_only to TRUE will create a csv file with only the Document and the empty "Label" columns.

```{r eval=FALSE}
tds_generate(dir = "training_corpus", metadata_only = TRUE)
```

With a big training corpus, it may be more advisable to only use a random sample. Beyond a certain size, supervised classification will pay off less: the slight gain in the quality of the results are imbalanced by the time it takes to label the corpus or to process the model. You can specify the number of documents you intend to use with random_document.

```{r eval=FALSE}
tds_generate(dir = "training_corpora", metadata_only = TRUE, random_document = 1000) #We will use only 1000 documents
```

## Prepare the corpus

Once the spreadsheet has been filled, you can reopen them like any structured data frame.

```{r eval=FALSE}
training_corpus = read_tsv("training_corpus.tsv") #If the file is a tsv.
training_corpus = read_csv("training_corpus.csv") #If the file is a csv.
```

In the special case where you have not kept the texts in the labelling dataset, you may retrieve them using tds_associate.

```{r eval=FALSE}
training_corpus = tds_associate(dir = "training_corpus", training_metadata = "training_corpus.csv")
```

Currently, our text dataset is at the *document level*. To make it usable for supervised classification we will have to move at the *token level*, thanks to unnest_token from TidyText. 

We are going to run a test with sample_classification.tsv. This file contain a random selection of 1500 text blocks from 15 issues of French newspapers published during the 1920s and the 1940s.

```{r}
library(tidysupervise)
library(readr)
library(tidytext)
training_corpus = read_tsv(system.file("extdata", "sample_classification.tsv.zip", package = "tidysupervise"), col_types = "ccc")
training_corpus = unnest_tokens(training_corpus, token, text)
training_corpus
```

Each occurrence of a token corresponds now to one line in a table. Several preparation steps remain necessary before the actual training. They can all be done using tds_process. The output is the number of occurrences per document and per word ponderated by term frequencies and inverse document frequencies (or tf_idf, using the bind_tf_idf function from tidy_text) and some cosine normalization. Concretely, it is not necessary to remove the "stop words": tds_process already weight the overtly frequent words. In some case, stop words can be actually meaningful. For instance, first-person pronouns may become fairly rare in more formal contexts: when trying to model newspaper genres, they turned out to be a pretty good predictor of the serial novel.

Besides, we set "training" to TRUE, simply in order to keep the label column at the end of the text processing. This will not be necessary when we will apply the model to a new set of data.

```{r}
training_corpus_processed = tds_process(training_corpus, training = TRUE)
training_corpus_processed
```

Since TidySupervise is compliant with the pipe symbol, you can chain all the steps in one batch.

```{r}
training_corpus = read_tsv(system.file("extdata", "sample_classification.tsv.zip", package = "tidysupervise"), col_types = "ccc")
training_corpus %>% 
  unnest_tokens(token, text) %>%
  tds_process(training = TRUE)
```

tds_process includes several parameters that can be customized. It is possible to lemmatize the texts if they are either in English, French, German, Spanish or Italian using lookup tables borrowed to Spacy. It can enhance significantly the quality of the classification since inflected forms may become more readily usable.

```{r}
training_corpus = unnest_tokens(training_corpus, token, text)
training_corpus_processed = tds_process(training_corpus, training = TRUE, lemmatization = "french")
training_corpus_processed
```

You can then text into continuous text segments (default is 100). This is recommended for training as the models can be affected by the use of documents of heterogeneous size. Longer segments may yield better results at the cost of less diversity (since there will be fewer segments).

```{r}
training_corpus_processed = tds_process(training_corpus, training = TRUE, segment_size = 100)
training_corpus_processed
```

Finally, when dealing with a bigger training corpus, you may want to have more control on the selection of tokens used for classification. By default, tds_process keeps the tokens that has appeared in at least two different documents and, at most, the 3000 more frequent token. You can specify both parameters:

```{r}
#Here we only keep token that have appeared in at least 4 different documents and, at most the 2000 more frequent token.
training_corpus_processed = tds_process(training_corpus, training = TRUE, segment_size = 100, min_doc_count = 4, max_word_set = 2000)
training_corpus_processed
```

Lowering max_word_set (or increasing the min_doc_count threshold) can quicken the training process with little quality loss by having smaller word/document matrix.

## Training the model

Now that the corpus has been processed, training the model is fairly straightforward. All you have to do is use tds_model. Notice it may take some time (not more than a few minutes).

```{r}
training_corpus_processed = tds_process(training_corpus, training = TRUE, lemmatization = "french", segment_size = 100)
corpus_model = tds_model(training_corpus_processed)
corpus_model
```

We can visualise the way words have been used to predict a label using tds_open. Besides the graph of the most "active" words per labels, tds_open return the probabilities for each possible pair of word and label.

```{r}
tds_open(corpus_model)
```

Once again, since TidySupervise is compliant with the pipe symbol, all theses functions can be chained.

```{r eval=FALSE}
word_model = tds_process(training_corpus, training = TRUE) %>%
  tds_model() %>%
  tds_open()
```

In a first phase you may want to evaluate the model using some part of the original corpus as a test. This is possible with the function tds_test. You may indicate the share of the corpus to be kept for classification using prop_train (the default is 80\%). Defining the good ratio is always a compromise: with a higher share of the corpus for training you will get a better model and with a higher share for testing, you will get more results.

```{r}
#Here we set prop_train to 85%, which leaves 15% of the original corpus for testing.
test_model = tds_process(training_corpus, training = TRUE, lemmatization = "french", segment_size = 100) %>%
  tds_test(prop_train = 85)
test_model
```

tds_test will print a message signaling the global results. Having look at the detailed results (especially at the mis-classification) can be very informative. We can check the "closest" labels, those that tend to overlap the most according to the model. Depending on the results, we may attempt to reclassify slightly our original corpus or consider this is part of some healthy intertextual relationship.
```{r}
test_model %>% 
  filter(result == 0) %>% #We only keep the "wrong" results
  group_by(original, prediction) %>% 
  summarise(count_misclassification = n()) %>% #We group and summarise all the possible combinations.
  arrange(-count_misclassification)
```

## Apply the model

Once you are satisfy with the model, you may attempt to predict the labels of a new corpus. The process is fairly quick and, providing you have enough RAM, you may safely attempt to tackle big set of texts. 

Here, we will only use a much smaller segment of 500 text blocks from 5 issues of French newspaper of the 20s.

Once again you will start by preparing the corpus, preferably using the same criteria as when you created the training corpus. Of course, this time, training will not be set to true (since we don't care about preserving a column label). You can also dismiss max_word_set by setting it to 0: we will anyway only reuse the words that have been used in the training phase.
```{r}
apply_corpus = read_tsv(system.file("extdata", "apply_classification.tsv.zip", package = "tidysupervise"), col_types = "ccc")
apply_corpus_processed = apply_corpus %>% 
  unnest_tokens(token, text) %>%
  tds_process(max_word_set = 0)
```

The model corpus_model can be applied to the corpus inside result_model using tds_apply. For each document you will get the probabilities of each label, ordered by the highest one. In a lot of cases, there will not be a "dominant" label: newspaper genre were not clear-cut and automated classification can give a better idea of the lexical "strength" of some classifications.

```{r}
result_model = tds_apply(apply_corpus_processed, model = corpus_model)
result_model
```
