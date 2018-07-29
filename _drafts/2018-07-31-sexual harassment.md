---
layout: post
title:  "What does critical data science add to our understanding of sexual harassment in academia?"
date:   2018-07-02 15:07:19
categories: [essay]
comments: true
image:
  feature: harassment/header.png
---

*A cautious introduction to NLP and Machine Learning methods in analyzing thousands of anonymous sexual harassment & assault reports.* 

## Introduction

> "The data are too messy."
>
> "There's no way we could work through it in time."
>
> "I'd like to figure something out. But I don't know."


I was sitting in the grad student lounge in Ann Arbor with three classmates from Information Visualization. Each of us had the same Google Sheet pulled up on our browsers: "[Sexual Harassment In the Academy: A Crowdsource Survey. By Dr. Karen Kelsky, of The Professor Is In](https://docs.google.com/spreadsheets/d/1S9KShDLvU7C-KkgEevYTHXr3F6InTenrBsS9yk-8C5M/edit#gid=1530077352)." In just a few months, a [call for anonymous survey submissions](http://theprofessorisin.com/2017/12/08/our-metoophd-moment/) in the popular The Professor is In blog had resulted in over 2,300 reports of sexual harassment and assault.

Our group quickly realized a few things: this data was immense. It was messy in the sense that data folks often describe messy data: non-standard, full of missing values and strange capitalizations. It was also data that spoke to an immensity of pain, the loss of futures diverted and destroyed.One peer described the issues in the data, column by column, but she stopped when it came to the "Event" column. We reached a point where we stopped knowing what to say, and just made eye contact with each another. Eventually, our group passed on the Sexual Harassment dataset in favor of a climate change-related project (which generated its own share of complex data issues). 
<!--more-->

However, I couldn't stop thinking about the survey. As somebody who is passionate about finding critical approaches to data science, I appreciated the importance of this unprecedented data. Thousands of individuals had revealed their experiences with sexual harassment and assault in academia, many apparently for the first time. These reports detailed the devastating effect these events had on their lives. This is vital, powerful data that deserves to have its story told.

When I had the opportunity to propose an individual project for my Data Manipulation & Analysis course, I petitioned successfully to build something around the sexual harassment data. In this post, I describe my semester-long attempt to use the toolkit of natural language processing and machine learning - specifically, the tools of **topic modeling** and **text classification** - hand-in-hand with a reflexive process of generating, challenging, and revising research questions.

After finishing the project, I felt the need to write about it more openly. I had done different types of data analysis and machine learning-ish tasks before, but this was the first time that I grappled so much with both the power of these methods and their dangers. In this post, I describe my process of identifying initial questions, implementing NLP using PySpark’s MLLib library (focusing on topic modeling and feature analysis of classifiers), and reflecting on the findings. I intend for this to be useful as both a big-picture technical primer, a reflection on methodology, and perhaps identifying some specific questions in sexual harassment in academia to further pursue.

Here's my tl;dr: Critical data science, as both a set of methods and an ethical stance, can absolutely play a part in challenging our assumptions, redirecting our attention, and prompting us to ask new and more careful questions. I also feel we must be vigilant in resisting the shift to reify and make empirical the outputs of algorithms, especially when claims of objectivity or strength of evidence have immense real-world impact for targets of harassment and survivors of assault.



## The Dataset

The Sexual Harassment in Academia dataset is a collection of responses to an open Google Survey. For my analysis, I download the data on April 4, 2018, when 2,438 reports had been submitted between December 1st, 2017 and April 4, 2018:

[Here is a website](http://people.csail.mit.edu/karger/Exhibit/Harass/) providing a faceted interface for accessing the data (the introduction also linking to the survey results, a blog post by its creator, etc.)

![]({{ site.url }}/img/harassment/image1.png)

This version uses a slightly modified version of the original survey results adds a few additional columns that attempt to clean or otherwise aggregate data. However, at the time of analysis those standardization attempts were still scattershot and only partially implemented (and as of July 2018 continue to be in-progress).

The survey includes only two categorical variables, presented as multiple choice options: gender of perpetrator (woman/man/non-binary/unsure/various/other\_\_) and type of institution (small liberal arts college/Elite Institution/Ivy League/Other R1/etc.) Otherwise, responses are all open text fields, and many are optional (or accept a blank or “none” as submission).

These text field submissions include:

> Event description, target status (e.g. student, professor), perpetrator status (e.g. professor/chair/fellow graduate student/etc.), name of the institution (optional), field discipline, institutional response (if any), institutional/career consequences for perp (if any), impact on your career, impact on your mental health, impact on life trajectory/choices, and other.

In my analysis, I make a distinction between data that describes the nature of the encounter (such as role of perpetrator and event description), the consequences for the perpetrator (response, punishment), and the long-term impact on the target (mental health, career, and life impacts). The survey does an excellent job in affording us a vivid picture of harassment across time, and laying the groundwork for investigating the correlations between aspects of the event and what comes after.



## The Initial Question

I knew that no matter what this type of analysis found, it would have to grapple with a basic question: **what does NLP/machine learning contribute that makes it worth paying attention to when we have the words of targets & survivors in front of us?**

This question had two main implications for me. First, it pointed to the possibility that there may be aspects of the experiences of these 3,000+ individuals that we may not be able to grasp through rote, line-by-line reading alone. I mean *possibility* here sincerely, as I was unsure what those elusive structures might hold, if anything.

Second, it called attention to the very real impact of prioritizing some types of data over others. As my mentor Justin Joque observed when I discussed this project with him this spring, administrators and other gatekeepers use the idea of threshold of objectivity to dismiss individual testimony of sexual harassment and assault. (The wider public uses similar rhetoric when using legal thresholds to reject individual reports as slander, not containing sufficient evidence, etc.) By claiming that algorithms can unearth new data from text responses in the aggregate, we must be careful not to reinforce this toxic idea that narratives alone are never enough to identify and combat systematic abuse.

Data science as a way of producing insight also brings with it an ideology prevalent in statistics and computational methods: *this thing is real because we've measured it*. This is especially powerful in something like textual survey responses. No matter how real these words felt to my group members and myself sitting around a table in Ann Arbor, most university discipline boards and other institutional actors ascribe greater power to what they perceive as objective evidence. At the same time, identifying implicit structures and other *unknown unknowns* can be an impetus to turn our attention back on these narratives with renewed intensity. Therein lies the opportunity, and the risk, in exploring texts of individual narratives.


## The Research Questions

With these concerns in mind, I attempted to surface elements in the survey responses that may not have been visible from the texts alone - but that also draw the researcher's attention *back into the reports* rather than abstracting the specificity away. I include four research questions in the original report (see the Notes below), but here I'm going to focus on three research questions that I’ve gathered into two categories: 

<br>

#### Topic modeling of responses: {#topic-modeling-of-responses-1}

**Q1**: What are the main themes we might infer from reports of harassment/assault events?

#### Feature analysis of classifiers:

**Q2**: What language in incident descriptions is most predictive of whether the perpetrator was a professor/faculty/department chair or not? How powerful is this prediction?

**Q3**: What language in the incident report is most predictive of whether the respondent “left” or “quit” their position/academia/the discipline/etc? How powerful is this prediction (in general and relative to the professor model)?

<br>


Below, I will describe the rationale for each approach and the methods I used to take them on.

## Methods: Topic Modeling

"I'm wondering not only *what* the topics are, but *of what order*..."

A doctoral student I had worked with on topic modeling posed this question in his presentation last week. It struck me as an exceptionally sharp way to articulate something I had only just started exploring in my own project. That is, what do the topics in topic modeling reveal about the domain of responses in these surveys, their types, and their relative position to one another with regard to hierarchies of all kinds? I was curious about experiences, attitudes, beliefs, and other elements I had not anticipated ahead of time.

In order to generate topics, I needed to prepare the surveys to go through a machine learning pipeline in Python, which I decided to implement in Apache Spark. This consisted of several steps, which I’ll describe in broader terms and with a few notes on implementation:

#### Cleaning and preprocessing the data.

I stored values within a Spark DataFrame structure (which behaves similarly to tables in SQL). After simplifying and standardizing the text (lower case, no punctuation), and battling nulls and missing values, it was ready for the pipeline.

#### Building the pipeline

I decided upon Latent Dirichlet allocation(LDA) as my algorithm of choice for generating models. Here is a [helpful primer](https://opendatascience.com/topic-modeling-with-lda-introduction/) on the intuition behind generating topics with LDA.

To get LDA up and running, I needed to construct a full machine learning pipeline. I used the MLLib library in PySpark (the Python implementation of the Apache Spark cluster-computing framework). While there are tradeoffs to using MLLib as opposed to a single-process approach (e.g. with TensorFlow), I wanted to teach myself more about distributed computing and the Spark way of doing things in this project. I also wanted to implement pipelines in a way that could scale to much bigger datasets later on. (I used the Databricks cloud-hosted platform to run my Spark code, but look for a future blogpost about running Spark/MLLib locally in JupyterLab using the BeakerX extension.)

The PySpark MLLib pipeline consisted of (1) tokenizing all words within a particular column, such as “event” description, (2) removing stopwords, (3) implementing a vectorizer (CountVectorizer) in order to represent each entry as a count of unique words, and (4) implementing a LDA model that outputs k topics for column c in the data.

In this pipeline, a paragraph of text response is converted to a wordcount of each unique words (steps 1-3). Stopwords are discarded at step 2, and any words that fail to meet a minimum threshold of frequency across all responses (i.e. any words that fall below the top 3,000 most frequent words in the corpus) are also discarded by the CountVectorizer function (step 3). The output is a list of topics, with words listed in order of their relevance to the topic (the first or left-most word being the most significantly correlated with the topic, which is not given a specific name).

#### Tweaking parameters

I spent some time adjusting the number of topics. While I wanted to maintain low log perplexity and high log likelihood (my two evaluation metrics), I also wanted to maximize (subjective) intelligibility for a human reader. I settled on k=15 as a balance between acceptable evaluation metrics and high coherence for a human reader.

#### Analysis

Finally, with the topic modeling pipeline constructed, I fed in survey responses from each response category individually, in order to highlight patterns more closely.



## Methods: Feature analysis of classification

While the survey results included very few categorical variables relative to the many open text-field responses, there were nonetheless clear patterns in those text responses that suggested possible categorizations.

Two jumped out at me early on. First, while there wasn't a controlled vocabulary for the "perpetrator" response, there was nonetheless a very clear subset of perpetrators who were faculty, staff, department chairs or heads, etc., and non-faculty perpetrators. The survey had thus anticipated the role of the perpetrator would be important, but hadn't included a clear categorical option for describing that perpetrator's faculty status.

Second, I noticed a disturbing trend: many of the survey respondents described quitting or leaving their job or career in the life consequences response fields. That so many individuals described their life so profoundly altered by these acts of harassment and assault struck me as of profound importance. While the survey had attempted to capture life impacts in general terms, it again did not anticipate this profound outcome through use of a categorical response.

I thought a lot about these two implicit categories. How could I explore their relationship to other aspects of the survey response? I took inspiration from the idea of *feature importance analysis* which appeared in an earlier assignment. The idea: when building a model to predict the correct class of some set of data, the ultimate goal isn't necessarily to have that model prepared and ready for data in the future. Instead, you can use the modeling process as a way to measure **what features of the data are most significant in predicting an outcome relative to all other features**. You can also determine the overall strength of the model.

The intuition from there is that: if you build a model for predicting one categorical variable in your data based on some other data, and the model has a high accuracy, you may wish to investigate the most important features used in this model. Why were these features so successful in producing the right classification? What's going on here?

My main interest was to see to what degree I could generate a realistically predictive model for each of the two categories (prof/non prof, or left/not left) based on the incident descriptions. If so, what specific fields of description (such as the event, the punishment, etc.) most strongly predicted professor/no-professor perpetrators? And given those fields, what features (in this case, words within the description) are the strongest predictors within the model?

#### Generate categorical variables/extract features

To categorize entries as professor and non-professor, I added a new column, “IsProf”, which contained a boolean value indicating if the substrings "prof", "faculty", or "chair" were found in the perpetrator field. I then followed the same procedure to generate isLeft based on "left" and "quit" appearing in the life and career impact columns.

(After some trial and error eventually realized that MLLib performed better if IsProf and IsLeft were represented as 1/0 instead of True/False. Spark was tricky in this way, among others!)

#### Construct pipeline to support classification

I approached this question as a machine learning/classification task, and built a classifier pipeline using PySpark with a Random Forest classifier. Here is a helpful introduction to [Random Forest classification](https://towardsdatascience.com/the-random-forest-algorithm-d457d499ffcd) that also touches on the concept of feature importance.

To carry out classification, I establishes a new pipeline using PySpark’s MLLib library. I used many elements of the earlier LDA topic modeling pipeline, such as starting with stop words removal and tokenization. As with LDA, I included a CountVectorizer vectorizer in order to represent entries in a given column as a collection of word counts, given a specific size of dictionary and minimum document frequency for terms. These counts act as features that I then passed into a Random Forest classifier (which is, technically, a meta-classifier of tree classifiers). Finally, I set the one-hot “IsProf” variable as the target label for classification.

This CountVectorizer + Random Forest approach felt like a clear way to pursue classification without further transforming the feature set (for example, I thought about but eventually decided against word embeddings, as I wasn’t sure if the semantic assumptions of a word embedding approach would overpower the data given the small-moderate sample size).

#### Training and testing

I trained and tested the model on a 80/20 split of the input data (after first forgetting to create a training set! I describe this in the results section.)

#### Output feature importance and overall model strength

I outputted accuracy as a percentage of successful predictions, and finally outputted a list of features ranked by importance in the model. I printed the success rate to screen and visualized the features ranked by importance as a seaborn barplot.

## The Findings


### Q1: What are the main themes (or topics) in reports of harassment/assault event?

I described above the importance of interpreting topics as possible relationships, hierarchies, and domains of inquiry. The topics that resulted from this process certainly suggest a number of points of investigation within each subcorpus – in this case, the total set of responses to a given question. I believe these findings may correspond to real patterns in incidents, outcomes, and impacts worth investigating further.

To illustrate patterns in the context of the harassment/assault event, here is a table of topics from the “event” field of responses:

![]({{ site.url }}/img/harassment/image2.png)

These topics provoke a number of questions. Is there an association between the “dean” position and long-term, sustained harassment (“years”, “several”, “involved”?) When events appear to a peer (“student”, “grad”, “department”) in the final topic, why does “lab” show up frequently? How might “removed” or “separate” relate to a common strategy of shuffling around perpetrators without terminating employment? The relationship between “pressured, “raped,” and “groomed” seems to tell a specific story.

To illustrate patterns in impact on the target, here is a table of topics from the “mental” field of responses:

![]({{ site.url }}/img/harassment/image3.png)

These topics suggest how devastating the effects of sexual harassment and assault can be. Impacts range from higher stress and anxiety, distrust of men and authority figures, depression + ptsd, all the way to embarrassment and hostility, unhealthy dynamics, and powerlessness and being denied within an institution.

Finally, here is a table of topics in the punishment category:

![]({{ site.url }}/img/harassment/image4.png)

In these responses, we find very little evidence of a repercussion like firing, although “removed” does appear once. Instead, topics seem emphasize none/no/unknown/na punishment, and even point to some positive outcomes such as “promoted”, “get”, “tenure”.

These topic results alone do not provide conclusive evidence of the distribution of descriptions for these categories. However, they do sketch out the domain of possibilities and show relationships that tell cohesive stories about common elements in these narratives.



### Q2: What language in incident descriptions is most predictive of whether the perpetrator was a professor/faculty/department chair or not? How powerful is this prediction?

After much tweaking, the first classification model used to predict professor/not professor perpetrators resulted in modest success. The most accurate prediction occurred with the “event” column – however, I chose to exclude this result given that the most accurate feature by far was “professor,” and simply mentioning a professor in both the perpetrator and event field didn’t feel meaningful in any particular way. I instead focused on the perpetrator outcome and target impact.

At first, I thought I had achieved a very strong result: accuracies in the range of .72 to .80 (that is, 72% to 80% of the test subset being classified correctly)! However, I noticed my mistake: I had failed to create a distinct test and training sets. The model had clearly been overfitted.

Once I created a test/train split correctly (oof), the correctly fit model ranged from .59 to .637 accuracy in predicting professor status.

I felt that, given these very low numbers in terms of classification success, it would be misleading to try to interpret the featureImportance figures in great detail. For instance, here is the raw output of the most successful classification task (.637 accuracy) which occurred with the mental health impacts category:

![]({{ site.url }}/img/harassment/image5.png)

I am not sure how to interpret this outcome. Mentioning “think”, or a blank entry (somehow this persisted even after attempting to remove nulls at length!) do not easily lead to a real-world interpretation, and “academia” and “women”, while interesting, elude easy interpretation as well.

The main finding from this question is that the survey descriptions only weakly predict professor/no professor. Perhaps another approach would lead to a more successful model, but at the moment, the relationship doesn’t appear to be very strong.



### Q3: What language in the incident report is most predictive of whether the respondent “left” or “quit” their position/academia/the discipline/etc.? How powerful is this prediction (in general and relative to the professor model)?

In the previous step, we achieved classification success of just .59 to .637 in the attempt to predict professor status of perpetrator. In contrast, I discovered that the task of predicting the target leaving or quitting a job/academia/etc. based on either descriptions was a much higher success rate of 79.3%! This signifies much stronger evidence of predictive power in this model. In other words, it appeared the existing data lends itself much more strongly to predicting whether an individual will leave academia or quit their job after an incident – the classifier succeeds 4 out of 5 times.

From a domain perspective, this finding has a strong potential to tell a compelling story about how sexual harassment and assault incidents drastically alter the lives of targets. Given the moderately strong predictive power of this model, I felt it was appropriate to delve into the specific features and their relative importance. This resulted in what I believe to be some of the strongest findings of the project:

![]({{ site.url }}/img/harassment/image6.png)

Analyzing the “response” descriptions in order to predict a “left”/”quit” outcome reveals that the strongest predictors appear to be a non-response (“none”, blank, “nothing”, “never”) or mention of a supervisor (“boss”).

An incredibly important caveat is that we cannot, from the results above, definitively state the *direction* of this association. It’s possible, for example, that a “none” response is correlated with *not leaving* academia. What we can say more generally is that, if we know about the way a department or institution responds to sexual harassment/assault, we can develop a pretty good guess of whether the target will eventually leave their graduate program or their academic career.

Despite the caveats, this exploratory finding might give us pause and prompt us to consider how a non-response might be systematically encouraging individuals to abandon their studies and careers.

![]({{ site.url }}/img/harassment/image7.png)

Quite similarly, events that include references to people in direct positions of power (“supervisor”, “dean”) appear predictive of an individual leaving or quitting their position or academic career. Again we see the lab setting referenced meaningfully here. I am very curious what “comments” might mean in this context – perhaps a reference to how the incident is discussed?

These two visualizations point to a possible correlation between individuals experiencing assault and harassment from their direct supervisors and eventually leaving or quitting their career/academic pursuits, as well as a relationship between non-responses after incidents and leaving/quitting. This might compel us to direct our attention towards (1) the direct supervision of individuals and (2) peer interactions in departmental spaces such as labs as focal areas for gathering more data about sexual harassment and assault.



## Conclusion

At the outset of this analysis, I posed the question: **what does NLP/machine learning contribute that makes it worth paying attention to when we have the words of targets & survivors in front of us?**

What if the outcome of computational text analysis was not a set of algorithmically-divined observations so much as a new set of processes and ethics of investigation? What if instead of waiting for disciplinary boards and committees to find evidence of assault that rises above some satisfactory threshold, we in academic communities start investigating the safeguards (or lack thereof) that protect students from predatory advisors and supervisors? What if we demand focus on the profound structural factors that discourage targets from reporting perpetrators, or the cumulative effect of watching faculty member after faculty member receive no substantial consequence from their actions, or the experiences of individuals who end up leaving their professions and studies entirely?

Natural language processing and machine learning algorithms are often criticized for promising incredible empirical insights while remaining totally opaque in regards to how they are producing those results – and that criticism is entirely fair! What I’m curious about is what happens when, rather than leaning on these tools to predict outcomes or manufacture new evidence, we incorporate them into a process of interrogating structures of power along with our own blind-spots.

The process of exploratory data science I’ve outlined here is, by its nature, deeply iterative. Data leads to analysis, which leads to more carefully gathered and framed data, which leads to new forms of analysis, which leads to yet more data. But while it takes time to develop the skills to get an LDA topic modeling pipeline up and running, it isn’t contingent on a year-long peer review process for a journal publication, or a department’s internal process for responding to complaints against its faculty members. Perhaps this type of investigation may belong within an academic research process, or perhaps it can be utilized by graduate student unions, whistleblowers within academic institutions, investigative media outlets, or whomever else wishes to derive practical, specific actions from complex data.

My hope is that you find something useful in these questions and observations – whether that means continuing to developing more nuanced, powerful methods of data collection and analysis to confront sexual harassment in academia, or as an impetus to get excited about critical data science in the context of the questions you care about most deeply. I am still finding my voice within this realm of code and communities and data, and plan to continue adding tutorials, methodological reflections, and weird little hybrids to this space. Onwards!

## Acknowledgments

My sincere thanks to Justin Joque for helping me think through the “critical” component of critical data science in this project and beyond. Thank you to my readers Sarah and Diana who genenrously offered feedback on an early draft to shape this essay for the better.

## Notes

In the original report submitted for course credit, I also included a round of analysis using ngrams (an additional NLP strategy), which I excluded here in order to provide a more focused introduction to the ML-intensive methods in Spark. To read more about those methods (e.g. using the pandas library) and see those findings (which closely mirror the trends re: lack of repercussions and importance of supervision relationships/common spaces like labs/etc.), you may wish to review my original report [here](https://github.com/zoews/academic-harassment-data/blob/master/si618wn2018_report_zoews.pdf).

