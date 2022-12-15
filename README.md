# Pagerank Project

In this project, you will create a simple search engine for the website <https://www.lawfareblog.com>.
This website provides legal analysis on US national security issues.

**Due date:** Sunday, 18 September at midnight

**Computation:**
This project has low computational requirements.
You are not required to complete it on the lambda server (although you are welcome to if you'd like).

## Background

**Data:**

The `data` folder contains two files that store example "web graphs".
The file `small.csv.gz` contains the example graph from the *Deeper Inside Pagerank* paper.
This is a small graph, so we can manually inspect the contents of this file with the following command:
```
$ zcat data/small.csv.gz
source,target
1,2
1,3
3,1
3,2
3,5
4,5
4,6
5,6
5,4
6,4
```

> **Recall:**
> The `cat` terminal command outputs the contents of a file to stdout, and the `zcat` command first decompressed a gzipped file and then outputs the decompressed contents.

As you can see, the graph is stored as a CSV file.
The first line is a header,
and each subsequent line stores a single edge in the graph.
The first column contains the source node of the edge and the second column the target node.
The file is assumed to be sorted alphabetically.

The second data file `lawfareblog.csv.gz` contains the link structure for the lawfare blog.
Let's take a look at the first 10 of these lines:
```
$ zcat data/lawfareblog.csv.gz | head
source,target
www.lawfareblog.com/,www.lawfareblog.com/topic/interrogation
www.lawfareblog.com/,www.lawfareblog.com/upcoming-events
www.lawfareblog.com/,www.lawfareblog.com/
www.lawfareblog.com/,www.lawfareblog.com/our-comments-policy
www.lawfareblog.com/,www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
www.lawfareblog.com/,www.lawfareblog.com/topic/lawfare-research-paper-series
www.lawfareblog.com/,www.lawfareblog.com/topic/book-reviews
www.lawfareblog.com/,www.lawfareblog.com/documents-related-mueller-investigation
www.lawfareblog.com/,www.lawfareblog.com/topic/international-law-loac
```
You can see that in this file, the node names are URLs.
Semantically, each line corresponds to an HTML `<a>` tag that is contained in the source webpage and links to the target webpage.

We can use the following command to count the total number of links in the file:
```
$ zcat data/lawfareblog.csv.gz | wc -l
1610789
```
Since every link corresponds to a non-zero entry in the `P` matrix,
this is also the value of `nnz(P)`.
(Technically, we should subtract 1 from this value since the `wc -l` command also counts the header line, not just the data lines.)

To get the dimensions of `P`, we need to count the total number of nodes in the graph.
The following command achieves this by: decompressing the file, extracting the first column, removing all duplicate lines, then counting the results.
```
$ zcat data/lawfareblog.csv.gz | cut -f1 -d, | uniq | wc -l
25761
```
This matrix is large enough that computing matrix products for dense matrices takes several minutes on a single CPU.
Fortunately, however, the matrix is very sparse.
The following python code computes the fraction of entries in the matrix with non-zero values:
```
>>> 1610788 / (25760**2)
0.0024274297384360172
```
Thus, by using sparse matrix operations, we will be able to speed up the code significantly.

**Code:**

The `pagerank.py` file contains code for loading the graph CSV files and searching through their nodes for key phrases.
For example, you can perform a search for all nodes (i.e. urls) that mention the string `corona` with the following command:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --search_query=corona
```

> **NOTE:**
> It will take about 10 seconds to load and parse the data files.
> All the other computation happens essentially instantly.

Currently, the pagerank of the nodes is not currently being calculated correctly, and so the webpages are returned in an arbitrary order.
Your task in this assignment will be to fix these calculations in order to have the most important results (i.e. highest pagerank results) returned first.

## Task 1: the power method

Implement the `WebGraph.power_method` function in `pagerank.py` for computing the pagerank vector by fixing the `FIXME` annotation.

**Part 1:**

To check that your implementation is working,
you should run the program on the `data/small.csv.gz` graph.
For my implementation, I get the following output.
```
$ python3 pagerank.py --data=data/small.csv.gz --verbose
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=2.5629e-01
DEBUG:root:i=1 residual=1.1841e-01
DEBUG:root:i=2 residual=7.0701e-02
DEBUG:root:i=3 residual=3.1815e-02
DEBUG:root:i=4 residual=2.0497e-02
DEBUG:root:i=5 residual=1.0108e-02
DEBUG:root:i=6 residual=6.3716e-03
DEBUG:root:i=7 residual=3.4228e-03
DEBUG:root:i=8 residual=2.0879e-03
DEBUG:root:i=9 residual=1.1750e-03
DEBUG:root:i=10 residual=7.0131e-04
DEBUG:root:i=11 residual=4.0321e-04
DEBUG:root:i=12 residual=2.3800e-04
DEBUG:root:i=13 residual=1.3812e-04
DEBUG:root:i=14 residual=8.1083e-05
DEBUG:root:i=15 residual=4.7251e-05
DEBUG:root:i=16 residual=2.7704e-05
DEBUG:root:i=17 residual=1.6164e-05
DEBUG:root:i=18 residual=9.4778e-06
DEBUG:root:i=19 residual=5.5066e-06
DEBUG:root:i=20 residual=3.2042e-06
DEBUG:root:i=21 residual=1.8612e-06
DEBUG:root:i=22 residual=1.1283e-06
DEBUG:root:i=23 residual=6.1907e-07
INFO:root:rank=0 pagerank=6.6270e-01 url=4
INFO:root:rank=1 pagerank=5.2179e-01 url=6
INFO:root:rank=2 pagerank=4.1434e-01 url=5
INFO:root:rank=3 pagerank=2.3175e-01 url=2
INFO:root:rank=4 pagerank=1.8590e-01 url=3
INFO:root:rank=5 pagerank=1.6917e-01 url=1
```
Yours likely won't be identical (due to weird floating point issues), but it should be similar.
In particular, the ranking of the nodes/urls should be the same order.

> **NOTE:**
> The `--verbose` flag causes all of the lines beginning with `DEBUG` to be printed.
> By default, only lines beginning with `INFO` are printed.

**Part 2:**

The `pagerank.py` file has an option `--search_query`, which takes a string as a parameter.
If this argument is used, then the program returns all nodes that match the query string sorted according to their pagerank.
Essentially, this gives us the most important pages related to our query.

Again, you may not get the exact same results as me,
but you should get similar results to the examples I've shown below.
Verify that you do in fact get similar results.

```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='corona'
INFO:root:rank=0 pagerank=1.0038e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=1 pagerank=8.9224e-04 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=2 pagerank=7.0390e-04 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=3 pagerank=6.9153e-04 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=4 pagerank=6.7041e-04 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=5 pagerank=6.6256e-04 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
INFO:root:rank=6 pagerank=6.5046e-04 url=www.lawfareblog.com/congressional-homeland-security-committees-seek-ways-support-state-federal-responses-coronavirus
INFO:root:rank=7 pagerank=6.3620e-04 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=8 pagerank=6.1248e-04 url=www.lawfareblog.com/house-subcommittee-voices-concerns-over-us-management-coronavirus
INFO:root:rank=9 pagerank=6.0187e-04 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='trump'
INFO:root:rank=0 pagerank=5.7826e-03 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=5.2338e-03 url=www.lawfareblog.com/document-trump-revokes-obama-executive-order-counterterrorism-strike-casualty-reporting
INFO:root:rank=2 pagerank=5.1297e-03 url=www.lawfareblog.com/trump-administrations-worrying-new-policy-israeli-settlements
INFO:root:rank=3 pagerank=4.6599e-03 url=www.lawfareblog.com/dc-circuit-overrules-district-courts-due-process-ruling-qasim-v-trump
INFO:root:rank=4 pagerank=4.5934e-03 url=www.lawfareblog.com/donald-trump-and-politically-weaponized-executive-branch
INFO:root:rank=5 pagerank=4.3071e-03 url=www.lawfareblog.com/how-trumps-approach-middle-east-ignores-past-future-and-human-condition
INFO:root:rank=6 pagerank=4.0935e-03 url=www.lawfareblog.com/why-trump-cant-buy-greenland
INFO:root:rank=7 pagerank=3.7591e-03 url=www.lawfareblog.com/oral-argument-summary-qassim-v-trump
INFO:root:rank=8 pagerank=3.4509e-03 url=www.lawfareblog.com/dc-circuit-court-denies-trump-rehearing-mazars-case
INFO:root:rank=9 pagerank=3.4484e-03 url=www.lawfareblog.com/second-circuit-rules-mazars-must-hand-over-trump-tax-returns-new-york-prosecutors

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='iran'
INFO:root:rank=0 pagerank=4.5746e-03 url=www.lawfareblog.com/praise-presidents-iran-tweets
INFO:root:rank=1 pagerank=4.4174e-03 url=www.lawfareblog.com/how-us-iran-tensions-could-disrupt-iraqs-fragile-peace
INFO:root:rank=2 pagerank=2.6928e-03 url=www.lawfareblog.com/cyber-command-operational-update-clarifying-june-2019-iran-operation
INFO:root:rank=3 pagerank=1.9391e-03 url=www.lawfareblog.com/aborted-iran-strike-fine-line-between-necessity-and-revenge
INFO:root:rank=4 pagerank=1.5452e-03 url=www.lawfareblog.com/parsing-state-departments-letter-use-force-against-iran
INFO:root:rank=5 pagerank=1.5357e-03 url=www.lawfareblog.com/iranian-hostage-crisis-and-its-effect-american-politics
INFO:root:rank=6 pagerank=1.5258e-03 url=www.lawfareblog.com/announcing-united-states-and-use-force-against-iran-new-lawfare-e-book
INFO:root:rank=7 pagerank=1.4221e-03 url=www.lawfareblog.com/us-names-iranian-revolutionary-guard-terrorist-organization-and-sanctions-international-criminal
INFO:root:rank=8 pagerank=1.1788e-03 url=www.lawfareblog.com/iran-shoots-down-us-drone-domestic-and-international-legal-implications
INFO:root:rank=9 pagerank=1.1463e-03 url=www.lawfareblog.com/israel-iran-syria-clash-and-law-use-force
```

**Part 3:**

The webgraph of lawfareblog.com (i.e. the `P` matrix) naturally contains a lot of structure.
For example, essentially all pages on the domain have links to the root page <https://lawfareblog.com/> and other "non-article" pages like <https://www.lawfareblog.com/topics> and <https://www.lawfareblog.com/subscribe-lawfare>.
These pages therefore have a large pagerank.
We can get a list of the pages with the largest pagerank by running

```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz
INFO:root:rank=0 pagerank=2.8741e-01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=1 pagerank=2.8741e-01 url=www.lawfareblog.com/masthead
INFO:root:rank=2 pagerank=2.8741e-01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=3 pagerank=2.8741e-01 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=4 pagerank=2.8741e-01 url=www.lawfareblog.com/topics
INFO:root:rank=5 pagerank=2.8741e-01 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=6 pagerank=2.8741e-01 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=7 pagerank=2.8741e-01 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=8 pagerank=2.8741e-01 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=2.8741e-01 url=www.lawfareblog.com/our-comments-policy
```

Most of these pages are not very interesting, however, because they are not articles,
and usually when we are performing a web search, we only want articles.

This raises the question: How can we find the most important articles filtering out the non-article pages?
The answer is to modify the `P` matrix by removing all links to non-article pages.

This raises another question: How do we know if a link is a non-article page?
Unfortunately, this is a hard question to answer with 100% accuracy,
but there are many methods that get us most of the way there.
One easy to implement method is to compute what's called the "in-link ratio" of each node (i.e. the total number of edges with the node as a target divided by the total number of nodes),
and then remove nodes from the search results with too-high of a ratio.
The intuition is that non-article pages often appear in the menu of a webpage, and so have links from almost all of the other webpages;
but article-webpages are unlikely to appear on a menu and so will only have a small number of links from other webpages.
The `--filter_ratio` parameter causes the code to remove all pages that have an in-link ratio larger than the provided value.

Using this option, we can estimate the most important articles on the domain with the following command:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2
INFO:root:rank=0 pagerank=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 pagerank=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull
```
Notice that the urls in this list look much more like articles than the urls in the previous list.

When Google calculates their `P` matrix for the web,
they use a similar (but much more complicated) process to modify the `P` matrix in order to reduce spam results.
The exact formula they use is a jealously guarded secret that they update continuously.

In the case above, notice that we have accidentally removed the blog's most popular article (<www.lawfareblog.com/snowden-revelations>).
The blog editors believed that Snowden's revelations about NSA spying are so important that they directly put a link to the article on the menu.
So every single webpage in the domain links to the Snowden article,
and our "anti-spam" `--filter-ratio` argument removed this article from the list.
In general, it is a challenging open problem to remove spam from pagerank results,
and all current solutions rely on careful human tuning and still have lots of false positives and false negatives.

**Part 4:**

Recall from the reading that the runtime of pagerank depends heavily on the eigengap of the `\bar\bar P` matrix,
and that this eigengap is bounded by the alpha parameter.

Run the following four commands:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose 
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --alpha=0.99999
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
```
You should notice that the last command takes considerably more iterations to compute the pagerank vector.
(My code takes 685 iterations for this call, and about 10 iterations for all the others.)

This raises the question: Why does the second command (with the `--alpha` option but without the `--filter_ratio`) option not take a long time to run?
The answer is that the `P` graph for <https://www.lawfareblog.com> naturally has a large eigengap and so is fast to compute for all alpha values,
but the modified graph does not have a large eigengap and so requires a small alpha for fast convergence.

Changing the value of alpha also gives us very different pagerank rankings.
For example, 
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
INFO:root:rank=0 pagerank=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 pagerank=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
INFO:root:rank=0 pagerank=7.0149e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=7.0149e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.0552e-01 url=www.lawfareblog.com/cost-using-zero-days
INFO:root:rank=3 pagerank=3.1755e-02 url=www.lawfareblog.com/lawfare-podcast-former-congressman-brian-baird-and-daniel-schuman-how-congress-can-continue-function
INFO:root:rank=4 pagerank=2.2040e-02 url=www.lawfareblog.com/events
INFO:root:rank=5 pagerank=1.6027e-02 url=www.lawfareblog.com/water-wars-increased-us-focus-indo-pacific
INFO:root:rank=6 pagerank=1.6026e-02 url=www.lawfareblog.com/water-wars-drill-maybe-drill
INFO:root:rank=7 pagerank=1.6023e-02 url=www.lawfareblog.com/water-wars-disjointed-operations-south-china-sea
INFO:root:rank=8 pagerank=1.6020e-02 url=www.lawfareblog.com/water-wars-song-oil-and-fire
INFO:root:rank=9 pagerank=1.6020e-02 url=www.lawfareblog.com/water-wars-sinking-feeling-philippine-china-relations
```

Which of these rankings is better is entirely subjective,
and the only way to know if you have the "best" alpha for your application is to try several variations and see what is best.
If large alphas are good for your application, you can see that there is a trade-off between quality answers and algorithmic runtime.
We'll be exploring this trade-off more formally in class over the rest of the semester.

## Task 2: the personalization vector

The most interesting applications of pagerank involve the personalization vector.
Implement the `WebGraph.make_personalization_vector` function so that it outputs a personalization vector tuned for the input query.
The pseudocode for the function is:
```
for each index in the personalization vector:
    get the url for the index (see the _index_to_url function)
    check if the url satisfies the input query (see the url_satisfies_query function)
    if so, set the corresponding index to one
normalize the vector
```

**Part 1:**

The command line argument `--personalization_vector_query` will use the function you created above to augment your search with a custom personalization vector.
If you've implemented the function correctly,
you should get results similar to:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona'
INFO:root:rank=0 pagerank=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=1.2209e-01 url=www.lawfareblog.com/brexit-not-immune-coronavirus
INFO:root:rank=4 pagerank=1.2209e-01 url=www.lawfareblog.com/rational-security-my-corona-edition
INFO:root:rank=5 pagerank=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=6 pagerank=9.1920e-02 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=7 pagerank=9.1920e-02 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=8 pagerank=7.7770e-02 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=9 pagerank=7.2888e-02 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
```

Notice that these results are significantly different than when using the `--search_query` option:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --search_query='corona'
INFO:root:rank=0 pagerank=8.1320e-03 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=1 pagerank=7.7908e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=2 pagerank=5.2262e-03 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response
INFO:root:rank=3 pagerank=3.9584e-03 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=4 pagerank=3.8114e-03 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=5 pagerank=3.3973e-03 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=6 pagerank=3.3633e-03 url=www.lawfareblog.com/cyberlaw-podcast-how-israel-fighting-coronavirus
INFO:root:rank=7 pagerank=3.3557e-03 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=8 pagerank=3.2160e-03 url=www.lawfareblog.com/congress-needs-coronavirus-failsafe-its-too-late
INFO:root:rank=9 pagerank=3.1036e-03 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
```

Which results are better?
Again, that depends on what you mean by "better."
With the `--personalization_vector_query` option,
a webpage is important only if other coronavirus webpages also think it's important;
with the `--search_query` option,
a webpage is important if any other webpage thinks it's important.
You'll notice that in the later example, many of the webpages are about Congressional proceedings related to the coronavirus.
From a strictly coronavirus perspective, these are not very important webpages.
But in the broader context of national security, these are very important webpages.

Google engineers spend TONs of time fine-tuning their pagerank personalization vectors to remove spam webpages.
Exactly how they do this is another one of their secrets that they don't publicly talk about.

**Part 2:**

Another use of the `--personalization_vector_query` option is that we can find out what webpages are related to the coronavirus but don't directly mention the coronavirus.
This can be used to map out what types of topics are similar to the coronavirus.

For example, the following query ranks all webpages by their `corona` importance,
but removes webpages mentioning `corona` from the results.
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona' --search_query='-corona'
INFO:root:rank=0 pagerank=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=4 pagerank=7.0277e-02 url=www.lawfareblog.com/fault-lines-foreign-policy-quarantined
INFO:root:rank=5 pagerank=6.9713e-02 url=www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
INFO:root:rank=6 pagerank=6.4944e-02 url=www.lawfareblog.com/limits-world-health-organization
INFO:root:rank=7 pagerank=5.9492e-02 url=www.lawfareblog.com/chinatalk-dispatches-shanghai-beijing-and-hong-kong
INFO:root:rank=8 pagerank=5.1245e-02 url=www.lawfareblog.com/us-moves-dismiss-case-against-company-linked-ira-troll-farm
INFO:root:rank=9 pagerank=5.1245e-02 url=www.lawfareblog.com/livestream-house-armed-services-holds-hearing-national-security-challenges-north-and-south-america
```
You can see that there are many urls about concepts that are obviously related like "covid", "clinical trials", and "quarantine",
but this algorithm also finds articles about Chinese propaganda and Trump's policy decisions.
Both of these articles are highly relevant to coronavirus discussions,
but a simple keyword search for corona or related terms would not find these articles.

<!--
**Part 3:**

Select another topic related to national security.
You should experiment with a national security topic other than the coronavirus.
For example, find out what articles are important to the `iran` topic but do not contain the word `iran`.
Your goal should be to discover what topics that www.lawfareblog.com considers to be related to the national security topic you choose.
-->

## Submission

1. Create a new repo on github (not a fork of this repo).

2. Run the following commands, and paste their output into the code blocks below.
   
   Task 1, part 1:
   ```
   $ python3 ws_pagerank.py --data=./small.csv.gz --verbose
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=0.3775096535682678
DEBUG:root:i=1 residual=0.3134882152080536
DEBUG:root:i=2 residual=0.2756592035293579
DEBUG:root:i=3 residual=0.21698090434074402
DEBUG:root:i=4 residual=0.18984204530715942
DEBUG:root:i=5 residual=0.15531454980373383
DEBUG:root:i=6 residual=0.13266263902187347
DEBUG:root:i=7 residual=0.11062280833721161
DEBUG:root:i=8 residual=0.0935136154294014
DEBUG:root:i=9 residual=0.07847193628549576
DEBUG:root:i=10 residual=0.06611069291830063
DEBUG:root:i=11 residual=0.05558090656995773
DEBUG:root:i=12 residual=0.04677915573120117
DEBUG:root:i=13 residual=0.039349012076854706
DEBUG:root:i=14 residual=0.03310869261622429
DEBUG:root:i=15 residual=0.027853948995471
DEBUG:root:i=16 residual=0.023434894159436226
DEBUG:root:i=17 residual=0.019716130569577217
DEBUG:root:i=18 residual=0.016587838530540466
DEBUG:root:i=19 residual=0.013955758884549141
DEBUG:root:i=20 residual=0.011741633526980877
DEBUG:root:i=21 residual=0.009878222830593586
DEBUG:root:i=22 residual=0.008310933597385883
DEBUG:root:i=23 residual=0.006992380600422621
DEBUG:root:i=24 residual=0.00588278379291296
DEBUG:root:i=25 residual=0.00494928564876318
DEBUG:root:i=26 residual=0.004164101090282202
DEBUG:root:i=27 residual=0.003503275103867054
DEBUG:root:i=28 residual=0.002947412896901369
DEBUG:root:i=29 residual=0.0024798011872917414
DEBUG:root:i=30 residual=0.002086422871798277
DEBUG:root:i=31 residual=0.0017551911296322942
DEBUG:root:i=32 residual=0.0014767624670639634
DEBUG:root:i=33 residual=0.0012423915322870016
DEBUG:root:i=34 residual=0.0010452595306560397
DEBUG:root:i=35 residual=0.0008794325985945761
DEBUG:root:i=36 residual=0.0007398635498248041
DEBUG:root:i=37 residual=0.0006225021206773818
DEBUG:root:i=38 residual=0.0005237152217887342
DEBUG:root:i=39 residual=0.00044049162534065545
DEBUG:root:i=40 residual=0.0003707956930156797
DEBUG:root:i=41 residual=0.0003118595341220498
DEBUG:root:i=42 residual=0.00026235461700707674
DEBUG:root:i=43 residual=0.0002207769575761631
DEBUG:root:i=44 residual=0.00018579662719275802
DEBUG:root:i=45 residual=0.0001563065015943721
DEBUG:root:i=46 residual=0.0001315098925260827
DEBUG:root:i=47 residual=0.00011065506259910762
DEBUG:root:i=48 residual=9.283208783017471e-05
DEBUG:root:i=49 residual=7.835238648112863e-05
DEBUG:root:i=50 residual=6.579793989658356e-05
DEBUG:root:i=51 residual=5.543866427615285e-05
DEBUG:root:i=52 residual=4.6715060307178646e-05
DEBUG:root:i=53 residual=3.909929364454001e-05
DEBUG:root:i=54 residual=3.294386260677129e-05
DEBUG:root:i=55 residual=2.7852673156303354e-05
DEBUG:root:i=56 residual=2.3248065190273337e-05
DEBUG:root:i=57 residual=1.966203490155749e-05
DEBUG:root:i=58 residual=1.6471843991894275e-05
DEBUG:root:i=59 residual=1.4014856787980534e-05
DEBUG:root:i=60 residual=1.1800590982602444e-05
DEBUG:root:i=61 residual=9.742259862832725e-06
DEBUG:root:i=62 residual=8.302875357912853e-06
DEBUG:root:i=63 residual=7.063716111588292e-06
DEBUG:root:i=64 residual=5.845966825290816e-06
DEBUG:root:i=65 residual=4.962869752489496e-06
DEBUG:root:i=66 residual=4.206880475976504e-06
DEBUG:root:i=67 residual=3.499711510812631e-06
DEBUG:root:i=68 residual=2.992129338963423e-06
DEBUG:root:i=69 residual=2.5033950805664062e-06
DEBUG:root:i=70 residual=2.214214191553765e-06
DEBUG:root:i=71 residual=1.955177822310361e-06
DEBUG:root:i=72 residual=1.3902072169003077e-06
DEBUG:root:i=73 residual=1.244581540049694e-06
DEBUG:root:i=74 residual=9.97376446321141e-07
INFO:root:rank=0 ranking=2.1634e+00 url=4
INFO:root:rank=1 ranking=1.6664e+00 url=6
INFO:root:rank=2 ranking=1.2402e+00 url=5
INFO:root:rank=3 ranking=4.5712e-01 url=2
INFO:root:rank=4 ranking=3.5620e-01 url=3
INFO:root:rank=5 ranking=3.2078e-01 url=1
   ```

   Task 1, part 2:
   ```
   $ python3 ws_pagerank.py --data=./lawfareblog.csv.gz --search_query='corona'
INFO:root:rank=0 ranking=4.7776e-16 url=www.lawfareblog.com/ted-cruz-vs-section-230-misrepresenting-communications-decency-act
INFO:root:rank=1 ranking=4.3523e-16 url=www.lawfareblog.com/guess-what-play-ted-cruz-read-filibuster
INFO:root:rank=2 ranking=4.3217e-16 url=www.lawfareblog.com/publicity-stunt-postscript-senators-cruz-and-paul-propose-legislation-targeted-killing-domestic
INFO:root:rank=3 ranking=0.0000e+00 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=4 ranking=0.0000e+00 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=5 ranking=0.0000e+00 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=6 ranking=0.0000e+00 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=7 ranking=0.0000e+00 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=8 ranking=0.0000e+00 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
INFO:root:rank=9 ranking=0.0000e+00 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response

   $ python3 ws_pagerank.py --data=./lawfareblog.csv.gz --search_query='trump'
INFO:root:rank=0 ranking=4.3965e-11 url=www.lawfareblog.com/donald-trump-and-politically-weaponized-executive-branch
INFO:root:rank=1 ranking=5.7033e-12 url=www.lawfareblog.com/did-donald-trump-jr-admit-violating-computer-fraud-and-abuse-act
INFO:root:rank=2 ranking=2.2925e-12 url=www.lawfareblog.com/documents-saifullah-paracha-v-donald-j-trump
INFO:root:rank=3 ranking=1.4129e-12 url=www.lawfareblog.com/cta9-decides-al-nashiri-v-macdonald
INFO:root:rank=4 ranking=1.1447e-12 url=www.lawfareblog.com/donald-trump-danger-our-national-security
INFO:root:rank=5 ranking=1.0365e-12 url=www.lawfareblog.com/strategic-underpinning-and-limits-republican-due-process-defense-donald-trump
INFO:root:rank=6 ranking=1.0311e-12 url=www.lawfareblog.com/burden-donald-trump
INFO:root:rank=7 ranking=9.7108e-13 url=www.lawfareblog.com/nashiri-v-macdonald-dismissed
INFO:root:rank=8 ranking=9.6641e-13 url=www.lawfareblog.com/does-trump-want-lose-eo-battle-court-or-donald-mcgahn-simply-ineffectual-or-worse
INFO:root:rank=9 ranking=9.6401e-13 url=www.lawfareblog.com/donald-trumps-pardon-power-and-state-exception

   $ python3 ws_pagerank.py --data=./lawfareblog.csv.gz --search_query='iran'
INFO:root:rank=0 ranking=1.2998e-09 url=www.lawfareblog.com/update-military-commissions-continued-health-issues-recusal-motion-and-new-cell-al-iraqi
INFO:root:rank=1 ranking=1.1119e-09 url=www.lawfareblog.com/how-us-iran-tensions-could-disrupt-iraqs-fragile-peace
INFO:root:rank=2 ranking=8.9087e-10 url=www.lawfareblog.com/france-makes-play-try-foreign-fighters-iraq
INFO:root:rank=3 ranking=5.5710e-10 url=www.lawfareblog.com/its-not-only-iraq-and-syria
INFO:root:rank=4 ranking=4.6238e-10 url=www.lawfareblog.com/document-sens-kaine-and-young-introduce-bill-revoke-iraq-force-authorizations
INFO:root:rank=5 ranking=3.4343e-10 url=www.lawfareblog.com/assessing-aclu-habeas-petition-behalf-unnamed-us-citizen-held-enemy-combatant-iraq
INFO:root:rank=6 ranking=3.3964e-10 url=www.lawfareblog.com/primer-can-trump-administration-transfer-american-citizen-enemy-combatant-iraqi-custody
INFO:root:rank=7 ranking=1.9822e-10 url=www.lawfareblog.com/iranian-hostage-crisis-and-its-effect-american-politics
INFO:root:rank=8 ranking=1.6873e-10 url=www.lawfareblog.com/us-names-iranian-revolutionary-guard-terrorist-organization-and-sanctions-international-criminal
INFO:root:rank=9 ranking=1.5433e-10 url=www.lawfareblog.com/2002-iraq-aumf-almost-certainly-authorizes-president-use-force-today-iraq-and-might-authorize-use
   ```

   Task 1, part 3:
   ```
   $ python3 ws_pagerank.py --data=./lawfareblog.csv.gz
INFO:root:rank=0 ranking=8.4165e+00 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=1 ranking=8.4165e+00 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=2 ranking=8.4165e+00 url=www.lawfareblog.com/masthead
INFO:root:rank=3 ranking=8.4165e+00 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
INFO:root:rank=4 ranking=8.4165e+00 url=www.lawfareblog.com/subscribe-lawfare
INFO:root:rank=5 ranking=8.4165e+00 url=www.lawfareblog.com/our-comments-policy
INFO:root:rank=6 ranking=8.4165e+00 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=7 ranking=8.4165e+00 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=8 ranking=8.4165e+00 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=9 ranking=8.4165e+00 url=www.lawfareblog.com/topics

   $ python3 ws_pagerank.py --data=./lawfareblog.csv.gz --filter_ratio=0.2
INFO:root:rank=0 ranking=4.6096e+00 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 ranking=2.9870e+00 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 ranking=2.9672e+00 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 ranking=2.0175e+00 url=www.lawfareblog.com/senate-examines-threats-homeland
INFO:root:rank=4 ranking=1.8771e+00 url=www.lawfareblog.com/what-make-first-day-impeachment-hearings
INFO:root:rank=5 ranking=1.8764e+00 url=www.lawfareblog.com/livestream-house-armed-services-committee-hearing-f-35-program
INFO:root:rank=6 ranking=1.8695e+00 url=www.lawfareblog.com/whats-house-resolution-impeachment
INFO:root:rank=7 ranking=1.7657e+00 url=www.lawfareblog.com/congress-us-policy-toward-syria-and-turkey-overview-recent-hearings
INFO:root:rank=8 ranking=1.6809e+00 url=www.lawfareblog.com/summary-david-holmess-deposition-testimony
INFO:root:rank=9 ranking=9.8355e-01 url=www.lawfareblog.com/events
   ```

   Task 1, part 4:
   ```
   $ python3 ws_pagerank.py --data=./lawfareblog.csv.gz --verbose 
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=20.521549224853516
DEBUG:root:i=1 residual=6.110840797424316
DEBUG:root:i=2 residual=1.921691656112671
DEBUG:root:i=3 residual=0.5883616805076599
DEBUG:root:i=4 residual=0.17544545233249664
DEBUG:root:i=5 residual=0.051543526351451874
DEBUG:root:i=6 residual=0.014953291974961758
DEBUG:root:i=7 residual=0.004281134810298681
DEBUG:root:i=8 residual=0.001166230533272028
DEBUG:root:i=9 residual=0.0002583185560069978
DEBUG:root:i=10 residual=3.8402398786274716e-05
DEBUG:root:i=11 residual=6.37924240436405e-05
DEBUG:root:i=12 residual=7.276752148754895e-05
DEBUG:root:i=13 residual=6.609542469959706e-05
DEBUG:root:i=14 residual=5.948194666416384e-05
DEBUG:root:i=15 residual=5.287175372359343e-05
DEBUG:root:i=16 residual=4.296085535315797e-05
DEBUG:root:i=17 residual=3.965138239436783e-05
DEBUG:root:i=18 residual=2.9744605853920802e-05
DEBUG:root:i=19 residual=2.973879236378707e-05
DEBUG:root:i=20 residual=2.3133206923375838e-05
DEBUG:root:i=21 residual=1.9827675714623183e-05
DEBUG:root:i=22 residual=1.6522120859008282e-05
DEBUG:root:i=23 residual=1.3219477295933757e-05
DEBUG:root:i=24 residual=9.915469490806572e-06
DEBUG:root:i=25 residual=6.609816864511231e-06
DEBUG:root:i=26 residual=9.911632332659792e-06
DEBUG:root:i=27 residual=6.609716365346685e-06
DEBUG:root:i=28 residual=3.3058179269573884e-06
DEBUG:root:i=29 residual=9.91109300230164e-06
DEBUG:root:i=30 residual=3.3084629649238195e-06
DEBUG:root:i=31 residual=3.304229494460742e-06
DEBUG:root:i=32 residual=3.30417969962582e-06
DEBUG:root:i=33 residual=1.20138352599497e-07
INFO:root:rank=0 ranking=8.4165e+00 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=1 ranking=8.4165e+00 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=2 ranking=8.4165e+00 url=www.lawfareblog.com/masthead
INFO:root:rank=3 ranking=8.4165e+00 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
INFO:root:rank=4 ranking=8.4165e+00 url=www.lawfareblog.com/subscribe-lawfare
INFO:root:rank=5 ranking=8.4165e+00 url=www.lawfareblog.com/our-comments-policy
INFO:root:rank=6 ranking=8.4165e+00 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=7 ranking=8.4165e+00 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=8 ranking=8.4165e+00 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=9 ranking=8.4165e+00 url=www.lawfareblog.com/topics
    
   $ python3 ws_pagerank.py --data=./lawfareblog.csv.gz --verbose --alpha=0.99999
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=24.140165328979492
DEBUG:root:i=1 residual=8.459001541137695
DEBUG:root:i=2 residual=3.1287007331848145
DEBUG:root:i=3 residual=1.1249890327453613
DEBUG:root:i=4 residual=0.3952835500240326
DEBUG:root:i=5 residual=0.13689903914928436
DEBUG:root:i=6 residual=0.04697415605187416
DEBUG:root:i=7 residual=0.015963079407811165
DEBUG:root:i=8 residual=0.005269130691885948
DEBUG:root:i=9 residual=0.0016010492108762264
DEBUG:root:i=10 residual=0.0003603844961617142
DEBUG:root:i=11 residual=0.00015495483239647
DEBUG:root:i=12 residual=0.00026582475402392447
DEBUG:root:i=13 residual=0.00029755287687294185
DEBUG:root:i=14 residual=0.0003073327534366399
DEBUG:root:i=15 residual=0.00031393207609653473
DEBUG:root:i=16 residual=0.00032053771428763866
DEBUG:root:i=17 residual=0.0003238447243347764
DEBUG:root:i=18 residual=0.0003238478093408048
DEBUG:root:i=19 residual=0.0003205450193490833
DEBUG:root:i=20 residual=0.0003205436805728823
DEBUG:root:i=21 residual=0.00031723958090879023
DEBUG:root:i=22 residual=0.0003172381839249283
DEBUG:root:i=23 residual=0.0003172381839249283
DEBUG:root:i=24 residual=0.00031723800930194557
DEBUG:root:i=25 residual=0.00031723958090879023
DEBUG:root:i=26 residual=0.00031723672873340547
DEBUG:root:i=27 residual=0.0003139353939332068
DEBUG:root:i=28 residual=0.0003139340551570058
DEBUG:root:i=29 residual=0.00031723675783723593
DEBUG:root:i=30 residual=0.00031723809661343694
DEBUG:root:i=31 residual=0.0003172380675096065
DEBUG:root:i=32 residual=0.00032054088660515845
DEBUG:root:i=33 residual=0.0003172408905811608
DEBUG:root:i=34 residual=0.0003172381839249283
DEBUG:root:i=35 residual=0.00031723809661343694
DEBUG:root:i=36 residual=0.0003139354521408677
DEBUG:root:i=37 residual=0.00031723672873340547
DEBUG:root:i=38 residual=0.000317238038405776
DEBUG:root:i=39 residual=0.0003172381257172674
DEBUG:root:i=40 residual=0.00031723943538963795
DEBUG:root:i=41 residual=0.0003172381257172674
DEBUG:root:i=42 residual=0.00031723809661343694
DEBUG:root:i=43 residual=0.00031723681604489684
DEBUG:root:i=44 residual=0.00031723809661343694
DEBUG:root:i=45 residual=0.0003172381257172674
DEBUG:root:i=46 residual=0.0003172381257172674
DEBUG:root:i=47 residual=0.00031723943538963795
DEBUG:root:i=48 residual=0.00032054082839749753
DEBUG:root:i=49 residual=0.00031723958090879023
DEBUG:root:i=50 residual=0.0003172381839249283
DEBUG:root:i=51 residual=0.0003172380675096065
DEBUG:root:i=52 residual=0.0003172395227011293
DEBUG:root:i=53 residual=0.0003172381257172674
DEBUG:root:i=54 residual=0.0003172381839249283
DEBUG:root:i=55 residual=0.0003172381257172674
DEBUG:root:i=56 residual=0.00031723809661343694
DEBUG:root:i=57 residual=0.0003172381257172674
DEBUG:root:i=58 residual=0.0003172394062858075
DEBUG:root:i=59 residual=0.0003172381839249283
DEBUG:root:i=60 residual=0.00031723809661343694
DEBUG:root:i=61 residual=0.0003172381257172674
DEBUG:root:i=62 residual=0.00031723809661343694
DEBUG:root:i=63 residual=0.0003172380675096065
DEBUG:root:i=64 residual=0.00031723672873340547
DEBUG:root:i=65 residual=0.0003172395227011293
DEBUG:root:i=66 residual=0.0003172381257172674
DEBUG:root:i=67 residual=0.0003172381839249283
DEBUG:root:i=68 residual=0.0003172382421325892
DEBUG:root:i=69 residual=0.000317238038405776
DEBUG:root:i=70 residual=0.00031723809661343694
DEBUG:root:i=71 residual=0.00031723949359729886
DEBUG:root:i=72 residual=0.0003139355103485286
DEBUG:root:i=73 residual=0.0003139340551570058
DEBUG:root:i=74 residual=0.00031723667052574456
DEBUG:root:i=75 residual=0.00031723809661343694
DEBUG:root:i=76 residual=0.0003172380675096065
DEBUG:root:i=77 residual=0.0003172381257172674
DEBUG:root:i=78 residual=0.00031723809661343694
DEBUG:root:i=79 residual=0.00032054082839749753
DEBUG:root:i=80 residual=0.0003172408032696694
DEBUG:root:i=81 residual=0.0003172382421325892
DEBUG:root:i=82 residual=0.0003205407701898366
DEBUG:root:i=83 residual=0.00031393690733239055
DEBUG:root:i=84 residual=0.0003172367869410664
DEBUG:root:i=85 residual=0.00031723667052574456
DEBUG:root:i=86 residual=0.00031723943538963795
DEBUG:root:i=87 residual=0.00031723809661343694
DEBUG:root:i=88 residual=0.0003172381257172674
DEBUG:root:i=89 residual=0.00031723815482109785
DEBUG:root:i=90 residual=0.0003172381257172674
DEBUG:root:i=91 residual=0.0003172380675096065
DEBUG:root:i=92 residual=0.0003172381839249283
DEBUG:root:i=93 residual=0.0003172395227011293
DEBUG:root:i=94 residual=0.00031723815482109785
DEBUG:root:i=95 residual=0.0003172381257172674
DEBUG:root:i=96 residual=0.00031393542303703725
DEBUG:root:i=97 residual=0.0003172366414219141
DEBUG:root:i=98 residual=0.000317238038405776
DEBUG:root:i=99 residual=0.0003172381257172674
DEBUG:root:i=100 residual=0.000317238038405776
DEBUG:root:i=101 residual=0.0003172381839249283
DEBUG:root:i=102 residual=0.0003172380675096065
DEBUG:root:i=103 residual=0.0003172381257172674
DEBUG:root:i=104 residual=0.0003205407992936671
DEBUG:root:i=105 residual=0.0003172395518049598
DEBUG:root:i=106 residual=0.000320542196277529
DEBUG:root:i=107 residual=0.00031723958090879023
DEBUG:root:i=108 residual=0.00031723815482109785
DEBUG:root:i=109 residual=0.0003172381839249283
DEBUG:root:i=110 residual=0.0003172381257172674
DEBUG:root:i=111 residual=0.000320542196277529
DEBUG:root:i=112 residual=0.0003172395518049598
DEBUG:root:i=113 residual=0.00031723815482109785
DEBUG:root:i=114 residual=0.00031723800930194557
DEBUG:root:i=115 residual=0.0003172381839249283
DEBUG:root:i=116 residual=0.00031723958090879023
DEBUG:root:i=117 residual=0.0003172381839249283
DEBUG:root:i=118 residual=0.0003172381257172674
DEBUG:root:i=119 residual=0.000317238038405776
DEBUG:root:i=120 residual=0.0003172381257172674
DEBUG:root:i=121 residual=0.0003172380675096065
DEBUG:root:i=122 residual=0.0003172395518049598
DEBUG:root:i=123 residual=0.00031723809661343694
DEBUG:root:i=124 residual=0.00031723815482109785
DEBUG:root:i=125 residual=0.00031723809661343694
DEBUG:root:i=126 residual=0.00031723675783723593
DEBUG:root:i=127 residual=0.00031723809661343694
DEBUG:root:i=128 residual=0.0003172381257172674
DEBUG:root:i=129 residual=0.0003139368782285601
DEBUG:root:i=130 residual=0.00031723667052574456
DEBUG:root:i=131 residual=0.0003139354521408677
DEBUG:root:i=132 residual=0.000317236699629575
DEBUG:root:i=133 residual=0.0003172380675096065
DEBUG:root:i=134 residual=0.0003172381257172674
DEBUG:root:i=135 residual=0.00031723821302875876
DEBUG:root:i=136 residual=0.00031723800930194557
DEBUG:root:i=137 residual=0.0003172381257172674
DEBUG:root:i=138 residual=0.0003172394644934684
DEBUG:root:i=139 residual=0.0003172381839249283
DEBUG:root:i=140 residual=0.00031723821302875876
DEBUG:root:i=141 residual=0.000317238038405776
DEBUG:root:i=142 residual=0.0003172381839249283
DEBUG:root:i=143 residual=0.0003205393732059747
DEBUG:root:i=144 residual=0.0003172409487888217
DEBUG:root:i=145 residual=0.0003172381839249283
DEBUG:root:i=146 residual=0.00031723815482109785
DEBUG:root:i=147 residual=0.0003172381257172674
DEBUG:root:i=148 residual=0.0003172380675096065
DEBUG:root:i=149 residual=0.0003172381839249283
DEBUG:root:i=150 residual=0.00031723943538963795
DEBUG:root:i=151 residual=0.00032054088660515845
DEBUG:root:i=152 residual=0.0003205421380698681
DEBUG:root:i=153 residual=0.00031723949359729886
DEBUG:root:i=154 residual=0.0003172395227011293
DEBUG:root:i=155 residual=0.00031723821302875876
DEBUG:root:i=156 residual=0.0003172381839249283
DEBUG:root:i=157 residual=0.000317238038405776
DEBUG:root:i=158 residual=0.0003172381257172674
DEBUG:root:i=159 residual=0.0003172380675096065
DEBUG:root:i=160 residual=0.0003172395227011293
DEBUG:root:i=161 residual=0.0003139354521408677
DEBUG:root:i=162 residual=0.0003172367869410664
DEBUG:root:i=163 residual=0.00031393548124469817
DEBUG:root:i=164 residual=0.000317236699629575
DEBUG:root:i=165 residual=0.0003205394314136356
DEBUG:root:i=166 residual=0.00031723949359729886
DEBUG:root:i=167 residual=0.0003172381257172674
DEBUG:root:i=168 residual=0.00031723949359729886
DEBUG:root:i=169 residual=0.0003172381257172674
DEBUG:root:i=170 residual=0.0003172380675096065
DEBUG:root:i=171 residual=0.0003172381839249283
DEBUG:root:i=172 residual=0.0003172381257172674
DEBUG:root:i=173 residual=0.00031723809661343694
DEBUG:root:i=174 residual=0.00031723809661343694
DEBUG:root:i=175 residual=0.0003205421380698681
DEBUG:root:i=176 residual=0.0003172395227011293
DEBUG:root:i=177 residual=0.0003172381839249283
DEBUG:root:i=178 residual=0.00031723821302875876
DEBUG:root:i=179 residual=0.00031723809661343694
DEBUG:root:i=180 residual=0.0003172395227011293
DEBUG:root:i=181 residual=0.0003172380675096065
DEBUG:root:i=182 residual=0.0003172381839249283
DEBUG:root:i=183 residual=0.00031723809661343694
DEBUG:root:i=184 residual=0.0003172381257172674
DEBUG:root:i=185 residual=0.0003172381257172674
DEBUG:root:i=186 residual=0.0003172380675096065
DEBUG:root:i=187 residual=0.0003172381257172674
DEBUG:root:i=188 residual=0.0003172381257172674
DEBUG:root:i=189 residual=0.0003172382421325892
DEBUG:root:i=190 residual=0.0003139354521408677
DEBUG:root:i=191 residual=0.00031723667052574456
DEBUG:root:i=192 residual=0.000317238038405776
DEBUG:root:i=193 residual=0.00031393542303703725
DEBUG:root:i=194 residual=0.00031723672873340547
DEBUG:root:i=195 residual=0.000317238038405776
DEBUG:root:i=196 residual=0.00031723958090879023
DEBUG:root:i=197 residual=0.000317238038405776
DEBUG:root:i=198 residual=0.0003172381257172674
DEBUG:root:i=199 residual=0.000317238038405776
DEBUG:root:i=200 residual=0.0003172381839249283
DEBUG:root:i=201 residual=0.00031723809661343694
DEBUG:root:i=202 residual=0.0003172395227011293
DEBUG:root:i=203 residual=0.0003172381257172674
DEBUG:root:i=204 residual=0.0003172381257172674
DEBUG:root:i=205 residual=0.00031723672873340547
DEBUG:root:i=206 residual=0.00031723809661343694
DEBUG:root:i=207 residual=0.0003172381257172674
DEBUG:root:i=208 residual=0.000317238038405776
DEBUG:root:i=209 residual=0.0003172395227011293
DEBUG:root:i=210 residual=0.0003172381257172674
DEBUG:root:i=211 residual=0.00032054082839749753
DEBUG:root:i=212 residual=0.00031723949359729886
DEBUG:root:i=213 residual=0.0003172382421325892
DEBUG:root:i=214 residual=0.0003172394644934684
DEBUG:root:i=215 residual=0.0003172381257172674
DEBUG:root:i=216 residual=0.0003172381257172674
DEBUG:root:i=217 residual=0.00031723809661343694
DEBUG:root:i=218 residual=0.0003172381839249283
DEBUG:root:i=219 residual=0.0003139354521408677
DEBUG:root:i=220 residual=0.0003172366414219141
DEBUG:root:i=221 residual=0.000317238038405776
DEBUG:root:i=222 residual=0.0003172395227011293
DEBUG:root:i=223 residual=0.0003139355394523591
DEBUG:root:i=224 residual=0.0003172353026457131
DEBUG:root:i=225 residual=0.000317238038405776
DEBUG:root:i=226 residual=0.000317238038405776
DEBUG:root:i=227 residual=0.0003172381839249283
DEBUG:root:i=228 residual=0.0003172381257172674
DEBUG:root:i=229 residual=0.0003172381257172674
DEBUG:root:i=230 residual=0.0003172394644934684
DEBUG:root:i=231 residual=0.0003172381257172674
DEBUG:root:i=232 residual=0.000317238038405776
DEBUG:root:i=233 residual=0.0003172381839249283
DEBUG:root:i=234 residual=0.0003172381257172674
DEBUG:root:i=235 residual=0.0003172381257172674
DEBUG:root:i=236 residual=0.0003139353939332068
DEBUG:root:i=237 residual=0.0003205394314136356
DEBUG:root:i=238 residual=0.0003172410069964826
DEBUG:root:i=239 residual=0.0003172381257172674
DEBUG:root:i=240 residual=0.0003172381839249283
DEBUG:root:i=241 residual=0.0003172381257172674
DEBUG:root:i=242 residual=0.00031723800930194557
DEBUG:root:i=243 residual=0.0003172381257172674
DEBUG:root:i=244 residual=0.0003172381839249283
DEBUG:root:i=245 residual=0.0003172381839249283
DEBUG:root:i=246 residual=0.0003172381257172674
DEBUG:root:i=247 residual=0.0003172380675096065
DEBUG:root:i=248 residual=0.00031723809661343694
DEBUG:root:i=249 residual=0.0003172381257172674
DEBUG:root:i=250 residual=0.0003172394644934684
DEBUG:root:i=251 residual=0.0003172381839249283
DEBUG:root:i=252 residual=0.0003172381257172674
DEBUG:root:i=253 residual=0.000317238038405776
DEBUG:root:i=254 residual=0.0003172381257172674
DEBUG:root:i=255 residual=0.0003172381257172674
DEBUG:root:i=256 residual=0.0003172395227011293
DEBUG:root:i=257 residual=0.0003172381839249283
DEBUG:root:i=258 residual=0.0003172380675096065
DEBUG:root:i=259 residual=0.000317238038405776
DEBUG:root:i=260 residual=0.0003172381839249283
DEBUG:root:i=261 residual=0.0003172381257172674
DEBUG:root:i=262 residual=0.0003139355103485286
DEBUG:root:i=263 residual=0.0003172354190610349
DEBUG:root:i=264 residual=0.0003172394062858075
DEBUG:root:i=265 residual=0.0003172381839249283
DEBUG:root:i=266 residual=0.0003172380675096065
DEBUG:root:i=267 residual=0.0003172381839249283
DEBUG:root:i=268 residual=0.00031723809661343694
DEBUG:root:i=269 residual=0.0003172381257172674
DEBUG:root:i=270 residual=0.00031723800930194557
DEBUG:root:i=271 residual=0.000320542196277529
DEBUG:root:i=272 residual=0.0003172395227011293
DEBUG:root:i=273 residual=0.0003172381839249283
DEBUG:root:i=274 residual=0.0003172381839249283
DEBUG:root:i=275 residual=0.0003172380675096065
DEBUG:root:i=276 residual=0.0003172395518049598
DEBUG:root:i=277 residual=0.0003172381257172674
DEBUG:root:i=278 residual=0.0003172381257172674
DEBUG:root:i=279 residual=0.00031393542303703725
DEBUG:root:i=280 residual=0.00031723672873340547
DEBUG:root:i=281 residual=0.00031723809661343694
DEBUG:root:i=282 residual=0.00031723667052574456
DEBUG:root:i=283 residual=0.00031723809661343694
DEBUG:root:i=284 residual=0.0003172395518049598
DEBUG:root:i=285 residual=0.0003172381839249283
DEBUG:root:i=286 residual=0.0003172380675096065
DEBUG:root:i=287 residual=0.0003172381839249283
DEBUG:root:i=288 residual=0.0003172381257172674
DEBUG:root:i=289 residual=0.0003172380675096065
DEBUG:root:i=290 residual=0.00031723809661343694
DEBUG:root:i=291 residual=0.0003172395518049598
DEBUG:root:i=292 residual=0.0003139354521408677
DEBUG:root:i=293 residual=0.000317236699629575
DEBUG:root:i=294 residual=0.0003172381257172674
DEBUG:root:i=295 residual=0.000317238038405776
DEBUG:root:i=296 residual=0.0003172381839249283
DEBUG:root:i=297 residual=0.00031723800930194557
DEBUG:root:i=298 residual=0.00031393542303703725
DEBUG:root:i=299 residual=0.0003139326290693134
DEBUG:root:i=300 residual=0.0003139339678455144
DEBUG:root:i=301 residual=0.0003172366414219141
DEBUG:root:i=302 residual=0.00031063283677212894
DEBUG:root:i=303 residual=0.00031393254175782204
DEBUG:root:i=304 residual=0.0003139339678455144
DEBUG:root:i=305 residual=0.0003139339969493449
DEBUG:root:i=306 residual=0.0003139339678455144
DEBUG:root:i=307 residual=0.000317236699629575
DEBUG:root:i=308 residual=0.00031723667052574456
DEBUG:root:i=309 residual=0.000317238038405776
DEBUG:root:i=310 residual=0.00031723815482109785
DEBUG:root:i=311 residual=0.00031723815482109785
DEBUG:root:i=312 residual=0.0003172395227011293
DEBUG:root:i=313 residual=0.0003172381257172674
DEBUG:root:i=314 residual=0.0003172381257172674
DEBUG:root:i=315 residual=0.00031723809661343694
DEBUG:root:i=316 residual=0.0003172381257172674
DEBUG:root:i=317 residual=0.0003172381257172674
DEBUG:root:i=318 residual=0.0003172381257172674
DEBUG:root:i=319 residual=0.00031723943538963795
DEBUG:root:i=320 residual=0.0003172381257172674
DEBUG:root:i=321 residual=0.0003172381839249283
DEBUG:root:i=322 residual=0.0003172380675096065
DEBUG:root:i=323 residual=0.00031723815482109785
DEBUG:root:i=324 residual=0.0003172381257172674
DEBUG:root:i=325 residual=0.0003172394644934684
DEBUG:root:i=326 residual=0.0003172380675096065
DEBUG:root:i=327 residual=0.0003172381839249283
DEBUG:root:i=328 residual=0.00031393402605317533
DEBUG:root:i=329 residual=0.0003139339678455144
DEBUG:root:i=330 residual=0.0003172366414219141
DEBUG:root:i=331 residual=0.0003205407701898366
DEBUG:root:i=332 residual=0.00032054222538135946
DEBUG:root:i=333 residual=0.00032054365146905184
DEBUG:root:i=334 residual=0.00031723963911645114
DEBUG:root:i=335 residual=0.0003172381839249283
DEBUG:root:i=336 residual=0.0003172380675096065
DEBUG:root:i=337 residual=0.0003172395518049598
DEBUG:root:i=338 residual=0.0003172381257172674
DEBUG:root:i=339 residual=0.0003139354521408677
DEBUG:root:i=340 residual=0.00031393402605317533
DEBUG:root:i=341 residual=0.00031723675783723593
DEBUG:root:i=342 residual=0.0003172379801981151
DEBUG:root:i=343 residual=0.0003172381257172674
DEBUG:root:i=344 residual=0.00031723809661343694
DEBUG:root:i=345 residual=0.0003172381257172674
DEBUG:root:i=346 residual=0.00031723667052574456
DEBUG:root:i=347 residual=0.00031723949359729886
DEBUG:root:i=348 residual=0.0003172381257172674
DEBUG:root:i=349 residual=0.0003172381257172674
DEBUG:root:i=350 residual=0.00031723821302875876
DEBUG:root:i=351 residual=0.0003172381257172674
DEBUG:root:i=352 residual=0.00031723815482109785
DEBUG:root:i=353 residual=0.0003172394062858075
DEBUG:root:i=354 residual=0.0003139355103485286
DEBUG:root:i=355 residual=0.00031723672873340547
DEBUG:root:i=356 residual=0.00031723809661343694
DEBUG:root:i=357 residual=0.0003172381257172674
DEBUG:root:i=358 residual=0.000317238038405776
DEBUG:root:i=359 residual=0.0003172381839249283
DEBUG:root:i=360 residual=0.0003139354521408677
DEBUG:root:i=361 residual=0.00031723675783723593
DEBUG:root:i=362 residual=0.000317238038405776
DEBUG:root:i=363 residual=0.0003172381839249283
DEBUG:root:i=364 residual=0.000317238038405776
DEBUG:root:i=365 residual=0.0003172380675096065
DEBUG:root:i=366 residual=0.0003172381257172674
DEBUG:root:i=367 residual=0.0003172381839249283
DEBUG:root:i=368 residual=0.0003172381257172674
DEBUG:root:i=369 residual=0.00031723943538963795
DEBUG:root:i=370 residual=0.0003172381839249283
DEBUG:root:i=371 residual=0.0003172381257172674
DEBUG:root:i=372 residual=0.0003172381257172674
DEBUG:root:i=373 residual=0.00031393542303703725
DEBUG:root:i=374 residual=0.0003139340551570058
DEBUG:root:i=375 residual=0.0003172366414219141
DEBUG:root:i=376 residual=0.0003172380675096065
DEBUG:root:i=377 residual=0.000317238038405776
DEBUG:root:i=378 residual=0.0003172381839249283
DEBUG:root:i=379 residual=0.0003172380675096065
DEBUG:root:i=380 residual=0.000317238038405776
DEBUG:root:i=381 residual=0.00031723809661343694
DEBUG:root:i=382 residual=0.0003172381839249283
DEBUG:root:i=383 residual=0.0003172381839249283
DEBUG:root:i=384 residual=0.0003172381257172674
DEBUG:root:i=385 residual=0.0003172395518049598
DEBUG:root:i=386 residual=0.000317238038405776
DEBUG:root:i=387 residual=0.0003172381257172674
DEBUG:root:i=388 residual=0.00031723809661343694
DEBUG:root:i=389 residual=0.0003172381257172674
DEBUG:root:i=390 residual=0.0003139354521408677
DEBUG:root:i=391 residual=0.00031393408426083624
DEBUG:root:i=392 residual=0.00030732867890037596
DEBUG:root:i=393 residual=0.00030732579762116075
DEBUG:root:i=394 residual=0.0003139284090138972
DEBUG:root:i=395 residual=0.00031393388053402305
DEBUG:root:i=396 residual=0.00031723661231808364
DEBUG:root:i=397 residual=0.0003172380675096065
DEBUG:root:i=398 residual=0.000317238038405776
DEBUG:root:i=399 residual=0.0003139354521408677
DEBUG:root:i=400 residual=0.0003172367869410664
DEBUG:root:i=401 residual=0.0003172380675096065
DEBUG:root:i=402 residual=0.00031393676181323826
DEBUG:root:i=403 residual=0.0003172367869410664
DEBUG:root:i=404 residual=0.00031723667052574456
DEBUG:root:i=405 residual=0.0003172381257172674
DEBUG:root:i=406 residual=0.0003172380675096065
DEBUG:root:i=407 residual=0.0003172381839249283
DEBUG:root:i=408 residual=0.0003172380675096065
DEBUG:root:i=409 residual=0.0003172381839249283
DEBUG:root:i=410 residual=0.0003172395227011293
DEBUG:root:i=411 residual=0.0003172381257172674
DEBUG:root:i=412 residual=0.0003172381257172674
DEBUG:root:i=413 residual=0.000317238038405776
DEBUG:root:i=414 residual=0.0003139354521408677
DEBUG:root:i=415 residual=0.00031063135247677565
DEBUG:root:i=416 residual=0.0003106299845967442
DEBUG:root:i=417 residual=0.00030732573941349983
DEBUG:root:i=418 residual=0.0003139298059977591
DEBUG:root:i=419 residual=0.00031393382232636213
DEBUG:root:i=420 residual=0.0003172366414219141
DEBUG:root:i=421 residual=0.0003172380675096065
DEBUG:root:i=422 residual=0.00031393542303703725
DEBUG:root:i=423 residual=0.0003139341133646667
DEBUG:root:i=424 residual=0.0003139339096378535
DEBUG:root:i=425 residual=0.0003106298972852528
DEBUG:root:i=426 residual=0.00030732719460502267
DEBUG:root:i=427 residual=0.00031062710331752896
DEBUG:root:i=428 residual=0.0003139324835501611
DEBUG:root:i=429 residual=0.0003172365250065923
DEBUG:root:i=430 residual=0.0003139354521408677
DEBUG:root:i=431 residual=0.0003139339678455144
DEBUG:root:i=432 residual=0.0003139326290693134
DEBUG:root:i=433 residual=0.000317236699629575
DEBUG:root:i=434 residual=0.0003172381257172674
DEBUG:root:i=435 residual=0.00031723800930194557
DEBUG:root:i=436 residual=0.00031723815482109785
DEBUG:root:i=437 residual=0.0003172380675096065
DEBUG:root:i=438 residual=0.0003172381839249283
DEBUG:root:i=439 residual=0.0003172381257172674
DEBUG:root:i=440 residual=0.00031723949359729886
DEBUG:root:i=441 residual=0.0003139354521408677
DEBUG:root:i=442 residual=0.0003073287080042064
DEBUG:root:i=443 residual=0.00030732445884495974
DEBUG:root:i=444 residual=0.0003106270160060376
DEBUG:root:i=445 residual=0.00031393111567012966
DEBUG:root:i=446 residual=0.00031393382232636213
DEBUG:root:i=447 residual=0.00031063129426911473
DEBUG:root:i=448 residual=0.0003139326290693134
DEBUG:root:i=449 residual=0.0003073271655011922
DEBUG:root:i=450 residual=0.0003073244006372988
DEBUG:root:i=451 residual=0.00031393105746246874
DEBUG:root:i=452 residual=0.0003139338514301926
DEBUG:root:i=453 residual=0.00031393393874168396
DEBUG:root:i=454 residual=0.00031393265817314386
DEBUG:root:i=455 residual=0.0003139325126539916
DEBUG:root:i=456 residual=0.00030732862069271505
DEBUG:root:i=457 residual=0.00030732579762116075
DEBUG:root:i=458 residual=0.0003106271615251899
DEBUG:root:i=459 residual=0.0003139325126539916
DEBUG:root:i=460 residual=0.0003139324835501611
DEBUG:root:i=461 residual=0.0003139339096378535
DEBUG:root:i=462 residual=0.0003139339678455144
DEBUG:root:i=463 residual=0.0003205394314136356
DEBUG:root:i=464 residual=0.00031723943538963795
DEBUG:root:i=465 residual=0.00032054088660515845
DEBUG:root:i=466 residual=0.00031724083237349987
DEBUG:root:i=467 residual=0.0003172381839249283
DEBUG:root:i=468 residual=0.0003172381257172674
DEBUG:root:i=469 residual=0.0003172381839249283
DEBUG:root:i=470 residual=0.0003172381839249283
DEBUG:root:i=471 residual=0.0003172380675096065
DEBUG:root:i=472 residual=0.00031723809661343694
DEBUG:root:i=473 residual=0.00031393684912472963
DEBUG:root:i=474 residual=0.000317235360853374
DEBUG:root:i=475 residual=0.000317238038405776
DEBUG:root:i=476 residual=0.0003172381257172674
DEBUG:root:i=477 residual=0.000317238038405776
DEBUG:root:i=478 residual=0.0003139354521408677
DEBUG:root:i=479 residual=0.0003139339969493449
DEBUG:root:i=480 residual=0.00031063135247677565
DEBUG:root:i=481 residual=0.0003139325708616525
DEBUG:root:i=482 residual=0.0003106313815806061
DEBUG:root:i=483 residual=0.0003139310865662992
DEBUG:root:i=484 residual=0.00031723667052574456
DEBUG:root:i=485 residual=0.0003139339678455144
DEBUG:root:i=486 residual=0.000317236699629575
DEBUG:root:i=487 residual=0.0003172380675096065
DEBUG:root:i=488 residual=0.00031723943538963795
DEBUG:root:i=489 residual=0.0003172381257172674
DEBUG:root:i=490 residual=0.0003139354521408677
DEBUG:root:i=491 residual=0.000317236699629575
DEBUG:root:i=492 residual=0.0003172381257172674
DEBUG:root:i=493 residual=0.00031723809661343694
DEBUG:root:i=494 residual=0.0003172380675096065
DEBUG:root:i=495 residual=0.000317238038405776
DEBUG:root:i=496 residual=0.00031723958090879023
DEBUG:root:i=497 residual=0.0003172381257172674
DEBUG:root:i=498 residual=0.0003172381839249283
DEBUG:root:i=499 residual=0.00031723809661343694
DEBUG:root:i=500 residual=0.0003172381257172674
DEBUG:root:i=501 residual=0.0003172381257172674
DEBUG:root:i=502 residual=0.00031723667052574456
DEBUG:root:i=503 residual=0.0003172395227011293
DEBUG:root:i=504 residual=0.00031723809661343694
DEBUG:root:i=505 residual=0.0003139353939332068
DEBUG:root:i=506 residual=0.0003139339969493449
DEBUG:root:i=507 residual=0.0003139341133646667
DEBUG:root:i=508 residual=0.0003106313233729452
DEBUG:root:i=509 residual=0.0003106285585090518
DEBUG:root:i=510 residual=0.00030732713639736176
DEBUG:root:i=511 residual=0.0003139298059977591
DEBUG:root:i=512 residual=0.00031393245444633067
DEBUG:root:i=513 residual=0.0003106312651652843
DEBUG:root:i=514 residual=0.0003139325708616525
DEBUG:root:i=515 residual=0.0003172365832142532
DEBUG:root:i=516 residual=0.00031723809661343694
DEBUG:root:i=517 residual=0.0003172380675096065
DEBUG:root:i=518 residual=0.0003172381839249283
DEBUG:root:i=519 residual=0.0003172381257172674
DEBUG:root:i=520 residual=0.0003106328076682985
DEBUG:root:i=521 residual=0.00031062844209372997
DEBUG:root:i=522 residual=0.0003073272237088531
DEBUG:root:i=523 residual=0.00031393111567012966
DEBUG:root:i=524 residual=0.0003172366414219141
DEBUG:root:i=525 residual=0.0003172380675096065
DEBUG:root:i=526 residual=0.000317238038405776
DEBUG:root:i=527 residual=0.0003172381257172674
DEBUG:root:i=528 residual=0.0003172381257172674
DEBUG:root:i=529 residual=0.00031723667052574456
DEBUG:root:i=530 residual=0.00031723809661343694
DEBUG:root:i=531 residual=0.00031723958090879023
DEBUG:root:i=532 residual=0.0003172381257172674
DEBUG:root:i=533 residual=0.0003172380675096065
DEBUG:root:i=534 residual=0.0003172382421325892
DEBUG:root:i=535 residual=0.0003139354521408677
DEBUG:root:i=536 residual=0.00031393402605317533
DEBUG:root:i=537 residual=0.00031063129426911473
DEBUG:root:i=538 residual=0.0003073271655011922
DEBUG:root:i=539 residual=0.0003106271324213594
DEBUG:root:i=540 residual=0.0003106284129898995
DEBUG:root:i=541 residual=0.0003139325126539916
DEBUG:root:i=542 residual=0.00031393254175782204
DEBUG:root:i=543 residual=0.00031063120695762336
DEBUG:root:i=544 residual=0.00031393254175782204
DEBUG:root:i=545 residual=0.0003139339969493449
DEBUG:root:i=546 residual=0.0003073272237088531
DEBUG:root:i=547 residual=0.0003073244006372988
DEBUG:root:i=548 residual=0.00031062832567840815
DEBUG:root:i=549 residual=0.000310629780869931
DEBUG:root:i=550 residual=0.00031393117387779057
DEBUG:root:i=551 residual=0.0003073271655011922
DEBUG:root:i=552 residual=0.00031062847119756043
DEBUG:root:i=553 residual=0.0003106284129898995
DEBUG:root:i=554 residual=0.000310629780869931
DEBUG:root:i=555 residual=0.00031393254175782204
DEBUG:root:i=556 residual=0.0003106299554929137
DEBUG:root:i=557 residual=0.00031723390566185117
DEBUG:root:i=558 residual=0.0003172379801981151
DEBUG:root:i=559 residual=0.0003172394644934684
DEBUG:root:i=560 residual=0.0003172381839249283
DEBUG:root:i=561 residual=0.000317238038405776
DEBUG:root:i=562 residual=0.0003172381839249283
DEBUG:root:i=563 residual=0.000317238038405776
DEBUG:root:i=564 residual=0.0003172380675096065
DEBUG:root:i=565 residual=0.0003106342046521604
DEBUG:root:i=566 residual=0.0003106286167167127
DEBUG:root:i=567 residual=0.00030732728191651404
DEBUG:root:i=568 residual=0.00031392843811772764
DEBUG:root:i=569 residual=0.00031393388053402305
DEBUG:root:i=570 residual=0.0003139339096378535
DEBUG:root:i=571 residual=0.0003106312651652843
DEBUG:root:i=572 residual=0.00030732728191651404
DEBUG:root:i=573 residual=0.0003073244297411293
DEBUG:root:i=574 residual=0.0003106269286945462
DEBUG:root:i=575 residual=0.0003172338183503598
DEBUG:root:i=576 residual=0.0003139366745017469
DEBUG:root:i=577 residual=0.00031063135247677565
DEBUG:root:i=578 residual=0.00031393126118928194
DEBUG:root:i=579 residual=0.00030732707818970084
DEBUG:root:i=580 residual=0.0003073257685173303
DEBUG:root:i=581 residual=0.00031062710331752896
DEBUG:root:i=582 residual=0.0003139325126539916
DEBUG:root:i=583 residual=0.0003106312651652843
DEBUG:root:i=584 residual=0.0003139297477900982
DEBUG:root:i=585 residual=0.00030732862069271505
DEBUG:root:i=586 residual=0.00030732579762116075
DEBUG:root:i=587 residual=0.00031392977689392865
DEBUG:root:i=588 residual=0.0003172365832142532
DEBUG:root:i=589 residual=0.0003172365832142532
DEBUG:root:i=590 residual=0.00031723809661343694
DEBUG:root:i=591 residual=0.0003172394644934684
DEBUG:root:i=592 residual=0.00031723815482109785
DEBUG:root:i=593 residual=0.0003172380675096065
DEBUG:root:i=594 residual=0.0003139355103485286
DEBUG:root:i=595 residual=0.00031723667052574456
DEBUG:root:i=596 residual=0.00031723809661343694
DEBUG:root:i=597 residual=0.0003172380675096065
DEBUG:root:i=598 residual=0.0003139355103485286
DEBUG:root:i=599 residual=0.00031063129426911473
DEBUG:root:i=600 residual=0.00031723527354188263
DEBUG:root:i=601 residual=0.00031393536482937634
DEBUG:root:i=602 residual=0.0003106299554929137
DEBUG:root:i=603 residual=0.0003139325126539916
DEBUG:root:i=604 residual=0.00030732862069271505
DEBUG:root:i=605 residual=0.0003106271324213594
DEBUG:root:i=606 residual=0.0003172352444380522
DEBUG:root:i=607 residual=0.00031393536482937634
DEBUG:root:i=608 residual=0.00031063135247677565
DEBUG:root:i=609 residual=0.00031393117387779057
DEBUG:root:i=610 residual=0.00030732856248505414
DEBUG:root:i=611 residual=0.00030402172706089914
DEBUG:root:i=612 residual=0.0003106255899183452
DEBUG:root:i=613 residual=0.00030732713639736176
DEBUG:root:i=614 residual=0.00030402030097320676
DEBUG:root:i=615 residual=0.000307324284221977
DEBUG:root:i=616 residual=0.0003139296604786068
DEBUG:root:i=617 residual=0.0003172352153342217
DEBUG:root:i=618 residual=0.0003139353357255459
DEBUG:root:i=619 residual=0.0003139339678455144
DEBUG:root:i=620 residual=0.0003139340551570058
DEBUG:root:i=621 residual=0.0003205393149983138
DEBUG:root:i=622 residual=0.00031723943538963795
DEBUG:root:i=623 residual=0.0003172381257172674
DEBUG:root:i=624 residual=0.00031723958090879023
DEBUG:root:i=625 residual=0.0003139354521408677
DEBUG:root:i=626 residual=0.00031723667052574456
DEBUG:root:i=627 residual=0.0003172380675096065
DEBUG:root:i=628 residual=0.0003172381257172674
DEBUG:root:i=629 residual=0.00031723809661343694
DEBUG:root:i=630 residual=0.0003172382421325892
DEBUG:root:i=631 residual=0.00031723675783723593
DEBUG:root:i=632 residual=0.0003172394062858075
DEBUG:root:i=633 residual=0.00031723815482109785
DEBUG:root:i=634 residual=0.0003172381257172674
DEBUG:root:i=635 residual=0.0003172381257172674
DEBUG:root:i=636 residual=0.00031723809661343694
DEBUG:root:i=637 residual=0.0003172381257172674
DEBUG:root:i=638 residual=0.0003172395227011293
DEBUG:root:i=639 residual=0.0003172381257172674
DEBUG:root:i=640 residual=0.0003172381839249283
DEBUG:root:i=641 residual=0.0003139353939332068
DEBUG:root:i=642 residual=0.0003073287080042064
DEBUG:root:i=643 residual=0.0003073243424296379
DEBUG:root:i=644 residual=0.0003106270742136985
DEBUG:root:i=645 residual=0.0003172351571265608
DEBUG:root:i=646 residual=0.0003139353357255459
DEBUG:root:i=647 residual=0.0003106313815806061
DEBUG:root:i=648 residual=0.00031393111567012966
DEBUG:root:i=649 residual=0.00030732862069271505
DEBUG:root:i=650 residual=0.0003073260304518044
DEBUG:root:i=651 residual=0.0003172317810822278
DEBUG:root:i=652 residual=0.0003139358595944941
DEBUG:root:i=653 residual=0.00031723681604489684
DEBUG:root:i=654 residual=0.0003172380675096065
DEBUG:root:i=655 residual=0.0003172395227011293
DEBUG:root:i=656 residual=0.0003172381257172674
DEBUG:root:i=657 residual=0.00031723815482109785
DEBUG:root:i=658 residual=0.00031723815482109785
DEBUG:root:i=659 residual=0.00031723815482109785
DEBUG:root:i=660 residual=0.0003139354521408677
DEBUG:root:i=661 residual=0.00031723527354188263
DEBUG:root:i=662 residual=0.0003172380675096065
DEBUG:root:i=663 residual=0.00031723949359729886
DEBUG:root:i=664 residual=0.0003172381839249283
DEBUG:root:i=665 residual=0.00031723800930194557
DEBUG:root:i=666 residual=0.00031723815482109785
DEBUG:root:i=667 residual=0.00031723809661343694
DEBUG:root:i=668 residual=0.0003172381257172674
DEBUG:root:i=669 residual=0.00031723809661343694
DEBUG:root:i=670 residual=0.0003172395227011293
DEBUG:root:i=671 residual=0.0003172381257172674
DEBUG:root:i=672 residual=0.000317238038405776
DEBUG:root:i=673 residual=0.0003172381257172674
DEBUG:root:i=674 residual=0.00031723809661343694
DEBUG:root:i=675 residual=0.0003172381257172674
DEBUG:root:i=676 residual=0.0003139353939332068
DEBUG:root:i=677 residual=0.00031063135247677565
DEBUG:root:i=678 residual=0.0003139325708616525
DEBUG:root:i=679 residual=0.00030732728191651404
DEBUG:root:i=680 residual=0.0003073244297411293
DEBUG:root:i=681 residual=0.00031062847119756043
DEBUG:root:i=682 residual=0.00031723512802273035
DEBUG:root:i=683 residual=0.00032054062467068434
DEBUG:root:i=684 residual=0.00031723809661343694
DEBUG:root:i=685 residual=0.00031723809661343694
DEBUG:root:i=686 residual=0.0003172381257172674
DEBUG:root:i=687 residual=0.00031723943538963795
DEBUG:root:i=688 residual=0.0003139355103485286
DEBUG:root:i=689 residual=0.00031723672873340547
DEBUG:root:i=690 residual=0.0003172381257172674
DEBUG:root:i=691 residual=0.0003172381839249283
DEBUG:root:i=692 residual=0.0003172380675096065
DEBUG:root:i=693 residual=0.0003172381257172674
DEBUG:root:i=694 residual=0.00031723809661343694
DEBUG:root:i=695 residual=0.0003172395227011293
DEBUG:root:i=696 residual=0.0003172381257172674
DEBUG:root:i=697 residual=0.0003172381257172674
DEBUG:root:i=698 residual=0.000317238038405776
DEBUG:root:i=699 residual=0.0003172381257172674
DEBUG:root:i=700 residual=0.0003172380675096065
DEBUG:root:i=701 residual=0.0003172395518049598
DEBUG:root:i=702 residual=0.0003172381839249283
DEBUG:root:i=703 residual=0.0003172381257172674
DEBUG:root:i=704 residual=0.000317236699629575
DEBUG:root:i=705 residual=0.00031723800930194557
DEBUG:root:i=706 residual=0.00031723815482109785
DEBUG:root:i=707 residual=0.00031723809661343694
DEBUG:root:i=708 residual=0.0003172395227011293
DEBUG:root:i=709 residual=0.0003172381257172674
DEBUG:root:i=710 residual=0.00031723815482109785
DEBUG:root:i=711 residual=0.0003139354521408677
DEBUG:root:i=712 residual=0.00031063135247677565
DEBUG:root:i=713 residual=0.0003139326290693134
DEBUG:root:i=714 residual=0.0003106312651652843
DEBUG:root:i=715 residual=0.00031393117387779057
DEBUG:root:i=716 residual=0.0003106298390775919
DEBUG:root:i=717 residual=0.0003139325126539916
DEBUG:root:i=718 residual=0.0003139339969493449
DEBUG:root:i=719 residual=0.0003172366414219141
DEBUG:root:i=720 residual=0.000317238038405776
DEBUG:root:i=721 residual=0.00031723809661343694
DEBUG:root:i=722 residual=0.0003172381257172674
DEBUG:root:i=723 residual=0.00031723809661343694
DEBUG:root:i=724 residual=0.0003172395518049598
DEBUG:root:i=725 residual=0.0003172381257172674
DEBUG:root:i=726 residual=0.00031723815482109785
DEBUG:root:i=727 residual=0.0003139354521408677
DEBUG:root:i=728 residual=0.000317235360853374
DEBUG:root:i=729 residual=0.000317238038405776
DEBUG:root:i=730 residual=0.00031723809661343694
DEBUG:root:i=731 residual=0.00031393536482937634
DEBUG:root:i=732 residual=0.00031063135247677565
DEBUG:root:i=733 residual=0.0003139326290693134
DEBUG:root:i=734 residual=0.00030732862069271505
DEBUG:root:i=735 residual=0.0003073244297411293
DEBUG:root:i=736 residual=0.0003106270742136985
DEBUG:root:i=737 residual=0.00031723373103886843
DEBUG:root:i=738 residual=0.00031723943538963795
DEBUG:root:i=739 residual=0.0003172381257172674
DEBUG:root:i=740 residual=0.0003139354521408677
DEBUG:root:i=741 residual=0.00031063135247677565
DEBUG:root:i=742 residual=0.0003139325708616525
DEBUG:root:i=743 residual=0.0003073272237088531
DEBUG:root:i=744 residual=0.0003040216688532382
DEBUG:root:i=745 residual=0.0003106256481260061
DEBUG:root:i=746 residual=0.000310629780869931
DEBUG:root:i=747 residual=0.0003172351571265608
DEBUG:root:i=748 residual=0.0003139339678455144
DEBUG:root:i=749 residual=0.00030732862069271505
DEBUG:root:i=750 residual=0.00030732445884495974
DEBUG:root:i=751 residual=0.0003139310865662992
DEBUG:root:i=752 residual=0.0003172351571265608
DEBUG:root:i=753 residual=0.00031723943538963795
DEBUG:root:i=754 residual=0.0003172381257172674
DEBUG:root:i=755 residual=0.0003172381257172674
DEBUG:root:i=756 residual=0.000317236699629575
DEBUG:root:i=757 residual=0.0003172380675096065
DEBUG:root:i=758 residual=0.00031723809661343694
DEBUG:root:i=759 residual=0.0003172395518049598
DEBUG:root:i=760 residual=0.0003139354521408677
DEBUG:root:i=761 residual=0.00031723675783723593
DEBUG:root:i=762 residual=0.00031723809661343694
DEBUG:root:i=763 residual=0.000317238038405776
DEBUG:root:i=764 residual=0.0003172381839249283
DEBUG:root:i=765 residual=0.00031723809661343694
DEBUG:root:i=766 residual=0.00031393548124469817
DEBUG:root:i=767 residual=0.00031063135247677565
DEBUG:root:i=768 residual=0.00031723533174954355
DEBUG:root:i=769 residual=0.0003172379801981151
DEBUG:root:i=770 residual=0.000317236699629575
DEBUG:root:i=771 residual=0.000317238038405776
DEBUG:root:i=772 residual=0.00031393690733239055
DEBUG:root:i=773 residual=0.00031063135247677565
DEBUG:root:i=774 residual=0.0003139311447739601
DEBUG:root:i=775 residual=0.0003073285915888846
DEBUG:root:i=776 residual=0.00031062852940522134
DEBUG:root:i=777 residual=0.00031393111567012966
DEBUG:root:i=778 residual=0.0003172353026457131
DEBUG:root:i=779 residual=0.00031393527751788497
DEBUG:root:i=780 residual=0.00031723667052574456
DEBUG:root:i=781 residual=0.0003172381257172674
DEBUG:root:i=782 residual=0.00031723949359729886
DEBUG:root:i=783 residual=0.0003172382421325892
DEBUG:root:i=784 residual=0.0003172381257172674
DEBUG:root:i=785 residual=0.00031723800930194557
DEBUG:root:i=786 residual=0.00031723809661343694
DEBUG:root:i=787 residual=0.00031723815482109785
DEBUG:root:i=788 residual=0.0003172395227011293
DEBUG:root:i=789 residual=0.0003172381257172674
DEBUG:root:i=790 residual=0.00031723809661343694
DEBUG:root:i=791 residual=0.0003172381257172674
DEBUG:root:i=792 residual=0.00031723815482109785
DEBUG:root:i=793 residual=0.00031723815482109785
DEBUG:root:i=794 residual=0.0003139339969493449
DEBUG:root:i=795 residual=0.00031723672873340547
DEBUG:root:i=796 residual=0.00031723943538963795
DEBUG:root:i=797 residual=0.0003172381257172674
DEBUG:root:i=798 residual=0.00031723809661343694
DEBUG:root:i=799 residual=0.00031723815482109785
DEBUG:root:i=800 residual=0.0003172381257172674
DEBUG:root:i=801 residual=0.0003172380675096065
DEBUG:root:i=802 residual=0.00031723809661343694
DEBUG:root:i=803 residual=0.0003172395518049598
DEBUG:root:i=804 residual=0.0003172381257172674
DEBUG:root:i=805 residual=0.0003172381257172674
DEBUG:root:i=806 residual=0.0003172381257172674
DEBUG:root:i=807 residual=0.00031723809661343694
DEBUG:root:i=808 residual=0.0003172381257172674
DEBUG:root:i=809 residual=0.0003205407701898366
DEBUG:root:i=810 residual=0.0003172395518049598
DEBUG:root:i=811 residual=0.00031723949359729886
DEBUG:root:i=812 residual=0.0003139341133646667
DEBUG:root:i=813 residual=0.0003205393732059747
DEBUG:root:i=814 residual=0.00031723943538963795
DEBUG:root:i=815 residual=0.0003172395518049598
DEBUG:root:i=816 residual=0.0003139355394523591
DEBUG:root:i=817 residual=0.00032053946051746607
DEBUG:root:i=818 residual=0.0003172394062858075
DEBUG:root:i=819 residual=0.00031723815482109785
DEBUG:root:i=820 residual=0.0003139354521408677
DEBUG:root:i=821 residual=0.00031723672873340547
DEBUG:root:i=822 residual=0.0003172381257172674
DEBUG:root:i=823 residual=0.0003172395227011293
DEBUG:root:i=824 residual=0.00031723809661343694
DEBUG:root:i=825 residual=0.00031723809661343694
DEBUG:root:i=826 residual=0.0003172381839249283
DEBUG:root:i=827 residual=0.00031723809661343694
DEBUG:root:i=828 residual=0.0003172381257172674
DEBUG:root:i=829 residual=0.00031723943538963795
DEBUG:root:i=830 residual=0.00031723815482109785
DEBUG:root:i=831 residual=0.00031723672873340547
DEBUG:root:i=832 residual=0.00031393548124469817
DEBUG:root:i=833 residual=0.00031063135247677565
DEBUG:root:i=834 residual=0.0003139325708616525
DEBUG:root:i=835 residual=0.0003073285915888846
DEBUG:root:i=836 residual=0.00031062710331752896
DEBUG:root:i=837 residual=0.00030732713639736176
DEBUG:root:i=838 residual=0.0003139284090138972
DEBUG:root:i=839 residual=0.0003073285333812237
DEBUG:root:i=840 residual=0.00030732573941349983
DEBUG:root:i=841 residual=0.0003139298059977591
DEBUG:root:i=842 residual=0.0003172365832142532
DEBUG:root:i=843 residual=0.000317238038405776
DEBUG:root:i=844 residual=0.00031723672873340547
DEBUG:root:i=845 residual=0.00031723809661343694
DEBUG:root:i=846 residual=0.0003172394644934684
DEBUG:root:i=847 residual=0.0003139354521408677
DEBUG:root:i=848 residual=0.00031063135247677565
DEBUG:root:i=849 residual=0.0003139312320854515
DEBUG:root:i=850 residual=0.0003106312360614538
DEBUG:root:i=851 residual=0.00031393254175782204
DEBUG:root:i=852 residual=0.0003106312651652843
DEBUG:root:i=853 residual=0.00031393259996548295
DEBUG:root:i=854 residual=0.00031062847119756043
DEBUG:root:i=855 residual=0.0003139325126539916
DEBUG:root:i=856 residual=0.0003139339096378535
DEBUG:root:i=857 residual=0.0003106313233729452
DEBUG:root:i=858 residual=0.00031723527354188263
DEBUG:root:i=859 residual=0.0003172366414219141
DEBUG:root:i=860 residual=0.0003172380675096065
DEBUG:root:i=861 residual=0.000317238038405776
DEBUG:root:i=862 residual=0.00031723809661343694
DEBUG:root:i=863 residual=0.0003106328076682985
DEBUG:root:i=864 residual=0.00031062992638908327
DEBUG:root:i=865 residual=0.0003073272528126836
DEBUG:root:i=866 residual=0.00031062704510986805
DEBUG:root:i=867 residual=0.00030732713639736176
DEBUG:root:i=868 residual=0.00031062570633366704
DEBUG:root:i=869 residual=0.000310629780869931
DEBUG:root:i=870 residual=0.0003073272237088531
DEBUG:root:i=871 residual=0.00031062704510986805
DEBUG:root:i=872 residual=0.0003073271072935313
DEBUG:root:i=873 residual=0.0003139284090138972
DEBUG:root:i=874 residual=0.0003106311778537929
DEBUG:root:i=875 residual=0.00031393259996548295
DEBUG:root:i=876 residual=0.0003106312651652843
DEBUG:root:i=877 residual=0.0003139325126539916
DEBUG:root:i=878 residual=0.00031393259996548295
DEBUG:root:i=879 residual=0.0003172352153342217
DEBUG:root:i=880 residual=0.0003172394644934684
DEBUG:root:i=881 residual=0.00031393548124469817
DEBUG:root:i=882 residual=0.0003106313233729452
DEBUG:root:i=883 residual=0.00031393117387779057
DEBUG:root:i=884 residual=0.00030732862069271505
DEBUG:root:i=885 residual=0.00031062847119756043
DEBUG:root:i=886 residual=0.0003106270742136985
DEBUG:root:i=887 residual=0.0003106297517661005
DEBUG:root:i=888 residual=0.00031062980997376144
DEBUG:root:i=889 residual=0.00031393254175782204
DEBUG:root:i=890 residual=0.00031393254175782204
DEBUG:root:i=891 residual=0.00031062986818142235
DEBUG:root:i=892 residual=0.0003106298390775919
DEBUG:root:i=893 residual=0.00031062986818142235
DEBUG:root:i=894 residual=0.00031062847119756043
DEBUG:root:i=895 residual=0.00030732719460502267
DEBUG:root:i=896 residual=0.0003073257685173303
DEBUG:root:i=897 residual=0.00031062561902217567
DEBUG:root:i=898 residual=0.0003073270490858704
DEBUG:root:i=899 residual=0.0003139298059977591
DEBUG:root:i=900 residual=0.0003073285915888846
DEBUG:root:i=901 residual=0.0003139298059977591
DEBUG:root:i=902 residual=0.0003073285333812237
DEBUG:root:i=903 residual=0.00030732437153346837
DEBUG:root:i=904 residual=0.00031062698690220714
DEBUG:root:i=905 residual=0.00031393105746246874
DEBUG:root:i=906 residual=0.00031393393874168396
DEBUG:root:i=907 residual=0.0003139339096378535
DEBUG:root:i=908 residual=0.00031063129426911473
DEBUG:root:i=909 residual=0.0003139326290693134
DEBUG:root:i=910 residual=0.0003139339678455144
DEBUG:root:i=911 residual=0.0003073272237088531
DEBUG:root:i=912 residual=0.0003106270742136985
DEBUG:root:i=913 residual=0.000310629780869931
DEBUG:root:i=914 residual=0.00031393111567012966
DEBUG:root:i=915 residual=0.0003106312651652843
DEBUG:root:i=916 residual=0.0003106298972852528
DEBUG:root:i=917 residual=0.0003073257685173303
DEBUG:root:i=918 residual=0.00031062704510986805
DEBUG:root:i=919 residual=0.000310629780869931
DEBUG:root:i=920 residual=0.0003139311447739601
DEBUG:root:i=921 residual=0.0003106312651652843
DEBUG:root:i=922 residual=0.0003139311447739601
DEBUG:root:i=923 residual=0.0003106312360614538
DEBUG:root:i=924 residual=0.0003106298972852528
DEBUG:root:i=925 residual=0.0003106284129898995
DEBUG:root:i=926 residual=0.00031062992638908327
DEBUG:root:i=927 residual=0.0003139325126539916
DEBUG:root:i=928 residual=0.00031062986818142235
DEBUG:root:i=929 residual=0.00031062844209372997
DEBUG:root:i=930 residual=0.0003073271655011922
DEBUG:root:i=931 residual=0.00031062852940522134
DEBUG:root:i=932 residual=0.0003106284129898995
DEBUG:root:i=933 residual=0.0003106284129898995
DEBUG:root:i=934 residual=0.0003073271655011922
DEBUG:root:i=935 residual=0.0003106270742136985
DEBUG:root:i=936 residual=0.0003139325126539916
DEBUG:root:i=937 residual=0.00031723661231808364
DEBUG:root:i=938 residual=0.0003172365832142532
DEBUG:root:i=939 residual=0.00031723809661343694
DEBUG:root:i=940 residual=0.0003172380675096065
DEBUG:root:i=941 residual=0.00031723815482109785
DEBUG:root:i=942 residual=0.00031723815482109785
DEBUG:root:i=943 residual=0.0003172395518049598
DEBUG:root:i=944 residual=0.00031723809661343694
DEBUG:root:i=945 residual=0.0003172381257172674
DEBUG:root:i=946 residual=0.0003172381257172674
DEBUG:root:i=947 residual=0.00031723809661343694
DEBUG:root:i=948 residual=0.0003139354521408677
DEBUG:root:i=949 residual=0.0003205394314136356
DEBUG:root:i=950 residual=0.0003172409487888217
DEBUG:root:i=951 residual=0.00031723815482109785
DEBUG:root:i=952 residual=0.0003172381257172674
DEBUG:root:i=953 residual=0.0003172381257172674
DEBUG:root:i=954 residual=0.00031723809661343694
DEBUG:root:i=955 residual=0.0003172380675096065
DEBUG:root:i=956 residual=0.00031723949359729886
DEBUG:root:i=957 residual=0.0003139354521408677
DEBUG:root:i=958 residual=0.0003172367869410664
DEBUG:root:i=959 residual=0.00031723681604489684
DEBUG:root:i=960 residual=0.00031723800930194557
DEBUG:root:i=961 residual=0.0003172381257172674
DEBUG:root:i=962 residual=0.0003172380675096065
DEBUG:root:i=963 residual=0.0003139354521408677
DEBUG:root:i=964 residual=0.0003139339969493449
DEBUG:root:i=965 residual=0.00031393402605317533
DEBUG:root:i=966 residual=0.0003139339096378535
DEBUG:root:i=967 residual=0.00031063135247677565
DEBUG:root:i=968 residual=0.00031393259996548295
DEBUG:root:i=969 residual=0.00031723527354188263
DEBUG:root:i=970 residual=0.000317238038405776
DEBUG:root:i=971 residual=0.000317238038405776
DEBUG:root:i=972 residual=0.00031723809661343694
DEBUG:root:i=973 residual=0.00031393548124469817
DEBUG:root:i=974 residual=0.0003106313233729452
DEBUG:root:i=975 residual=0.0003139326290693134
DEBUG:root:i=976 residual=0.00031723667052574456
DEBUG:root:i=977 residual=0.0003139353357255459
DEBUG:root:i=978 residual=0.00031723667052574456
DEBUG:root:i=979 residual=0.000317236699629575
DEBUG:root:i=980 residual=0.0003172381257172674
DEBUG:root:i=981 residual=0.00031723815482109785
DEBUG:root:i=982 residual=0.00031723943538963795
DEBUG:root:i=983 residual=0.00031723815482109785
DEBUG:root:i=984 residual=0.0003172380675096065
DEBUG:root:i=985 residual=0.000320540857501328
DEBUG:root:i=986 residual=0.00031723949359729886
DEBUG:root:i=987 residual=0.0003139368782285601
DEBUG:root:i=988 residual=0.0003172353026457131
DEBUG:root:i=989 residual=0.00031723943538963795
DEBUG:root:i=990 residual=0.0003172381257172674
DEBUG:root:i=991 residual=0.0003172382421325892
DEBUG:root:i=992 residual=0.0003172381839249283
DEBUG:root:i=993 residual=0.000317238038405776
DEBUG:root:i=994 residual=0.00031723809661343694
DEBUG:root:i=995 residual=0.00031723949359729886
DEBUG:root:i=996 residual=0.0003172381257172674
DEBUG:root:i=997 residual=0.0003172381257172674
DEBUG:root:i=998 residual=0.0003172384458594024
DEBUG:root:i=999 residual=0.0003139341133646667
INFO:root:rank=0 ranking=1.0624e+01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=1 ranking=1.0624e+01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=2 ranking=1.0624e+01 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
INFO:root:rank=3 ranking=1.0624e+01 url=www.lawfareblog.com/subscribe-lawfare
INFO:root:rank=4 ranking=1.0624e+01 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=5 ranking=1.0624e+01 url=www.lawfareblog.com/our-comments-policy
INFO:root:rank=6 ranking=1.0624e+01 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=7 ranking=1.0624e+01 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=8 ranking=1.0624e+01 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=9 ranking=1.0624e+01 url=www.lawfareblog.com/topics
    
   $ python3 ws_pagerank.py --data=./lawfareblog.csv.gz --verbose --filter_ratio=0.2
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=5.117094993591309
DEBUG:root:i=1 residual=2.980546712875366
DEBUG:root:i=2 residual=2.44927978515625
DEBUG:root:i=3 residual=1.6987190246582031
DEBUG:root:i=4 residual=1.1001713275909424
DEBUG:root:i=5 residual=0.7298131585121155
DEBUG:root:i=6 residual=0.5148048400878906
DEBUG:root:i=7 residual=0.379607617855072
DEBUG:root:i=8 residual=0.28510111570358276
DEBUG:root:i=9 residual=0.21571344137191772
DEBUG:root:i=10 residual=0.16483236849308014
DEBUG:root:i=11 residual=0.12841446697711945
DEBUG:root:i=12 residual=0.10304441303014755
DEBUG:root:i=13 residual=0.0855899527668953
DEBUG:root:i=14 residual=0.07334060966968536
DEBUG:root:i=15 residual=0.06422794610261917
DEBUG:root:i=16 residual=0.056897230446338654
DEBUG:root:i=17 residual=0.050569746643304825
DEBUG:root:i=18 residual=0.04486195743083954
DEBUG:root:i=19 residual=0.039611268788576126
DEBUG:root:i=20 residual=0.03476494178175926
DEBUG:root:i=21 residual=0.030315985903143883
DEBUG:root:i=22 residual=0.026269571855664253
DEBUG:root:i=23 residual=0.022627893835306168
DEBUG:root:i=24 residual=0.019384639337658882
DEBUG:root:i=25 residual=0.016523756086826324
DEBUG:root:i=26 residual=0.014022150076925755
DEBUG:root:i=27 residual=0.01185182761400938
DEBUG:root:i=28 residual=0.009982407093048096
DEBUG:root:i=29 residual=0.008381635881960392
DEBUG:root:i=30 residual=0.00701875239610672
DEBUG:root:i=31 residual=0.005863225553184748
DEBUG:root:i=32 residual=0.004888100549578667
DEBUG:root:i=33 residual=0.004067246336489916
DEBUG:root:i=34 residual=0.0033798241056501865
DEBUG:root:i=35 residual=0.0028047487139701843
DEBUG:root:i=36 residual=0.002324760891497135
DEBUG:root:i=37 residual=0.0019253231585025787
DEBUG:root:i=38 residual=0.0015933974646031857
DEBUG:root:i=39 residual=0.0013179727829992771
DEBUG:root:i=40 residual=0.0010894506704062223
DEBUG:root:i=41 residual=0.0009003649465739727
DEBUG:root:i=42 residual=0.0007438872125931084
DEBUG:root:i=43 residual=0.0006143544451333582
DEBUG:root:i=44 residual=0.0005076946108601987
DEBUG:root:i=45 residual=0.0004192432970739901
DEBUG:root:i=46 residual=0.00034654978662729263
DEBUG:root:i=47 residual=0.0002862305846065283
DEBUG:root:i=48 residual=0.00023634154058527201
DEBUG:root:i=49 residual=0.0001955306070158258
DEBUG:root:i=50 residual=0.00016156476340256631
DEBUG:root:i=51 residual=0.00013356450654100627
DEBUG:root:i=52 residual=0.00011056890070904046
DEBUG:root:i=53 residual=9.144437353825197e-05
DEBUG:root:i=54 residual=7.564097904833034e-05
DEBUG:root:i=55 residual=6.271536403801292e-05
DEBUG:root:i=56 residual=5.1968407206004485e-05
DEBUG:root:i=57 residual=4.298948624636978e-05
DEBUG:root:i=58 residual=3.573772119125351e-05
DEBUG:root:i=59 residual=2.9643088055308908e-05
DEBUG:root:i=60 residual=2.4610608306829818e-05
DEBUG:root:i=61 residual=2.0320285329944454e-05
DEBUG:root:i=62 residual=1.6897010937100276e-05
DEBUG:root:i=63 residual=1.3949570529803168e-05
DEBUG:root:i=64 residual=1.1637638635875192e-05
DEBUG:root:i=65 residual=9.701747330836952e-06
DEBUG:root:i=66 residual=8.168256499629933e-06
DEBUG:root:i=67 residual=6.8154154178046156e-06
DEBUG:root:i=68 residual=5.604125817626482e-06
DEBUG:root:i=69 residual=4.643360625777859e-06
DEBUG:root:i=70 residual=3.8143537040014053e-06
DEBUG:root:i=71 residual=3.2927264328463934e-06
DEBUG:root:i=72 residual=2.7293908715364523e-06
DEBUG:root:i=73 residual=2.2958959107199917e-06
DEBUG:root:i=74 residual=1.930193093357957e-06
DEBUG:root:i=75 residual=1.6076633073680568e-06
DEBUG:root:i=76 residual=1.4068068594497163e-06
DEBUG:root:i=77 residual=1.1460226687631803e-06
DEBUG:root:i=78 residual=8.762005450080323e-07
INFO:root:rank=0 ranking=4.6096e+00 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 ranking=2.9870e+00 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 ranking=2.9672e+00 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 ranking=2.0175e+00 url=www.lawfareblog.com/senate-examines-threats-homeland
INFO:root:rank=4 ranking=1.8771e+00 url=www.lawfareblog.com/what-make-first-day-impeachment-hearings
INFO:root:rank=5 ranking=1.8764e+00 url=www.lawfareblog.com/livestream-house-armed-services-committee-hearing-f-35-program
INFO:root:rank=6 ranking=1.8695e+00 url=www.lawfareblog.com/whats-house-resolution-impeachment
INFO:root:rank=7 ranking=1.7657e+00 url=www.lawfareblog.com/congress-us-policy-toward-syria-and-turkey-overview-recent-hearings
INFO:root:rank=8 ranking=1.6809e+00 url=www.lawfareblog.com/summary-david-holmess-deposition-testimony
INFO:root:rank=9 ranking=9.8355e-01 url=www.lawfareblog.com/events
    
   $ python3 ws_pagerank.py --data=./lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
DEBUG:root:i=1 residual=4.125221252441406
DEBUG:root:i=2 residual=3.988121509552002
DEBUG:root:i=3 residual=3.254075765609741
DEBUG:root:i=4 residual=2.479372501373291
DEBUG:root:i=5 residual=1.9349406957626343
DEBUG:root:i=6 residual=1.605736255645752
DEBUG:root:i=7 residual=1.3929795026779175
DEBUG:root:i=8 residual=1.230794072151184
DEBUG:root:i=9 residual=1.0955803394317627
DEBUG:root:i=10 residual=0.9848986268043518
DEBUG:root:i=11 residual=0.9026898741722107
DEBUG:root:i=12 residual=0.8521737456321716
DEBUG:root:i=13 residual=0.8327333331108093
DEBUG:root:i=14 residual=0.839459240436554
DEBUG:root:i=15 residual=0.8648901581764221
DEBUG:root:i=16 residual=0.9013575911521912
DEBUG:root:i=17 residual=0.9424899220466614
DEBUG:root:i=18 residual=0.9836524128913879
DEBUG:root:i=19 residual=1.0217764377593994
DEBUG:root:i=20 residual=1.054997444152832
DEBUG:root:i=21 residual=1.0823228359222412
DEBUG:root:i=22 residual=1.1033577919006348
DEBUG:root:i=23 residual=1.1181120872497559
DEBUG:root:i=24 residual=1.1268588304519653
DEBUG:root:i=25 residual=1.1300336122512817
DEBUG:root:i=26 residual=1.1281651258468628
DEBUG:root:i=27 residual=1.1218254566192627
DEBUG:root:i=28 residual=1.1116007566452026
DEBUG:root:i=29 residual=1.0980510711669922
DEBUG:root:i=30 residual=1.0817190408706665
DEBUG:root:i=31 residual=1.0630974769592285
DEBUG:root:i=32 residual=1.0426437854766846
DEBUG:root:i=33 residual=1.0207664966583252
DEBUG:root:i=34 residual=0.9978246092796326
DEBUG:root:i=35 residual=0.974136471748352
DEBUG:root:i=36 residual=0.9499737024307251
DEBUG:root:i=37 residual=0.9255669713020325
DEBUG:root:i=38 residual=0.9011210203170776
DEBUG:root:i=39 residual=0.8768018484115601
DEBUG:root:i=40 residual=0.8527415990829468
DEBUG:root:i=41 residual=0.8290578722953796
DEBUG:root:i=42 residual=0.8058370351791382
DEBUG:root:i=43 residual=0.7831490635871887
DEBUG:root:i=44 residual=0.7610563635826111
DEBUG:root:i=45 residual=0.7395846247673035
DEBUG:root:i=46 residual=0.7187681794166565
DEBUG:root:i=47 residual=0.6986279487609863
DEBUG:root:i=48 residual=0.6791647672653198
DEBUG:root:i=49 residual=0.6603870987892151
DEBUG:root:i=50 residual=0.6422830820083618
DEBUG:root:i=51 residual=0.6248483061790466
DEBUG:root:i=52 residual=0.6080687642097473
DEBUG:root:i=53 residual=0.5919331908226013
DEBUG:root:i=54 residual=0.5764169692993164
DEBUG:root:i=55 residual=0.5615051984786987
DEBUG:root:i=56 residual=0.5471781492233276
DEBUG:root:i=57 residual=0.5334064960479736
DEBUG:root:i=58 residual=0.5201748609542847
DEBUG:root:i=59 residual=0.5074607729911804
DEBUG:root:i=60 residual=0.4952464997768402
DEBUG:root:i=61 residual=0.483503520488739
DEBUG:root:i=62 residual=0.47221434116363525
DEBUG:root:i=63 residual=0.4613586366176605
DEBUG:root:i=64 residual=0.4509176015853882
DEBUG:root:i=65 residual=0.440864622592926
DEBUG:root:i=66 residual=0.4311904013156891
DEBUG:root:i=67 residual=0.4218740463256836
DEBUG:root:i=68 residual=0.41289955377578735
DEBUG:root:i=69 residual=0.4042448401451111
DEBUG:root:i=70 residual=0.3958982229232788
DEBUG:root:i=71 residual=0.387846976518631
DEBUG:root:i=72 residual=0.38007545471191406
DEBUG:root:i=73 residual=0.3725684583187103
DEBUG:root:i=74 residual=0.3653225600719452
DEBUG:root:i=75 residual=0.358310729265213
DEBUG:root:i=76 residual=0.3515266478061676
DEBUG:root:i=77 residual=0.34497004747390747
DEBUG:root:i=78 residual=0.33862170577049255
DEBUG:root:i=79 residual=0.33247560262680054
DEBUG:root:i=80 residual=0.32650789618492126
DEBUG:root:i=81 residual=0.3207361400127411
DEBUG:root:i=82 residual=0.3151337504386902
DEBUG:root:i=83 residual=0.3096948266029358
DEBUG:root:i=84 residual=0.3044244647026062
DEBUG:root:i=85 residual=0.2992852032184601
DEBUG:root:i=86 residual=0.2943057417869568
DEBUG:root:i=87 residual=0.28946202993392944
DEBUG:root:i=88 residual=0.2847641408443451
DEBUG:root:i=89 residual=0.28016984462738037
DEBUG:root:i=90 residual=0.27571284770965576
DEBUG:root:i=91 residual=0.2713773846626282
DEBUG:root:i=92 residual=0.2671472132205963
DEBUG:root:i=93 residual=0.2630273699760437
DEBUG:root:i=94 residual=0.25901228189468384
DEBUG:root:i=95 residual=0.2550939619541168
DEBUG:root:i=96 residual=0.2512722909450531
DEBUG:root:i=97 residual=0.24754422903060913
DEBUG:root:i=98 residual=0.24390968680381775
DEBUG:root:i=99 residual=0.24035800993442535
DEBUG:root:i=100 residual=0.2368863970041275
DEBUG:root:i=101 residual=0.23349985480308533
DEBUG:root:i=102 residual=0.23018260300159454
DEBUG:root:i=103 residual=0.22695006430149078
DEBUG:root:i=104 residual=0.22378401458263397
DEBUG:root:i=105 residual=0.2206840068101883
DEBUG:root:i=106 residual=0.2176527976989746
DEBUG:root:i=107 residual=0.21467958390712738
DEBUG:root:i=108 residual=0.2117774933576584
DEBUG:root:i=109 residual=0.20893587172031403
DEBUG:root:i=110 residual=0.20614409446716309
DEBUG:root:i=111 residual=0.20341798663139343
DEBUG:root:i=112 residual=0.20074403285980225
DEBUG:root:i=113 residual=0.19811983406543732
DEBUG:root:i=114 residual=0.1955476999282837
DEBUG:root:i=115 residual=0.1930224448442459
DEBUG:root:i=116 residual=0.19055438041687012
DEBUG:root:i=117 residual=0.18813040852546692
DEBUG:root:i=118 residual=0.18575036525726318
DEBUG:root:i=119 residual=0.18340915441513062
DEBUG:root:i=120 residual=0.1811220943927765
DEBUG:root:i=121 residual=0.17886847257614136
DEBUG:root:i=122 residual=0.17665323615074158
DEBUG:root:i=123 residual=0.1744844615459442
DEBUG:root:i=124 residual=0.1723461002111435
DEBUG:root:i=125 residual=0.1702539175748825
DEBUG:root:i=126 residual=0.16819213330745697
DEBUG:root:i=127 residual=0.16616086661815643
DEBUG:root:i=128 residual=0.16417039930820465
DEBUG:root:i=129 residual=0.1622154712677002
DEBUG:root:i=130 residual=0.16028806567192078
DEBUG:root:i=131 residual=0.1584015041589737
DEBUG:root:i=132 residual=0.15652674436569214
DEBUG:root:i=133 residual=0.1547030657529831
DEBUG:root:i=134 residual=0.1528937965631485
DEBUG:root:i=135 residual=0.15111981332302094
DEBUG:root:i=136 residual=0.1493707001209259
DEBUG:root:i=137 residual=0.14765673875808716
DEBUG:root:i=138 residual=0.14594928920269012
DEBUG:root:i=139 residual=0.14429017901420593
DEBUG:root:i=140 residual=0.1426451951265335
DEBUG:root:i=141 residual=0.1410275101661682
DEBUG:root:i=142 residual=0.13943710923194885
DEBUG:root:i=143 residual=0.13786351680755615
DEBUG:root:i=144 residual=0.1363118588924408
DEBUG:root:i=145 residual=0.13478998839855194
DEBUG:root:i=146 residual=0.1332874745130539
DEBUG:root:i=147 residual=0.13180691003799438
DEBUG:root:i=148 residual=0.13034555315971375
DEBUG:root:i=149 residual=0.12891128659248352
DEBUG:root:i=150 residual=0.12748323380947113
DEBUG:root:i=151 residual=0.1260848343372345
DEBUG:root:i=152 residual=0.12471358478069305
DEBUG:root:i=153 residual=0.12335880100727081
DEBUG:root:i=154 residual=0.12200223654508591
DEBUG:root:i=155 residual=0.12068584561347961
DEBUG:root:i=156 residual=0.11937284469604492
DEBUG:root:i=157 residual=0.11808685958385468
DEBUG:root:i=158 residual=0.11682000756263733
DEBUG:root:i=159 residual=0.11555647104978561
DEBUG:root:i=160 residual=0.11432254314422607
DEBUG:root:i=161 residual=0.11309979856014252
DEBUG:root:i=162 residual=0.11189359426498413
DEBUG:root:i=163 residual=0.11070118099451065
DEBUG:root:i=164 residual=0.10952252149581909
DEBUG:root:i=165 residual=0.1083628460764885
DEBUG:root:i=166 residual=0.1072196513414383
DEBUG:root:i=167 residual=0.10609017312526703
DEBUG:root:i=168 residual=0.10497703403234482
DEBUG:root:i=169 residual=0.1038646325469017
DEBUG:root:i=170 residual=0.10277648270130157
DEBUG:root:i=171 residual=0.10169937461614609
DEBUG:root:i=172 residual=0.10064124315977097
DEBUG:root:i=173 residual=0.09959155321121216
DEBUG:root:i=174 residual=0.09855037182569504
DEBUG:root:i=175 residual=0.09752541035413742
DEBUG:root:i=176 residual=0.09651423245668411
DEBUG:root:i=177 residual=0.09551149606704712
DEBUG:root:i=178 residual=0.09452761709690094
DEBUG:root:i=179 residual=0.0935521200299263
DEBUG:root:i=180 residual=0.09258513897657394
DEBUG:root:i=181 residual=0.09163177758455276
DEBUG:root:i=182 residual=0.09069196879863739
DEBUG:root:i=183 residual=0.08976327627897263
DEBUG:root:i=184 residual=0.08884039521217346
DEBUG:root:i=185 residual=0.08793105185031891
DEBUG:root:i=186 residual=0.08703263103961945
DEBUG:root:i=187 residual=0.08613483607769012
DEBUG:root:i=188 residual=0.08526112884283066
DEBUG:root:i=189 residual=0.08439312875270844
DEBUG:root:i=190 residual=0.08353335410356522
DEBUG:root:i=191 residual=0.08268208801746368
DEBUG:root:i=192 residual=0.08185218274593353
DEBUG:root:i=193 residual=0.08101748675107956
DEBUG:root:i=194 residual=0.08020159602165222
DEBUG:root:i=195 residual=0.07939145714044571
DEBUG:root:i=196 residual=0.07858698815107346
DEBUG:root:i=197 residual=0.07779347896575928
DEBUG:root:i=198 residual=0.07701090723276138
DEBUG:root:i=199 residual=0.07623129338026047
DEBUG:root:i=200 residual=0.07546006888151169
DEBUG:root:i=201 residual=0.07469970732927322
DEBUG:root:i=202 residual=0.07395811378955841
DEBUG:root:i=203 residual=0.07321171462535858
DEBUG:root:i=204 residual=0.07248400896787643
DEBUG:root:i=205 residual=0.07175419479608536
DEBUG:root:i=206 residual=0.0710325762629509
DEBUG:root:i=207 residual=0.0703192874789238
DEBUG:root:i=208 residual=0.06961939483880997
DEBUG:root:i=209 residual=0.06891991198062897
DEBUG:root:i=210 residual=0.06824184209108353
DEBUG:root:i=211 residual=0.06755886226892471
DEBUG:root:i=212 residual=0.06688674539327621
DEBUG:root:i=213 residual=0.0662228912115097
DEBUG:root:i=214 residual=0.06556463241577148
DEBUG:root:i=215 residual=0.06489892303943634
DEBUG:root:i=216 residual=0.06425445526838303
DEBUG:root:i=217 residual=0.06361829489469528
DEBUG:root:i=218 residual=0.06298761069774628
DEBUG:root:i=219 residual=0.06236258149147034
DEBUG:root:i=220 residual=0.0617457814514637
DEBUG:root:i=221 residual=0.061142466962337494
DEBUG:root:i=222 residual=0.06053679808974266
DEBUG:root:i=223 residual=0.05993673950433731
DEBUG:root:i=224 residual=0.059339605271816254
DEBUG:root:i=225 residual=0.05875329673290253
DEBUG:root:i=226 residual=0.058172572404146194
DEBUG:root:i=227 residual=0.05759475752711296
DEBUG:root:i=228 residual=0.05703304335474968
DEBUG:root:i=229 residual=0.05646888166666031
DEBUG:root:i=230 residual=0.05591043457388878
DEBUG:root:i=231 residual=0.05536011978983879
DEBUG:root:i=232 residual=0.05481269955635071
DEBUG:root:i=233 residual=0.05427607148885727
DEBUG:root:i=234 residual=0.053737200796604156
DEBUG:root:i=235 residual=0.05321164429187775
DEBUG:root:i=236 residual=0.052681077271699905
DEBUG:root:i=237 residual=0.05216660350561142
DEBUG:root:i=238 residual=0.05164723098278046
DEBUG:root:i=239 residual=0.05115426704287529
DEBUG:root:i=240 residual=0.0506485179066658
DEBUG:root:i=241 residual=0.05015614628791809
DEBUG:root:i=242 residual=0.04966136813163757
DEBUG:root:i=243 residual=0.04917214438319206
DEBUG:root:i=244 residual=0.04868847504258156
DEBUG:root:i=245 residual=0.048210352659225464
DEBUG:root:i=246 residual=0.04773765802383423
DEBUG:root:i=247 residual=0.04727310687303543
DEBUG:root:i=248 residual=0.046811431646347046
DEBUG:root:i=249 residual=0.0463552325963974
DEBUG:root:i=250 residual=0.045899346470832825
DEBUG:root:i=251 residual=0.045448921620845795
DEBUG:root:i=252 residual=0.0450039766728878
DEBUG:root:i=253 residual=0.044567182660102844
DEBUG:root:i=254 residual=0.04413324221968651
DEBUG:root:i=255 residual=0.043707385659217834
DEBUG:root:i=256 residual=0.04327385872602463
DEBUG:root:i=257 residual=0.042845919728279114
DEBUG:root:i=258 residual=0.04243384674191475
DEBUG:root:i=259 residual=0.04201943054795265
DEBUG:root:i=260 residual=0.04160787910223007
DEBUG:root:i=261 residual=0.04119923338294029
DEBUG:root:i=262 residual=0.04079334810376167
DEBUG:root:i=263 residual=0.04040338099002838
DEBUG:root:i=264 residual=0.04000585526227951
DEBUG:root:i=265 residual=0.03961895778775215
DEBUG:root:i=266 residual=0.03923233598470688
DEBUG:root:i=267 residual=0.03885116055607796
DEBUG:root:i=268 residual=0.038475435227155685
DEBUG:root:i=269 residual=0.0380973182618618
DEBUG:root:i=270 residual=0.037727221846580505
DEBUG:root:i=271 residual=0.03735475614666939
DEBUG:root:i=272 residual=0.03699295222759247
DEBUG:root:i=273 residual=0.03663397952914238
DEBUG:root:i=274 residual=0.036275215446949005
DEBUG:root:i=275 residual=0.03592446818947792
DEBUG:root:i=276 residual=0.035571351647377014
DEBUG:root:i=277 residual=0.035231564193964005
DEBUG:root:i=278 residual=0.034884072840213776
DEBUG:root:i=279 residual=0.03455246239900589
DEBUG:root:i=280 residual=0.03421582281589508
DEBUG:root:i=281 residual=0.03388207405805588
DEBUG:root:i=282 residual=0.033551063388586044
DEBUG:root:i=283 residual=0.0332254134118557
DEBUG:root:i=284 residual=0.03290000185370445
DEBUG:root:i=285 residual=0.03258528187870979
DEBUG:root:i=286 residual=0.03225759416818619
DEBUG:root:i=287 residual=0.031956300139427185
DEBUG:root:i=288 residual=0.03163953125476837
DEBUG:root:i=289 residual=0.03132811561226845
DEBUG:root:i=290 residual=0.031029939651489258
DEBUG:root:i=291 residual=0.030726730823516846
DEBUG:root:i=292 residual=0.030434219166636467
DEBUG:root:i=293 residual=0.03013908863067627
DEBUG:root:i=294 residual=0.029846884310245514
DEBUG:root:i=295 residual=0.02954963780939579
DEBUG:root:i=296 residual=0.029265573248267174
DEBUG:root:i=297 residual=0.028986915946006775
DEBUG:root:i=298 residual=0.028705837205052376
DEBUG:root:i=299 residual=0.028422309085726738
DEBUG:root:i=300 residual=0.02815202809870243
DEBUG:root:i=301 residual=0.02787664160132408
DEBUG:root:i=302 residual=0.02760663628578186
DEBUG:root:i=303 residual=0.02733682096004486
DEBUG:root:i=304 residual=0.02706979028880596
DEBUG:root:i=305 residual=0.026813244447112083
DEBUG:root:i=306 residual=0.02654648758471012
DEBUG:root:i=307 residual=0.02628774754703045
DEBUG:root:i=308 residual=0.02603701502084732
DEBUG:root:i=309 residual=0.025788990780711174
DEBUG:root:i=310 residual=0.025530675426125526
DEBUG:root:i=311 residual=0.025285599753260612
DEBUG:root:i=312 residual=0.02504853717982769
DEBUG:root:i=313 residual=0.024798491969704628
DEBUG:root:i=314 residual=0.02456430159509182
DEBUG:root:i=315 residual=0.02431974746286869
DEBUG:root:i=316 residual=0.02409111149609089
DEBUG:root:i=317 residual=0.023849422112107277
DEBUG:root:i=318 residual=0.02363141067326069
DEBUG:root:i=319 residual=0.023400496691465378
DEBUG:root:i=320 residual=0.02316451631486416
DEBUG:root:i=321 residual=0.022944375872612
DEBUG:root:i=322 residual=0.02272430807352066
DEBUG:root:i=323 residual=0.02250180020928383
DEBUG:root:i=324 residual=0.022284600883722305
DEBUG:root:i=325 residual=0.02206755056977272
DEBUG:root:i=326 residual=0.02185589261353016
DEBUG:root:i=327 residual=0.021641695871949196
DEBUG:root:i=328 residual=0.02142764814198017
DEBUG:root:i=329 residual=0.02122160606086254
DEBUG:root:i=330 residual=0.021020861342549324
DEBUG:root:i=331 residual=0.020820198580622673
DEBUG:root:i=332 residual=0.020611921325325966
DEBUG:root:i=333 residual=0.020414026454091072
DEBUG:root:i=334 residual=0.02022426575422287
DEBUG:root:i=335 residual=0.020021511241793633
DEBUG:root:i=336 residual=0.01983189769089222
DEBUG:root:i=337 residual=0.019639823585748672
DEBUG:root:i=338 residual=0.01944526843726635
DEBUG:root:i=339 residual=0.01925341971218586
DEBUG:root:i=340 residual=0.019077377393841743
DEBUG:root:i=341 residual=0.01889095827937126
DEBUG:root:i=342 residual=0.018707294017076492
DEBUG:root:i=343 residual=0.01853155344724655
DEBUG:root:i=344 residual=0.018350712954998016
DEBUG:root:i=345 residual=0.018172558397054672
DEBUG:root:i=346 residual=0.017986712977290154
DEBUG:root:i=347 residual=0.01782182790338993
DEBUG:root:i=348 residual=0.01764664053916931
DEBUG:root:i=349 residual=0.017479347065091133
DEBUG:root:i=350 residual=0.01730436086654663
DEBUG:root:i=351 residual=0.017139902338385582
DEBUG:root:i=352 residual=0.01698332466185093
DEBUG:root:i=353 residual=0.016806062310934067
DEBUG:root:i=354 residual=0.016657529398798943
DEBUG:root:i=355 residual=0.01649085059762001
DEBUG:root:i=356 residual=0.016329528763890266
DEBUG:root:i=357 residual=0.016165653243660927
DEBUG:root:i=358 residual=0.016012364998459816
DEBUG:root:i=359 residual=0.015856517478823662
DEBUG:root:i=360 residual=0.015703342854976654
DEBUG:root:i=361 residual=0.015552920289337635
DEBUG:root:i=362 residual=0.015402551740407944
DEBUG:root:i=363 residual=0.0152549147605896
DEBUG:root:i=364 residual=0.015115194022655487
DEBUG:root:i=365 residual=0.014959912747144699
DEBUG:root:i=366 residual=0.014815134927630424
DEBUG:root:i=367 residual=0.014667821116745472
DEBUG:root:i=368 residual=0.014536285772919655
DEBUG:root:i=369 residual=0.014394373632967472
DEBUG:root:i=370 residual=0.014249970205128193
DEBUG:root:i=371 residual=0.01411084271967411
DEBUG:root:i=372 residual=0.013979536481201649
DEBUG:root:i=373 residual=0.013848431408405304
DEBUG:root:i=374 residual=0.013709545135498047
DEBUG:root:i=375 residual=0.013581217266619205
DEBUG:root:i=376 residual=0.01344772893935442
DEBUG:root:i=377 residual=0.01331689115613699
DEBUG:root:i=378 residual=0.013188793323934078
DEBUG:root:i=379 residual=0.013065967708826065
DEBUG:root:i=380 residual=0.012935364618897438
DEBUG:root:i=381 residual=0.012817935086786747
DEBUG:root:i=382 residual=0.012684891000390053
DEBUG:root:i=383 residual=0.012565015815198421
DEBUG:root:i=384 residual=0.012442593462765217
DEBUG:root:i=385 residual=0.012320202775299549
DEBUG:root:i=386 residual=0.012200543656945229
DEBUG:root:i=387 residual=0.012086084112524986
DEBUG:root:i=388 residual=0.011974431574344635
DEBUG:root:i=389 residual=0.011852304451167583
DEBUG:root:i=390 residual=0.011738120578229427
DEBUG:root:i=391 residual=0.011631833389401436
DEBUG:root:i=392 residual=0.011512589640915394
DEBUG:root:i=393 residual=0.011406457051634789
DEBUG:root:i=394 residual=0.011295133270323277
DEBUG:root:i=395 residual=0.011183938011527061
DEBUG:root:i=396 residual=0.011075318790972233
DEBUG:root:i=397 residual=0.010969492606818676
DEBUG:root:i=398 residual=0.010863648727536201
DEBUG:root:i=399 residual=0.010744839906692505
DEBUG:root:i=400 residual=0.010652142576873302
DEBUG:root:i=401 residual=0.010549104772508144
DEBUG:root:i=402 residual=0.010446161031723022
DEBUG:root:i=403 residual=0.010348491370677948
DEBUG:root:i=404 residual=0.010245648212730885
DEBUG:root:i=405 residual=0.010145445354282856
DEBUG:root:i=406 residual=0.010047960095107555
DEBUG:root:i=407 residual=0.009950532577931881
DEBUG:root:i=408 residual=0.009853193536400795
DEBUG:root:i=409 residual=0.009758436121046543
DEBUG:root:i=410 residual=0.009663800708949566
DEBUG:root:i=411 residual=0.009579637087881565
DEBUG:root:i=412 residual=0.009479916654527187
DEBUG:root:i=413 residual=0.009388060308992863
DEBUG:root:i=414 residual=0.009298861026763916
DEBUG:root:i=415 residual=0.009201846085488796
DEBUG:root:i=416 residual=0.009120569564402103
DEBUG:root:i=417 residual=0.009028921835124493
DEBUG:root:i=418 residual=0.008947767317295074
DEBUG:root:i=419 residual=0.008861429058015347
DEBUG:root:i=420 residual=0.008772576227784157
DEBUG:root:i=421 residual=0.008686292916536331
DEBUG:root:i=422 residual=0.00860273465514183
DEBUG:root:i=423 residual=0.008521885611116886
DEBUG:root:i=424 residual=0.008435782976448536
DEBUG:root:i=425 residual=0.008357618004083633
DEBUG:root:i=426 residual=0.008274253457784653
DEBUG:root:i=427 residual=0.008196153677999973
DEBUG:root:i=428 residual=0.008120686747133732
DEBUG:root:i=429 residual=0.0080427136272192
DEBUG:root:i=430 residual=0.007964780554175377
DEBUG:root:i=431 residual=0.007886871695518494
DEBUG:root:i=432 residual=0.0078089796006679535
DEBUG:root:i=433 residual=0.007736438885331154
DEBUG:root:i=434 residual=0.0076560648158192635
DEBUG:root:i=435 residual=0.007583501283079386
DEBUG:root:i=436 residual=0.007516263518482447
DEBUG:root:i=437 residual=0.007441219873726368
DEBUG:root:i=438 residual=0.007368897087872028
DEBUG:root:i=439 residual=0.007296499330550432
DEBUG:root:i=440 residual=0.0072268289513885975
DEBUG:root:i=441 residual=0.007157261949032545
DEBUG:root:i=442 residual=0.007085041143000126
DEBUG:root:i=443 residual=0.00702597526833415
DEBUG:root:i=444 residual=0.006948580965399742
DEBUG:root:i=445 residual=0.006886952556669712
DEBUG:root:i=446 residual=0.0068149021826684475
DEBUG:root:i=447 residual=0.006750789936631918
DEBUG:root:i=448 residual=0.006691859103739262
DEBUG:root:i=449 residual=0.006622567307204008
DEBUG:root:i=450 residual=0.00656114611774683
DEBUG:root:i=451 residual=0.006491909734904766
DEBUG:root:i=452 residual=0.0064279260113835335
DEBUG:root:i=453 residual=0.006371865514665842
DEBUG:root:i=454 residual=0.006310609169304371
DEBUG:root:i=455 residual=0.006259767338633537
DEBUG:root:i=456 residual=0.006182883400470018
DEBUG:root:i=457 residual=0.0061269886791706085
DEBUG:root:i=458 residual=0.006063232198357582
DEBUG:root:i=459 residual=0.006015140563249588
DEBUG:root:i=460 residual=0.0059514944441616535
DEBUG:root:i=461 residual=0.005893083289265633
DEBUG:root:i=462 residual=0.005842533428221941
DEBUG:root:i=463 residual=0.005784173961728811
DEBUG:root:i=464 residual=0.00572590297088027
DEBUG:root:i=465 residual=0.005672805476933718
DEBUG:root:i=466 residual=0.005617197602987289
DEBUG:root:i=467 residual=0.0055668288841843605
DEBUG:root:i=468 residual=0.005511261522769928
DEBUG:root:i=469 residual=0.0054504903964698315
DEBUG:root:i=470 residual=0.005402804352343082
DEBUG:root:i=471 residual=0.0053499070927500725
DEBUG:root:i=472 residual=0.00529449013993144
DEBUG:root:i=473 residual=0.005241704173386097
DEBUG:root:i=474 residual=0.005188930779695511
DEBUG:root:i=475 residual=0.005146641284227371
DEBUG:root:i=476 residual=0.005091319791972637
DEBUG:root:i=477 residual=0.00504644401371479
DEBUG:root:i=478 residual=0.00499121006578207
DEBUG:root:i=479 residual=0.004951608367264271
DEBUG:root:i=480 residual=0.004901650361716747
DEBUG:root:i=481 residual=0.004856932908296585
DEBUG:root:i=482 residual=0.0048096212558448315
DEBUG:root:i=483 residual=0.004767547361552715
DEBUG:root:i=484 residual=0.004717657342553139
DEBUG:root:i=485 residual=0.004667853936553001
DEBUG:root:i=486 residual=0.004628450609743595
DEBUG:root:i=487 residual=0.004578676540404558
DEBUG:root:i=488 residual=0.004539392422884703
DEBUG:root:i=489 residual=0.004494883585721254
DEBUG:root:i=490 residual=0.004439942073076963
DEBUG:root:i=491 residual=0.004405918996781111
DEBUG:root:i=492 residual=0.004361467901617289
DEBUG:root:i=493 residual=0.004324937239289284
DEBUG:root:i=494 residual=0.0042778439819812775
DEBUG:root:i=495 residual=0.0042413403280079365
DEBUG:root:i=496 residual=0.004191796761006117
DEBUG:root:i=497 residual=0.0041526551358401775
DEBUG:root:i=498 residual=0.004113667644560337
DEBUG:root:i=499 residual=0.0040746224112808704
DEBUG:root:i=500 residual=0.004040820524096489
DEBUG:root:i=501 residual=0.0040018088184297085
DEBUG:root:i=502 residual=0.003962875343859196
DEBUG:root:i=503 residual=0.003921288065612316
DEBUG:root:i=504 residual=0.0038876072503626347
DEBUG:root:i=505 residual=0.0038408925756812096
DEBUG:root:i=506 residual=0.0038098583463579416
DEBUG:root:i=507 residual=0.003771012183278799
DEBUG:root:i=508 residual=0.003734779544174671
DEBUG:root:i=509 residual=0.0036985648330301046
DEBUG:root:i=510 residual=0.003659763839095831
DEBUG:root:i=511 residual=0.0036340919323265553
DEBUG:root:i=512 residual=0.003590100910514593
DEBUG:root:i=513 residual=0.0035592277999967337
DEBUG:root:i=514 residual=0.00352577050216496
DEBUG:root:i=515 residual=0.0034923157654702663
DEBUG:root:i=516 residual=0.003458893857896328
DEBUG:root:i=517 residual=0.0034228635486215353
DEBUG:root:i=518 residual=0.0033894525840878487
DEBUG:root:i=519 residual=0.0033612896222621202
DEBUG:root:i=520 residual=0.0033279075287282467
DEBUG:root:i=521 residual=0.003294609487056732
DEBUG:root:i=522 residual=0.003256041556596756
DEBUG:root:i=523 residual=0.0032279868610203266
DEBUG:root:i=524 residual=0.00320256594568491
DEBUG:root:i=525 residual=0.003166642738506198
DEBUG:root:i=526 residual=0.0031386325135827065
DEBUG:root:i=527 residual=0.003108020406216383
DEBUG:root:i=528 residual=0.003085255390033126
DEBUG:root:i=529 residual=0.003041608491912484
DEBUG:root:i=530 residual=0.0030188686214387417
DEBUG:root:i=531 residual=0.0029857733752578497
DEBUG:root:i=532 residual=0.002963050501421094
DEBUG:root:i=533 residual=0.002935195341706276
DEBUG:root:i=534 residual=0.00290206097997725
DEBUG:root:i=535 residual=0.0028742270078510046
DEBUG:root:i=536 residual=0.0028516233433037996
DEBUG:root:i=537 residual=0.0028132975567132235
DEBUG:root:i=538 residual=0.002795949811115861
DEBUG:root:i=539 residual=0.002768151229247451
DEBUG:root:i=540 residual=0.002735147485509515
DEBUG:root:i=541 residual=0.0027152071706950665
DEBUG:root:i=542 residual=0.002682231366634369
DEBUG:root:i=543 residual=0.0026571606285870075
DEBUG:root:i=544 residual=0.0026268113870173693
DEBUG:root:i=545 residual=0.002612143522128463
DEBUG:root:i=546 residual=0.002589720068499446
DEBUG:root:i=547 residual=0.0025620157830417156
DEBUG:root:i=548 residual=0.0025343794841319323
DEBUG:root:i=549 residual=0.0025146102998405695
DEBUG:root:i=550 residual=0.0024896026588976383
DEBUG:root:i=551 residual=0.0024593297857791185
DEBUG:root:i=552 residual=0.0024369608145207167
DEBUG:root:i=553 residual=0.0024172221310436726
DEBUG:root:i=554 residual=0.002392261754721403
DEBUG:root:i=555 residual=0.0023725335486233234
DEBUG:root:i=556 residual=0.0023450495209544897
DEBUG:root:i=557 residual=0.00231751031242311
DEBUG:root:i=558 residual=0.0022951974533498287
DEBUG:root:i=559 residual=0.0022833526600152254
DEBUG:root:i=560 residual=0.002263737376779318
DEBUG:root:i=561 residual=0.002236241241917014
DEBUG:root:i=562 residual=0.0022114135790616274
DEBUG:root:i=563 residual=0.0021892236545681953
DEBUG:root:i=564 residual=0.002174805151298642
DEBUG:root:i=565 residual=0.0021473944652825594
DEBUG:root:i=566 residual=0.0021278338972479105
DEBUG:root:i=567 residual=0.002105667954310775
DEBUG:root:i=568 residual=0.0020939644891768694
DEBUG:root:i=569 residual=0.0020665901247411966
DEBUG:root:i=570 residual=0.0020470714662224054
DEBUG:root:i=571 residual=0.002027541631832719
DEBUG:root:i=572 residual=0.0020158819388598204
DEBUG:root:i=573 residual=0.0019937583710998297
DEBUG:root:i=574 residual=0.0019690487533807755
DEBUG:root:i=575 residual=0.0019548414275050163
DEBUG:root:i=576 residual=0.0019353616517037153
DEBUG:root:i=577 residual=0.0019132868619635701
DEBUG:root:i=578 residual=0.0018991052638739347
DEBUG:root:i=579 residual=0.001879656920209527
DEBUG:root:i=580 residual=0.001860259915702045
DEBUG:root:i=581 residual=0.0018435072852298617
DEBUG:root:i=582 residual=0.001824068371206522
DEBUG:root:i=583 residual=0.00180210848338902
DEBUG:root:i=584 residual=0.001787975081242621
DEBUG:root:i=585 residual=0.0017712418921291828
DEBUG:root:i=586 residual=0.0017492937622591853
DEBUG:root:i=587 residual=0.001740418840199709
DEBUG:root:i=588 residual=0.0017184782773256302
DEBUG:root:i=589 residual=0.001706994604319334
DEBUG:root:i=590 residual=0.0016850605607032776
DEBUG:root:i=591 residual=0.0016709869960322976
DEBUG:root:i=592 residual=0.0016569197177886963
DEBUG:root:i=593 residual=0.0016323882155120373
DEBUG:root:i=594 residual=0.0016262440476566553
DEBUG:root:i=595 residual=0.0016095744213089347
DEBUG:root:i=596 residual=0.001592916320078075
DEBUG:root:i=597 residual=0.0015789377503097057
DEBUG:root:i=598 residual=0.0015596747398376465
DEBUG:root:i=599 residual=0.0015404938021674752
DEBUG:root:i=600 residual=0.001531692803837359
DEBUG:root:i=601 residual=0.001517749042250216
DEBUG:root:i=602 residual=0.001501124701462686
DEBUG:root:i=603 residual=0.001489801798015833
DEBUG:root:i=604 residual=0.001470640068873763
DEBUG:root:i=605 residual=0.0014566422905772924
DEBUG:root:i=606 residual=0.0014453425537794828
DEBUG:root:i=607 residual=0.0014314190484583378
DEBUG:root:i=608 residual=0.0014201239682734013
DEBUG:root:i=609 residual=0.001408829353749752
DEBUG:root:i=610 residual=0.0013870950788259506
DEBUG:root:i=611 residual=0.0013810412492603064
DEBUG:root:i=612 residual=0.0013593221083283424
DEBUG:root:i=613 residual=0.0013532775919884443
DEBUG:root:i=614 residual=0.001342006609775126
DEBUG:root:i=615 residual=0.0013307525077834725
DEBUG:root:i=616 residual=0.001309049315750599
DEBUG:root:i=617 residual=0.0012978638987988234
DEBUG:root:i=618 residual=0.0012866249307990074
DEBUG:root:i=619 residual=0.0012701608939096332
DEBUG:root:i=620 residual=0.0012641604989767075
DEBUG:root:i=621 residual=0.0012477627024054527
DEBUG:root:i=622 residual=0.001241773134097457
DEBUG:root:i=623 residual=0.0012306193821132183
DEBUG:root:i=624 residual=0.0012141758343204856
DEBUG:root:i=625 residual=0.0011978083057329059
DEBUG:root:i=626 residual=0.0011918365489691496
DEBUG:root:i=627 residual=0.0011806953698396683
DEBUG:root:i=628 residual=0.001174728968180716
DEBUG:root:i=629 residual=0.0011636081617325544
DEBUG:root:i=630 residual=0.0011472605401650071
DEBUG:root:i=631 residual=0.001130856922827661
DEBUG:root:i=632 residual=0.0011249666567891836
DEBUG:root:i=633 residual=0.0011138549307361245
DEBUG:root:i=634 residual=0.0011027599684894085
DEBUG:root:i=635 residual=0.0010916013270616531
DEBUG:root:i=636 residual=0.0010857268935069442
DEBUG:root:i=637 residual=0.0010745810577645898
DEBUG:root:i=638 residual=0.0010634937789291143
DEBUG:root:i=639 residual=0.001052409177646041
DEBUG:root:i=640 residual=0.0010361088206991553
DEBUG:root:i=641 residual=0.001030263607390225
DEBUG:root:i=642 residual=0.0010244245640933514
DEBUG:root:i=643 residual=0.0010055237216874957
DEBUG:root:i=644 residual=0.0009996893350034952
DEBUG:root:i=645 residual=0.0009886317420750856
DEBUG:root:i=646 residual=0.0009801945416256785
DEBUG:root:i=647 residual=0.0009692078456282616
DEBUG:root:i=648 residual=0.000963393016718328
DEBUG:root:i=649 residual=0.0009445216273888946
DEBUG:root:i=650 residual=0.0009439390851184726
DEBUG:root:i=651 residual=0.0009329692693427205
DEBUG:root:i=652 residual=0.0009219420026056468
DEBUG:root:i=653 residual=0.000916140794288367
DEBUG:root:i=654 residual=0.00091041240375489
DEBUG:root:i=655 residual=0.0008993924711830914
DEBUG:root:i=656 residual=0.000888377777300775
DEBUG:root:i=657 residual=0.000887886097189039
DEBUG:root:i=658 residual=0.0008664248744025826
DEBUG:root:i=659 residual=0.0008633268298581243
DEBUG:root:i=660 residual=0.000854938814882189
DEBUG:root:i=661 residual=0.0008544537122361362
DEBUG:root:i=662 residual=0.0008382395026274025
DEBUG:root:i=663 residual=0.0008325381786562502
DEBUG:root:i=664 residual=0.0008294447907246649
DEBUG:root:i=665 residual=0.0008132472285069525
DEBUG:root:i=666 residual=0.0008101690909825265
DEBUG:root:i=667 residual=0.0007991992169991136
DEBUG:root:i=668 residual=0.0007882787613198161
DEBUG:root:i=669 residual=0.0007826018845662475
DEBUG:root:i=670 residual=0.000771703606005758
DEBUG:root:i=671 residual=0.0007711857324466109
DEBUG:root:i=672 residual=0.0007576728821732104
DEBUG:root:i=673 residual=0.0007546358392573893
DEBUG:root:i=674 residual=0.0007489616982638836
DEBUG:root:i=675 residual=0.0007406923687085509
DEBUG:root:i=676 residual=0.0007376488065347075
DEBUG:root:i=677 residual=0.00073192676063627
DEBUG:root:i=678 residual=0.0007210492040030658
DEBUG:root:i=679 residual=0.0007206231821328402
DEBUG:root:i=680 residual=0.0006992988055571914
DEBUG:root:i=681 residual=0.0006988740060478449
DEBUG:root:i=682 residual=0.0006932421238161623
DEBUG:root:i=683 residual=0.0006849814089946449
DEBUG:root:i=684 residual=0.0006767413578927517
DEBUG:root:i=685 residual=0.0006737083895131946
DEBUG:root:i=686 residual=0.0006654771859757602
DEBUG:root:i=687 residual=0.0006598419859074056
DEBUG:root:i=688 residual=0.0006568303215317428
DEBUG:root:i=689 residual=0.0006460535805672407
DEBUG:root:i=690 residual=0.000643051927909255
DEBUG:root:i=691 residual=0.0006374312797561288
DEBUG:root:i=692 residual=0.0006291958852671087
DEBUG:root:i=693 residual=0.000620973645709455
DEBUG:root:i=694 residual=0.0006127518718130887
DEBUG:root:i=695 residual=0.0006072078249417245
DEBUG:root:i=696 residual=0.0006042182794772089
DEBUG:root:i=697 residual=0.0006012223311699927
DEBUG:root:i=698 residual=0.0005930167972110212
DEBUG:root:i=699 residual=0.0005848630098626018
DEBUG:root:i=700 residual=0.00058188964612782
DEBUG:root:i=701 residual=0.0005815168260596693
DEBUG:root:i=702 residual=0.0005655401037074625
DEBUG:root:i=703 residual=0.0005651744431816041
DEBUG:root:i=704 residual=0.0005622187745757401
DEBUG:root:i=705 residual=0.0005593074019998312
DEBUG:root:i=706 residual=0.0005484909634105861
DEBUG:root:i=707 residual=0.0005429761367850006
DEBUG:root:i=708 residual=0.0005347817204892635
DEBUG:root:i=709 residual=0.0005292705609463155
DEBUG:root:i=710 residual=0.0005236925208009779
DEBUG:root:i=711 residual=0.0005233545089140534
DEBUG:root:i=712 residual=0.0005178505089133978
DEBUG:root:i=713 residual=0.0005070543265901506
DEBUG:root:i=714 residual=0.0005067796446382999
DEBUG:root:i=715 residual=0.0005012818146497011
DEBUG:root:i=716 residual=0.0005009461892768741
DEBUG:root:i=717 residual=0.0004928518319502473
DEBUG:root:i=718 residual=0.0004872831632383168
DEBUG:root:i=719 residual=0.000487026060000062
DEBUG:root:i=720 residual=0.00048147168126888573
DEBUG:root:i=721 residual=0.0004733531968668103
DEBUG:root:i=722 residual=0.0004705033206846565
DEBUG:root:i=723 residual=0.0004701761354226619
DEBUG:root:i=724 residual=0.00046470953384414315
DEBUG:root:i=725 residual=0.00045399353257380426
DEBUG:root:i=726 residual=0.0004484530654735863
DEBUG:root:i=727 residual=0.00044558238005265594
DEBUG:root:i=728 residual=0.0004453366855159402
DEBUG:root:i=729 residual=0.00044502774835564196
DEBUG:root:i=730 residual=0.00042912914068438113
DEBUG:root:i=731 residual=0.0004262555157765746
DEBUG:root:i=732 residual=0.0004260187270119786
DEBUG:root:i=733 residual=0.00042315220343880355
DEBUG:root:i=734 residual=0.00041762428008951247
DEBUG:root:i=735 residual=0.0004147905856370926
DEBUG:root:i=736 residual=0.00041192700155079365
DEBUG:root:i=737 residual=0.00040647652349434793
DEBUG:root:i=738 residual=0.0004036337195429951
DEBUG:root:i=739 residual=0.00039811155875213444
DEBUG:root:i=740 residual=0.00039788693538866937
DEBUG:root:i=741 residual=0.00038981682155281305
DEBUG:root:i=742 residual=0.00038958791992627084
DEBUG:root:i=743 residual=0.0003893736284226179
DEBUG:root:i=744 residual=0.00037609192077070475
DEBUG:root:i=745 residual=0.0003758730017580092
DEBUG:root:i=746 residual=0.0003756483783945441
DEBUG:root:i=747 residual=0.0003728145675268024
DEBUG:root:i=748 residual=0.000364694424206391
DEBUG:root:i=749 residual=0.00035926164127886295
DEBUG:root:i=750 residual=0.0003590506676118821
DEBUG:root:i=751 residual=0.00035621525603346527
DEBUG:root:i=752 residual=0.00034818038693629205
DEBUG:root:i=753 residual=0.00034796760883182287
DEBUG:root:i=754 residual=0.00034253011108376086
DEBUG:root:i=755 residual=0.000337094534188509
DEBUG:root:i=756 residual=0.00033689389238134027
DEBUG:root:i=757 residual=0.00033668940886855125
DEBUG:root:i=758 residual=0.00033386770519427955
DEBUG:root:i=759 residual=0.00033366840216331184
DEBUG:root:i=760 residual=0.0003282365796621889
DEBUG:root:i=761 residual=0.00031765090534463525
DEBUG:root:i=762 residual=0.0003175033489242196
DEBUG:root:i=763 residual=0.0003173124569002539
DEBUG:root:i=764 residual=0.00031447247602045536
DEBUG:root:i=765 residual=0.0003116524312645197
DEBUG:root:i=766 residual=0.000311466894345358
DEBUG:root:i=767 residual=0.0003060377493966371
DEBUG:root:i=768 residual=0.00030585855711251497
DEBUG:root:i=769 residual=0.0003004443133249879
DEBUG:root:i=770 residual=0.0002950991620309651
DEBUG:root:i=771 residual=0.000294911558739841
DEBUG:root:i=772 residual=0.0002947280299849808
DEBUG:root:i=773 residual=0.00029194686794653535
DEBUG:root:i=774 residual=0.000286539871012792
DEBUG:root:i=775 residual=0.0002811265585478395
DEBUG:root:i=776 residual=0.0002810089790727943
DEBUG:root:i=777 residual=0.0002756128378678113
DEBUG:root:i=778 residual=0.0002728649997152388
DEBUG:root:i=779 residual=0.00027268435223959386
DEBUG:root:i=780 residual=0.0002672779664862901
DEBUG:root:i=781 residual=0.0002592952223494649
DEBUG:root:i=782 residual=0.0002591207448858768
DEBUG:root:i=783 residual=0.00025895380531437695
DEBUG:root:i=784 residual=0.00025878497399389744
DEBUG:root:i=785 residual=0.0002586714690551162
DEBUG:root:i=786 residual=0.00025328199262730777
DEBUG:root:i=787 residual=0.00025311566423624754
DEBUG:root:i=788 residual=0.0002529488701839
DEBUG:root:i=789 residual=0.0002476161462254822
DEBUG:root:i=790 residual=0.00024745226255618036
DEBUG:root:i=791 residual=0.00023942641564644873
DEBUG:root:i=792 residual=0.00023939574020914733
DEBUG:root:i=793 residual=0.00023923529079183936
DEBUG:root:i=794 residual=0.00023641686129849404
DEBUG:root:i=795 residual=0.0002310865675099194
DEBUG:root:i=796 residual=0.00023093356867320836
DEBUG:root:i=797 residual=0.00022554815222974867
DEBUG:root:i=798 residual=0.00022545544197782874
DEBUG:root:i=799 residual=0.0002253009588457644
DEBUG:root:i=800 residual=0.00022514964803121984
DEBUG:root:i=801 residual=0.0002224296476924792
DEBUG:root:i=802 residual=0.00022227420413400978
DEBUG:root:i=803 residual=0.00021689203276764601
DEBUG:root:i=804 residual=0.00021681144426111132
DEBUG:root:i=805 residual=0.00021665924577973783
DEBUG:root:i=806 residual=0.00020611647050827742
DEBUG:root:i=807 residual=0.00020596393733285367
DEBUG:root:i=808 residual=0.0002058215468423441
DEBUG:root:i=809 residual=0.0002057321253232658
DEBUG:root:i=810 residual=0.0002055928052868694
DEBUG:root:i=811 residual=0.00020551170746330172
DEBUG:root:i=812 residual=0.00019753671949729323
DEBUG:root:i=813 residual=0.00019738890114240348
DEBUG:root:i=814 residual=0.00019468525715637952
DEBUG:root:i=815 residual=0.00019454641733318567
DEBUG:root:i=816 residual=0.0001892404252430424
DEBUG:root:i=817 residual=0.0001838707976276055
DEBUG:root:i=818 residual=0.00017857110651675612
DEBUG:root:i=819 residual=0.00017842755187302828
DEBUG:root:i=820 residual=0.00017835812468547374
DEBUG:root:i=821 residual=0.00017821324581746012
DEBUG:root:i=822 residual=0.00017813350132200867
DEBUG:root:i=823 residual=0.0001780016318662092
DEBUG:root:i=824 residual=0.0001779390877345577
DEBUG:root:i=825 residual=0.00017520187248010188
DEBUG:root:i=826 residual=0.00017514621140435338
DEBUG:root:i=827 residual=0.00016976991901174188
DEBUG:root:i=828 residual=0.00016970207798294723
DEBUG:root:i=829 residual=0.0001695724349701777
DEBUG:root:i=830 residual=0.00016950941062532365
DEBUG:root:i=831 residual=0.00016673716891091317
DEBUG:root:i=832 residual=0.00016407940711360425
DEBUG:root:i=833 residual=0.00016395044804085046
DEBUG:root:i=834 residual=0.00015865758177824318
DEBUG:root:i=835 residual=0.00015852526121307164
DEBUG:root:i=836 residual=0.00015846225142013282
DEBUG:root:i=837 residual=0.00015569888637401164
DEBUG:root:i=838 residual=0.0001556425995659083
DEBUG:root:i=839 residual=0.00014776544412598014
DEBUG:root:i=840 residual=0.00014763552462682128
DEBUG:root:i=841 residual=0.00014493902563117445
DEBUG:root:i=842 residual=0.00014480759273283184
DEBUG:root:i=843 residual=0.00014475059288088232
DEBUG:root:i=844 residual=0.0001446246897103265
DEBUG:root:i=845 residual=0.00014456910139415413
DEBUG:root:i=846 residual=0.00014451822789851576
DEBUG:root:i=847 residual=0.0001417988387402147
DEBUG:root:i=848 residual=0.00013910252891946584
DEBUG:root:i=849 residual=0.00014163125888444483
DEBUG:root:i=850 residual=0.0001415789738530293
DEBUG:root:i=851 residual=0.0001388823875458911
DEBUG:root:i=852 residual=0.00014141692372504622
DEBUG:root:i=853 residual=0.0001308792270720005
DEBUG:root:i=854 residual=0.00012812732893507928
DEBUG:root:i=855 residual=0.00013071524153929204
DEBUG:root:i=856 residual=0.00012544119090307504
DEBUG:root:i=857 residual=0.00012790827895514667
DEBUG:root:i=858 residual=0.00012527211220003664
DEBUG:root:i=859 residual=0.00012521825556177646
DEBUG:root:i=860 residual=0.00012769430759362876
DEBUG:root:i=861 residual=0.0001250603818334639
DEBUG:root:i=862 residual=0.0001250154891749844
DEBUG:root:i=863 residual=0.0001274971873499453
DEBUG:root:i=864 residual=0.00012485824117902666
DEBUG:root:i=865 residual=0.00012209753913339227
DEBUG:root:i=866 residual=0.00012204967788420618
DEBUG:root:i=867 residual=0.00012200909259263426
DEBUG:root:i=868 residual=0.00012190308916615322
DEBUG:root:i=869 residual=0.00011402514792280272
DEBUG:root:i=870 residual=0.00010876129090320319
DEBUG:root:i=871 residual=0.00010871746053453535
DEBUG:root:i=872 residual=0.00010595984349492937
DEBUG:root:i=873 residual=0.00010855491564143449
DEBUG:root:i=874 residual=0.00010851070692297071
DEBUG:root:i=875 residual=0.00010317761916667223
DEBUG:root:i=876 residual=0.00010049103002529591
DEBUG:root:i=877 residual=9.524422785034403e-05
DEBUG:root:i=878 residual=9.25685089896433e-05
DEBUG:root:i=879 residual=9.50847752392292e-05
DEBUG:root:i=880 residual=9.248343121726066e-05
DEBUG:root:i=881 residual=9.236137702828273e-05
DEBUG:root:i=882 residual=9.488323121331632e-05
DEBUG:root:i=883 residual=9.228100680047646e-05
DEBUG:root:i=884 residual=9.223856613971293e-05
DEBUG:root:i=885 residual=9.47005974012427e-05
DEBUG:root:i=886 residual=9.20952734304592e-05
DEBUG:root:i=887 residual=8.940757106756791e-05
DEBUG:root:i=888 residual=8.409348083660007e-05
DEBUG:root:i=889 residual=8.149665518430993e-05
DEBUG:root:i=890 residual=8.400774822803214e-05
DEBUG:root:i=891 residual=8.141891885316e-05
DEBUG:root:i=892 residual=8.130367496050894e-05
DEBUG:root:i=893 residual=8.38207924971357e-05
DEBUG:root:i=894 residual=8.122769941110164e-05
DEBUG:root:i=895 residual=8.11929494375363e-05
DEBUG:root:i=896 residual=8.364526001969352e-05
DEBUG:root:i=897 residual=8.104898734018207e-05
DEBUG:root:i=898 residual=8.100797276711091e-05
DEBUG:root:i=899 residual=8.353954035555944e-05
DEBUG:root:i=900 residual=8.08705372037366e-05
DEBUG:root:i=901 residual=7.818121957825497e-05
DEBUG:root:i=902 residual=7.815023855073377e-05
DEBUG:root:i=903 residual=7.811719842720777e-05
DEBUG:root:i=904 residual=7.801575702615082e-05
DEBUG:root:i=905 residual=7.798620208632201e-05
DEBUG:root:i=906 residual=7.795520650688559e-05
DEBUG:root:i=907 residual=7.792522956151515e-05
DEBUG:root:i=908 residual=7.788870425429195e-05
DEBUG:root:i=909 residual=7.256819662870839e-05
DEBUG:root:i=910 residual=7.519612699979916e-05
DEBUG:root:i=911 residual=6.730160384904593e-05
DEBUG:root:i=912 residual=6.727182335453108e-05
DEBUG:root:i=913 residual=6.716473581036553e-05
DEBUG:root:i=914 residual=6.459499127231538e-05
DEBUG:root:i=915 residual=6.456689879996702e-05
DEBUG:root:i=916 residual=5.9363283071434125e-05
DEBUG:root:i=917 residual=5.933278225711547e-05
DEBUG:root:i=918 residual=6.174806912895292e-05
DEBUG:root:i=919 residual=5.9182439144933596e-05
DEBUG:root:i=920 residual=5.915221117902547e-05
DEBUG:root:i=921 residual=6.165072409203276e-05
DEBUG:root:i=922 residual=5.909185711061582e-05
DEBUG:root:i=923 residual=5.898743256693706e-05
DEBUG:root:i=924 residual=6.149607361294329e-05
DEBUG:root:i=925 residual=5.89317423873581e-05
DEBUG:root:i=926 residual=5.8904606703436e-05
DEBUG:root:i=927 residual=6.141585618024692e-05
DEBUG:root:i=928 residual=5.877811781829223e-05
DEBUG:root:i=929 residual=5.875242277397774e-05
DEBUG:root:i=930 residual=6.127517553977668e-05
DEBUG:root:i=931 residual=5.869529195479117e-05
DEBUG:root:i=932 residual=5.866989158676006e-05
DEBUG:root:i=933 residual=6.119571480667219e-05
DEBUG:root:i=934 residual=5.855507333762944e-05
DEBUG:root:i=935 residual=5.586153201875277e-05
DEBUG:root:i=936 residual=5.58355895918794e-05
DEBUG:root:i=937 residual=5.581193181569688e-05
DEBUG:root:i=938 residual=5.57907551410608e-05
DEBUG:root:i=939 residual=5.576896001002751e-05
DEBUG:root:i=940 residual=5.5676802730886266e-05
DEBUG:root:i=941 residual=5.044405406806618e-05
DEBUG:root:i=942 residual=5.309347761794925e-05
DEBUG:root:i=943 residual=5.0399503379594535e-05
DEBUG:root:i=944 residual=5.304879232426174e-05
DEBUG:root:i=945 residual=5.302126010064967e-05
DEBUG:root:i=946 residual=5.025593054597266e-05
DEBUG:root:i=947 residual=5.29140597791411e-05
DEBUG:root:i=948 residual=5.2892872190568596e-05
DEBUG:root:i=949 residual=5.019015588914044e-05
DEBUG:root:i=950 residual=5.285572842694819e-05
DEBUG:root:i=951 residual=5.283709106151946e-05
DEBUG:root:i=952 residual=5.013242844142951e-05
DEBUG:root:i=953 residual=5.004596459912136e-05
DEBUG:root:i=954 residual=4.23088094976265e-05
DEBUG:root:i=955 residual=4.228854231769219e-05
DEBUG:root:i=956 residual=3.959660170949064e-05
DEBUG:root:i=957 residual=4.224154326948337e-05
DEBUG:root:i=958 residual=4.22231933043804e-05
DEBUG:root:i=959 residual=3.953385021304712e-05
DEBUG:root:i=960 residual=4.210466067888774e-05
DEBUG:root:i=961 residual=4.208528116578236e-05
DEBUG:root:i=962 residual=3.939017187803984e-05
DEBUG:root:i=963 residual=4.204688229947351e-05
DEBUG:root:i=964 residual=4.203006756142713e-05
DEBUG:root:i=965 residual=3.9331218431470916e-05
DEBUG:root:i=966 residual=4.1994600906036794e-05
DEBUG:root:i=967 residual=4.1912338929250836e-05
DEBUG:root:i=968 residual=3.92035799450241e-05
DEBUG:root:i=969 residual=4.187586091575213e-05
DEBUG:root:i=970 residual=4.1860279452521354e-05
DEBUG:root:i=971 residual=3.915259367204271e-05
DEBUG:root:i=972 residual=4.182903649052605e-05
DEBUG:root:i=973 residual=4.181400436209515e-05
DEBUG:root:i=974 residual=3.909816950908862e-05
DEBUG:root:i=975 residual=4.178503877483308e-05
DEBUG:root:i=976 residual=4.1707091440912336e-05
DEBUG:root:i=977 residual=3.898552313330583e-05
DEBUG:root:i=978 residual=4.1682887967908755e-05
DEBUG:root:i=979 residual=4.166067810729146e-05
DEBUG:root:i=980 residual=3.894181645591743e-05
DEBUG:root:i=981 residual=4.1635510569904e-05
DEBUG:root:i=982 residual=3.640389331849292e-05
DEBUG:root:i=983 residual=3.890121297445148e-05
DEBUG:root:i=984 residual=3.113187995040789e-05
DEBUG:root:i=985 residual=3.358673711773008e-05
DEBUG:root:i=986 residual=3.357545210747048e-05
DEBUG:root:i=987 residual=3.356038359925151e-05
DEBUG:root:i=988 residual=3.3544325560797006e-05
DEBUG:root:i=989 residual=3.353145075379871e-05
DEBUG:root:i=990 residual=3.351622217451222e-05
DEBUG:root:i=991 residual=3.350472979946062e-05
DEBUG:root:i=992 residual=3.3489948691567406e-05
DEBUG:root:i=993 residual=3.347683377796784e-05
DEBUG:root:i=994 residual=3.3397234801668674e-05
DEBUG:root:i=995 residual=3.0907216569175944e-05
DEBUG:root:i=996 residual=2.81765478575835e-05
DEBUG:root:i=997 residual=3.0872826755512506e-05
DEBUG:root:i=998 residual=3.086397919105366e-05
DEBUG:root:i=999 residual=2.8129637939855456e-05
INFO:root:rank=0 ranking=5.2386e+01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 ranking=5.2386e+01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 ranking=7.9439e+00 url=www.lawfareblog.com/cost-using-zero-days
INFO:root:rank=3 ranking=2.3700e+00 url=www.lawfareblog.com/lawfare-podcast-former-congressman-brian-baird-and-daniel-schuman-how-congress-can-continue-function
INFO:root:rank=4 ranking=1.5530e+00 url=www.lawfareblog.com/events
INFO:root:rank=5 ranking=1.1867e+00 url=www.lawfareblog.com/water-wars-increased-us-focus-indo-pacific
INFO:root:rank=6 ranking=1.1867e+00 url=www.lawfareblog.com/water-wars-drill-maybe-drill
INFO:root:rank=7 ranking=1.1867e+00 url=www.lawfareblog.com/water-wars-disjointed-operations-south-china-sea
INFO:root:rank=8 ranking=1.1867e+00 url=www.lawfareblog.com/water-wars-song-oil-and-fire
INFO:root:rank=9 ranking=1.1867e+00 url=www.lawfareblog.com/water-wars-us-china-divide-shangri-la
   ```

   Task 2, part 1:
   ```
   $ python3 ws_pagerank.py --data=./lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona'
INFO:root:rank=0 ranking=8.6220e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 ranking=8.6217e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 ranking=1.7695e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 ranking=1.4434e-01 url=www.lawfareblog.com/brexit-not-immune-coronavirus
INFO:root:rank=4 ranking=1.4434e-01 url=www.lawfareblog.com/rational-security-my-corona-edition
INFO:root:rank=5 ranking=1.0448e-01 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=6 ranking=9.8889e-02 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=7 ranking=9.8889e-02 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=8 ranking=9.1303e-02 url=www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
INFO:root:rank=9 ranking=8.4442e-02 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
   ```

   Task 2, part 2:
   ```
   $ python3 ws_pagerank.py --data=./lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona' --search_query='-corona'
INFO:root:rank=0 ranking=0.0000e+00 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 ranking=0.0000e+00 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 ranking=0.0000e+00 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 ranking=0.0000e+00 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=4 ranking=0.0000e+00 url=www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
INFO:root:rank=5 ranking=0.0000e+00 url=www.lawfareblog.com/fault-lines-foreign-policy-quarantined
INFO:root:rank=6 ranking=0.0000e+00 url=www.lawfareblog.com/limits-world-health-organization
INFO:root:rank=7 ranking=0.0000e+00 url=www.lawfareblog.com/chinatalk-dispatches-shanghai-beijing-and-hong-kong
INFO:root:rank=8 ranking=0.0000e+00 url=www.lawfareblog.com/us-moves-dismiss-case-against-company-linked-ira-troll-farm
INFO:root:rank=9 ranking=0.0000e+00 url=www.lawfareblog.com/livestream-house-armed-services-committee-holds-hearing-priorities-missile-defense
   ```

3. Ensure that all your changes to the `pagerank.py` and `README.md` files are committed to your repo and pushed to github.

4. Get at least 5 stars on your repo.
   (You made trade stars with other students in the class.)

   > **NOTE:**
   > 
   > Recruiters use github profiles to determine who to hire,
   > and pagerank is used to rank user profiles and projects.
   > Links in this graph correspond to who has starred/followed who's repo.
   > By getting more stars on your repo, you'll be increasing your github pagerank, which increases the likelihood that recruiters will hire you.
   > To see an example, [perform a search for `data mining`](https://github.com/search?q=data+mining).
   > Notice that the results are returned "approximately" ranked by the number of stars,
   > but because "some stars count more than others" the results are not exactly ranked by the number of stars.
   > (I asked you not to fork this repo because forks are ranked lower than non-forks.)
   >
   > In some sense, we are doing a "dual problem" to data mining by getting these stars.
   > Recruiters are using data mining to find out who the best people to recruit are,
   > and we are hacking their data mining algorithms by making those algorithms select you instead of someone else.
   >
   > If you're interested in exploring this idea further, here's a python tutorial for extracting GitHub's social graph: <https://www.oreilly.com/library/view/mining-the-social/9781449368180/ch07.html> ; if you're interested in learning more about how recruiters use github profiles, read this Hacker News post: <https://news.ycombinator.com/item?id=19413348>.

5. Submit the url of your repo to sakai.

   Each part is worth 2 points, for 12 points overall.
