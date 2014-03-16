---
layout: post
title: "QueryOver Series - Part 1: Why QueryOver?"
date: 2014-03-12 22:50:58 -0400
comments: true
categories: 
---

QueryOver is a strongly-typed querying technology built on top of NHibernate's Criteria API. It was introduced in NHibernate 3.0. QueryOver is actually quite powerful and flexible, as I aim to demonstrate in this series of blog posts.

There is not much in the way of official documentation for NHibernate in general, and even less for QueryOver. The only article on NHForge that I can find is [here](http://nhforge.org/blogs/nhibernate/archive/2009/12/17/queryover-in-nh-3-0.aspx). While this is a good read for an introduction, it really doesn't do QueryOver justice. There's a lot of capability that's not demonstrated there.

### Overview of available querying technologies

If you're using NHibernate, you're probably aware that you have several options when you go to write a query:

* SQL (via `CreateSQLQuery`)
* HQL
* LINQ to NHibernate
* Criteria
* QueryOver

Now, in some cases you _must_ use SQL or HQL because of the type of query you're writing. For the vast majority of simple to intermediate queries though, LINQ to NHibernate, QueryOver, and Criteria all seem like viable options. 

### Why use QueryOver?

Let's assume that you're working on a large project and using magic strings everywhere to write queries makes you a bit nervous. People are changing property and class names every day, potentially breaking queries across your application. In this case, your better-looking options are LINQ to NHibernate and QueryOver.

If you've tried to use LINQ to NHibernate for anything remotely complex, you know you'll run into problems quickly. Currently you [can't](http://stackoverflow.com/q/15590021/497356) [even](http://stackoverflow.com/q/13624959/497356) [do a left join](https://nhibernate.jira.com/browse/NH-2379). What this means is if you have a left join in your query, or even if you think you'll need one one day, you can't currently use LINQ to NHibernate.

I'm not trying to pick on NHibernate's LINQ provider or minimize the amount of work contributors have done towards it; I'm just pointing out that it's incomplete and not a great option for querying right now. With this in mind, QueryOver is really your *only* viable option for writing queries with NHibernate. I'm sure the LINQ provider will only get better, but until then QueryOver is a good option.

