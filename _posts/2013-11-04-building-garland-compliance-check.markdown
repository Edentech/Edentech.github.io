---
layout: post
title:  "Building Garland"
date:   2013-11-04 08:15:00
categories: company
author:  Matt Mondok
---

<img src="/assets/garland.png" class="right" alt="Garland"> We're coming up on the two-year anniversary of [Garland][garland], and I think it's pretty interesting to reflect on how we went from a rudimentary search interface and manually importing data to a large-scale scale, self-sufficient web-based application.

In case you aren't familiar with the product, Garland is a service that we offer which allows users to verify both employees and vendors against government-provided exclusion lists.  The most popular list, the [System for Award Management][sam], is provided by the federal government.  Other lists such as the [State of New Jersey Debarment Report][nj] are provided by the individual states.  Mandates vary state by state, but most of our users are in the system weekly in order to meet different compliance laws.  While it doesn't necessarily pertain to this post, it's also worth noting that we provide Garland as a free service without limitations -- it's effectively a way of saying 'thanks' to our clients and (potential) customers.  As of this writing, Garland supports 10 different databases that are updated at least once per day if not more.

That's the end of the story.  Where we began was two years ago when I received an email from our managing partner Lou Liberio asking if I knew of an easy way to search a database called LEIE.  When we couldn't find anything powerful enough, we decided to build it out.  As for the name, every time I saw LEIE I thought of a Hawaiian lei or wreath, hence why it's called Garland.  

Lou needed something done in a week's time, so I slapped a Ruby on Rails app together as fast as I possibly could.  It was not pretty, and loading data was very manual, but it worked.  You could enter a term, hit search, and get a list of exact hits and fuzzy hits.  The front end was Rails and the backend was MySQL.  Each individual search was fast, but when you needed to run 800 to 1000 searches, it took about a minute or so.  The fuzzy searches were simply LIKE clauses, so fuzzy was a misnomer at best.  Data had to be loaded record by record, too.  There was no import function.  Nevertheless, it worked and we launched.  Our customer was thrilled.  This was truly a [minimum viable product (MVP)][mvp].  

After a few weeks, we added another customer.  They were thrilled as well, but the data input process was excruciating.  Lou said, "Matt, we have to do something about this - can you write an import routine?"  Done, and again our customers were happy.  

Another week passed and all was good, but the elephant in the room is that the fuzzy search was almost useless.  It turned out plenty of 'fuzzy' matches, but there was no ranking system and it was difficult to search through everything.  At this point, it was time to incorporate a real search interface.  Having worked with [Apache Solr and Lucene][solr] in the past, that was my hammer.  If you had worked with Solr two years ago, you know that fuzzy search isn't all that straight forward.  It's actually much easier today.  In any case, after some experimenting we came up with a configuration that was a marked improvement.  We added in some custom boosting and the search was ready for prime time.  Outside of some strange edge cases, we had a winner on our hands.  I stood up another Amazon EC2 instance, wrote a new ETL, and pushed it out.  

Here's the point where I fess up to my biggest mistake:  I wrote Garland with the idea that we'd only be supporting the LEIE dataset.  Well, Lou dropped the bomb on me.  "Matt, can we add Medicheck to our search, and can we add this System for Award Management dataset, too?"  My instant response was something along the lines of, "no."  But before he could reply again, I had already started refactoring our object models to be more general.  Generally speaking, Ruby is easy to refactor -- especially if you have tests which I did not.  In any case, after a few hours, I had a generic architecture in place.  If we could find the data, we could manually load it.  That meant pulling down the files, deploying them to the server, and running a rake import task.  It was a pain, but we agreed to only do it once per week. 

We did this for nearly a year until we decided it was time to optimize.  We had grown out by about 50 customers and the demand had grown for more frequent data updates.  While I love Amazon, I also wanted to move the app over to Heroku in order to take advantage of the easily scalable dynos.  At this point, it was time to go from MVP to an all-out product.  The original interface was ugly so I redesigned it using Bootstrap 2.  I migrated the data over to Postgres.  I also decided to switch from Solr to Postgres's [trigram matching][tg] for fuzzy search (a topic for a whole other post).  The best change I made though was moving away from DelayedJob and onto Sidekiq.  I was constantly babysitting DelayedJob, and one of our senior engineers strongly suggest Sidekiq.  He was right:  Sidekiq runs like a champ.  We were already using Redis behind the scenes, so introducing Sidekiq was friction-free.  I haven't had an issue with background jobs since.

Garland was in great shape.  The last thing I needed to do was automate the import process.  For all ten data sources we support, I think I said that each one was going to be impossible to automatically import for one reason or another.  Heroku's lack of a writable file system made the task even more challenging, and storing 50,000 records in memory is a bad idea for several reason.  Luckily, I was able to use Redis as a temporary buffer, write custom importers for each source, and voila - we had it all working.  Garland was now able to import all of our data sources automatically without human intervention.  

The icing on the cake for all of this was adding in "pretty" reports and an inline search (Google-esque search).  Moving over to Postgres gave us a nice performance bump, so adding the inline search was a no-brainer.  I should have said this earlier, but in order to generate a matching report, users first had to import all of their employees and vendors.  There was no way to just search for one term without the import.  Believe it or not, this isn't a popular use case, but it is nice to have.  

That's the story of Garland going from a minimum viable product to the rather powerful application that it is today.  What's most interesting to me is that, during this whole process, we never really thought we were doing MVP.  We were simply doing what our customers needed.  It goes without saying that not every product should be built in this manner, but it's been a real pleasure to see a product grow in a MVP-style approach.



[garland]: http://www.garlandlive.net/
[sam]: https://www.sam.gov/portal/public/SAM/
[nj]: http://www.state.nj.us/treasury/debarred/
[mvp]: http://en.wikipedia.org/wiki/Minimum_viable_product
[solr]: http://lucene.apache.org/solr/
[tg]: http://www.postgresql.org/docs/9.1/static/pgtrgm.html
