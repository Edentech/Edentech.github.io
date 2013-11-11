---
layout: post
title:  "Building Garland"
date:   2013-11-09 10:15:00
categories: product
author:  Matt Mondok
---

<img src="/assets/garland.png" class="right" alt="Garland"> We're coming up on the two-year anniversary of [Garland][garland], and it's pretty interesting to reflect on how we went from a rudimentary search interface and manually importing data to a large-scale scale, self-sufficient web-based application.

In case you aren't familiar with the product, Garland is a service that we offer which allows users to verify both employees and vendors against government-provided exclusion lists.  The most popular list, the [System for Award Management][sam], is provided by the federal government.  Other lists such as the [State of New Jersey Debarment Report][nj] are provided by the individual states.  Mandates vary state by state, but most of our users are in the system weekly in order to meet different compliance laws.  While it doesn't necessarily pertain to this post, it's also worth noting that we provide Garland as a free service without limitations -- it's effectively a way of saying 'thanks' to our clients and (potential) customers.  As of this writing, Garland supports 10 different databases that are updated at least once per day if not more.

That's the end of the story.  Where we began was two years ago when I received an email from our managing partner Lou Liberio asking if I knew of an easy way to search a database called LEIE.  When we couldn't find anything powerful enough, we decided to build it out.  As for the name, every time I saw LEIE I thought of a Hawaiian lei or wreath, hence why it's called Garland.  

Lou needed something done in a week's time, so we put together a Ruby on Rails app as fast as we possibly could.  It was not pretty, and loading data was very manual, but it worked.  You could enter your data, hit a 'Run Report' button, and get a list of exact and fuzzy matches.  Searches against MySQL weren't slow, but when you needed to run 800 to 1000 searches, the process took about a minute or so.  The fuzzy searches were LIKE clauses, so fuzzy was a misnomer at best.  Data had to be loaded record by record, too.  There was no import function.  Nevertheless, it worked and we launched.  Our customer was thrilled.  This was truly a [minimum viable product (MVP)][mvp].  

After a few weeks, we added another customer.  They were thrilled as well, but the manual data input process was excruciating.  Lou said, "Matt, we have to do something about this - can you write an import routine?"  We did, and again our customers were happy.  

Another week passed and all was good, but the elephant in the room was that the fuzzy search was almost useless.  Garland turned out plenty of 'fuzzy' matches, but there was no ranking system and it was difficult to sift through everything.  It was time to incorporate a real search interface.  Having worked with [Apache Solr and Lucene][solr] in the past, that was our hammer.  If you had worked with Solr two years ago, you probably know that fuzzy search wasn't all that straight forward.  It's actually much easier today.  In any case, after some experimenting we came up with a configuration that was a marked improvement.  We added in some custom boosting and the search was ready for prime time.  Outside of some strange edge cases, we had a winner on our hands.  We stood up another Amazon EC2 instance, wrote a new ETL process, and pushed it out.  

Here's the point where I fess up to our biggest mistake:  we wrote Garland with the idea that it would only support the LEIE dataset.  I remember Lou asking, "Can we add Medicheck to our reports, and can we add the System for Award Management dataset, too?"  The knee-jerk response was something along the lines of, "no."  Before he could reply again, we started refactoring our object models to be more general and better fit for multiple data sources.  Generally speaking, Ruby is easy to refactor -- especially if you have tests of which we had only a few.  In any case, after a few hours, we had a generic architecture in place.  If we could find the data, it could be manually loaded.  That meant pulling down the files, pushing them to the server, and running a rake import task.  It was a pain, but we agreed to only do it once per week. 

We did this for nearly a year until we decided it was time to optimize.  We had grown out by about 50 customers and the demand had grown for more frequent data updates.  We decided to move the app over to Heroku in order to take advantage of the easily scalable dynos and minimum support overhead.  At this point, it was time to go from minimum to an all-out product.  The original interface was ugly so we redesigned it using Bootstrap 2.  We migrated the data over to Postgres from MySQL.  We also decided to switch from Solr to Postgres's [trigram matching][tg] for fuzzy search (a topic for a whole other post).  The most effective change we made was moving away from DelayedJob and onto Sidekiq.  We were constantly babysitting DelayedJob, and one of our senior engineers strongly suggest Sidekiq.  He was right:  Sidekiq ran (and still runs) like a champ.  We were already using Redis behind the scenes, so introducing Sidekiq was friction-free.  We haven't had an issue with background jobs since.

Garland was in great shape.  The last thing we needed to do was automate the import process.  For all ten data sources we support, each one came with its own nuances that made automation difficult.  Heroku's lack of a writable file system made the task even more challenging, and storing 50,000 records in application memory is not wise.  We ended up using Redis as a temporary buffer, wrote custom importers for each source (relying heavily on Nokogiri), and finally had it all working.  Garland was now able to import all of the data sources automatically without human intervention.  

The icing on the cake for all of this was adding in "pretty" reports and an inline search (Google-esque search).  Until this point, in order to generate a matching report, users first had to import all of their employees and vendors.  There was no way to just search for one term without the import.  Adding the inline search solved that.  Inline search isn't a popular use case, but it is nice to have.    

That's the story of Garland going from a minimum viable product to the rather powerful application that it is today.  What's most interesting is that, during this whole process, we never really thought we were doing MVP.  We were simply doing what we felt made the application better (and what our customers needed).  It goes without saying that not every product should be built in this manner, but it's been a real pleasure to see a product grow in an MVP-style approach.



[garland]: https://www.garlandlive.net/
[sam]: https://www.sam.gov/portal/public/SAM/
[nj]: http://www.state.nj.us/treasury/debarred/
[mvp]: http://en.wikipedia.org/wiki/Minimum_viable_product
[solr]: http://lucene.apache.org/solr/
[tg]: http://www.postgresql.org/docs/9.1/static/pgtrgm.html
