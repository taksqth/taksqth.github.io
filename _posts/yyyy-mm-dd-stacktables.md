---
layout: post
author: taksqth
title:
---

## The problem, the context and a bad approach

This post is about the hard problem of joining data from different sources and,
in the worst cases, with different grain (the level of detail at which the data
is stored). Let's say you need to prepare data for a dashboard that combines
sales data from your department and advertising costs from different vendors
such as Google and Facebook. Depending on your background you may need more
information or you can already think of some ways to tackle this problem. What I
will describe below is the way I usually do it and why I think one of the more
common strategies, which is a simple JOIN statement, is not something I would
recommend. But first, a bit of much needed context.

![Here's the example that will follow us throughout the article. Pleased to meet you!]()

At work, I get to interact with the marketing department of companies at
different points in their journey to become more data-driven. Most of the time,
the areas I interact with have little to no data infrastructure and, even if the
company itself has a data team, they have a hard time connecting to the needs of
the teams I work with. There's the need for information to be available fast and
it to be presented in a way that supports rapidly changing processes.

The technique I will share in this post is one that I've acquired with practice
building datasets that feed into dashboards and reports for those teams. I think
it can be of help to anyone, no matter their level of experience with SQL.
However the people that will benefit the most from this post are those that work
in scenarios close to what I've described above: Projects that need to deliver
value quickly in an environment not already well developed enough that you can
simply build on top of a solid foundation, such as a Data Warehouse that already
has all the data neatly organized into fact tables.

So let's get back to my first example. If you know a bit about how digital
advertising works, for ppc (pay per click) models it can be tempting to say
"It's easy! I can get the information from both sources at click level and they
both should have a click id field by which I can join the data". Even if you
knew nothing about this specific problem, maybe this approach sounds plausible.
Well, as I've previously stated this is not a strategy that I would recommend.

The thing is, JOINs are hard to deal with. If you already know the struggle by
heart, I'm here to tell you that you don't need JOINs to solve the problem and I
find it better to avoid them. For those that don't see any issues with JOINs,
one of the problems is that they rely a lot on the underlying data and the key
(simple or composite) behaving well. Maybe in a more traditional transaction
based database this is to be expected, but this is rarely the case in the world
of analytics.

Back to our example, how can things go wrong? The thing is, the click id is
usually tracked via a query parameter in the URL that takes the user from the ad
to the website where they will eventually make a purchase. It can happen that
some users share this URL with their friends or family members and suddenly that
id doesn't belong only to the click made by that user. When we JOIN this data
the end result is that the cost row associated with that click gets matched with
the purchase made by every user that used that same shared url.[^1]

The outcome would be at best a wrong metric that someone notices, which in turn
may lead to that team losing trust in the data, and at worst one that no one
does, which leads to wrong decisions that impact business.

Another way to look at this problem is that we're effectively altering what each
row of the table represents. In some cases this is okay, and if we absolutely
trust the data that we have to behave well when joined, we can get an enriched
vision of everything together. What we really were aiming to do with our example
was to build a table where each row would be a click with its associated cost
(from advertising) and return (from sales).  Unfortunately even in those cases
where it does makes sense we can get gibberish due to issues like in the
example.

![Oh no, now our cost is all wrong because of duplicated matches]()

There are cases where the problem is worse. The cpc deal is only one type of
deal when it comes to buying ads. In another example, a company may pay a fixed
price to have ads running for entire day in a news website for their campaign.
Suddenly it doesn't make sense to look at click level anymore. In order to match
the different grain from the sales table to the campaign level cost you are
required to either aggregate it to be the same grain or instead to broadcast the
cost data for each sale. The better option is the first, but it can be hard to
maintain different tables at different grains for each case.

Okay, so if JOINs are no good, what should we do? Actually, before I present my
alternative, let me be clear that I do believe there is a time and place for the
JOIN statement, so I'll get back to it later in this article. Now, without
further ado.

## Just stack everything with UNION ALL

If the problem with JOINs is that we have to find a way to give new meaning to
each row of the resulting table without things going wrong, then I propose that
we avoid doing exactly this. By stacking each table on top of each other we
don't need to change what each row represents, preserving their original
meaning. The resulting table will probably have mixed grain, but that's
fine.[^2] 

How do we do this? After all, for our UNION ALL clause to work, the different
tables need matching schemas and, in our example, the sales table and the costs
table must both have at least one column that the other lacks. Namely, the main
metrics we want from both, sales and costs. To solve this is actually pretty
simple, we just need to create the missing columns in each table at query time
with a fixed value that makes sense. In this case, 0 will do fine since it
changes nothing under addition, the operation we'll use to aggregate the values
later.

![I find it better to use meaningful names for string placeholders]()

But how does that accomplishes our goal of joining the data exactly? The rows
never mix, so how are we supposed to know how much we invested in a campaign
together with how many sales we got from it? If we aggregate the data grouping
by the campaign column that's exactly what we get.

![Now the numbers make sense]()

Notice that we still require the attributes by which we associate the data to
be in both sources (in the example, the campaign column). This is not a demerit
of this method, as doing a JOIN requires something like this as well. However
with this stacking technique we don't run into problems when the attributes from
different tables don't match in detail.

[^1] In most cases the first key people come up with when trying to JOIN
the data isn't even the click id, but the url parameters from the analytics
vendor of choice.  Those are not created automatically by the advertising vendor
server, but instead configured either manually or with rules by the agency that
sets up the advertising campaign, to match certain attributes about the ad in
the platform (vendor, strategy, format, audience, etc.). Those parameters are
great for reporting and we can leverage them with the UNION ALL technique, but
are even more disastrous than click ids for the JOIN approach.

[^2] You may have heard that this is a bad practice. I would agree if
our objective was to build a solid foundation and leave the work of joining the
data to the end user with the help of a solid BI tool. But as I've said before,
this is not always the context we're in, and from my experience this is an
approach is pretty general and should always deliver.
