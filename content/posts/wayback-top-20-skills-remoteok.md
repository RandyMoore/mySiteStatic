---
title: "Wayback Top 20 Skills Over Time - remoteok.io"
date: 2021-05-01T00:00:00+00:00
draft: false
---
V2 of the last blog post. Last time job postings were collected every few weeks in real time from remoteok.io. An alternative is to use the [Wayback Machine](https://archive.org/web/) to collect the postings. The result:

{{< youtube S7k7eDsB8AE >}}

### Legend
* \# Posts: Total number of posts on remoteok.io saved at archive.org for that month
* Total # NEs: The sum of counts of the top 20 Named Entities (skills)

# Pipeline
1. [Fetch the posts from the Wayback Machine](https://github.com/RandyMoore/Jobs/blob/master/fetch_wayback.py)
2. [Process the posts generating top 50 named entities for each month](https://github.com/RandyMoore/Jobs/blob/master/process_wayback.py)
3. Manually reduce the top 50 named entities to the top 20 skills for each month
4. Run [a script](https://github.com/RandyMoore/Jobs/blob/master/blender.py) in [Blender (2.83 LTS)](https://www.blender.org/download/lts/) on the top 20 skills to create the animation
5. Render the animation in Blender to a video file

# Lessons Learned

## Find Existing Sources of Data
The most obvious lesson. Instead of sampling over a long period of time seek existing sources of data. Here the objective is to discover what are the most in demand software development skills. Many preexisting sources are available for this kind of information.

## Elbow Grease Makes a Difference
A different approach was taken compared to the last time. This time the top 50 named entities were generated, then reduced down to the top 20 skills by hand for each month.

* Instead of [ignoring a named entity if a substring matches an ignore word](https://github.com/RandyMoore/Jobs/blob/617c0583a7b91f199de02fe02b0fe8a299eca072/process.py#L99), only ignore it if [the whole string matches an ignore string](https://github.com/RandyMoore/Jobs/blob/617c0583a7b91f199de02fe02b0fe8a299eca072/process_wayback.py#L42). This time named entities reflecting actual skills did not get discarded due to a substring match on an ignore word. It was not obvious valid skills were being discarded by the old script; only working with the data now the issue became visible.
* Existing synonym map kept to help merging process but manual merging also done due to the copious variety of synonyms. Again here skills were miscounted in the old script (synonyms appearing beyond the 20th rank).
* Discovered "Happiness Engineer" is a role / skill (seen in 2017-05). This alone was worth all the effort.
 * A much better sense of the data was acquired. For a one-time effort the manual approach is the right way. For a repeated process working with the data manually and then programming the know-how into a machine makes more sense.
 * All data science courses teach that you need to get your hands dirty to really understand the data, it is true.

## Have Clear Rules for How To Interpret the Data
What counts as a skill? Manually working with the data exposed the nuance involved. Here, languages (English, German, ...) are included as skills but very broad terms (Engineer) are not. Perceived applicability of a named entity as a skill worth learning was the rule of thumb for this toy exercise. The detail and rigor of the rules should trend with the seriousness of the subject and audience. Discovering and refining rules is an iterative process, much like [coding qualitative data](https://en.wikipedia.org/wiki/Qualitative_research#Coding).

## Seek Large Amounts of Evenly Distributed Data
Using the WayBack machine seemed like a great idea. Plenty of data was available. Unfortunately the distribution was not ideal. Many months are missing posts, and the majority of the posts are clustered across a few months. This project went ahead anyway, partially out of curiosity to see how months with sparse data compared to those flush with data.

Fortunately the Wayback Machine website shows you the distribution of the data up front:

![Job Posting Distribution](/mySiteStatic/images/RemoteioDistribution.png)

## Use LTS Tool Versions
The [Blender script](https://github.com/RandyMoore/Jobs/blob/master/blender.py) does not work in the most recent version of Blender (>=2.9x). Fortunately an LTS version of Blender was used and so it was easy to download the latest LTS version and get up and running quickly again.

## Write Clear Code
[The original version of the processing code](https://github.com/RandyMoore/Jobs/blob/master/process.py) is not well written in terms of clarity. Specifically, returning tuples from functions is hard to reason about. [The new version of the script](https://github.com/RandyMoore/Jobs/blob/master/process_wayback.py) is a bit less complex in terms of data flow within the program. Even if software is meant as a personal toy it makes sense to invest some time in quality, at least for the future self.
