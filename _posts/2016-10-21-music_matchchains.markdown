---
layout: post
author: David M. Weigl
title:  "Music, match chains, and madness"
date:   2016-10-14 15:58:56 +0100
categories: Virtuoso SPARQL bug 
---

TL;DR: Virtuoso has a [bug](https://github.com/openlink/virtuoso-opensource/issues/590) relating to SPARQL property paths with zero-occurrence patterns when they are used within named graphs where the name is supplied as a bound variable.

This was a fun one to debug. 

So, SPARQL is a query language that lets you define patterns to be matched against graphs of data expressed as RDF. Property paths are a super useful part of SPARQL that allow you define graph traversals in a very concise way. They are simple, powerful, and reminiscent of regular expressions.

Imagine a graph structure that expresses "#matches" relationships between different entities. Perhaps each node being "#matched" is associated with complementary information about some shared entity in the world---say, a composer of musical works. 

Perhaps the nodes come from different datasets. One might be a library catalogue describing books on the composer; another, broadcast information from a radio programme presenting the composer's works; and another still, a corpus of symbolic music encodings of his or her compositions. 

A friendly musicologist with a penchant for semantic alignment has come along and kindly specified the appropriate <#matches> relationships for us. Now we can find out which radio episode features a recording of this encoded composition, and request high-resolution images of the score in the book in the library!

![Simple match chain graph]({{ site.url }}/assets/virtuoso-bug-graph-example.svg)

Which we can express as RDF (using the awesome Turtle syntax) as:

{% highlight turtle %}
<#a> <#matches> <#m> .
<#b> <#matches> <#m> . 
<#b> <#matches> <#n> . 
<#c> <#matches> <#n> . 
{% endhighlight %}

As you can see, <#a>, <#b>, and <#c> are all connected along a sequence of matches (<#m> and <#n>), forming a *match chain*. Thousands of such chains might exist in our triplestore database. We can make use of the interconnections to extract all of the entities associated with any given entity, using the following SPARQL query:

{% highlight sparql %}
SELECT ?uri { 
   GRAPH <#matchDecisions> { 
      <#a> (<#matches>/^<#matches>)* ?uri .
   }
}     
{% endhighlight %}

This query looks into the <#matchDecisions> named graph, and selects all entities (bound to the ``?uri'' variable) that are zero or more <#matches> hops away from the given entity <#a>. So the query will return <#a>, which is zero hops away from itself; <#b>, as <#a> <#matches> the same node (<#m>) as <#b> <#matches>; and <#c>, as <#a> <#matches> a node <#matched> by an entity (<#b>) that also <#matches> a node (<#n>) matched by <#c>. 

So far, so good. 

Now, what happens if many musicologists are providing us with matches? We could store them all in the same place (the <#matchDecisions> named graph in the SPARQL query above). But then, we have no way of telling apart the decisions made by one musicologist from another's. It turns out that such match decisions are far from straightforward, and musicologists will occasionally disagree with one another on whether or not two entities should form a match; this may be down to ambiguities in the data, or to scholarly disagreements, such as suspected misattributions of particular works. To solve this, we store each musicologist's decisions in their very own, individual named graph. So the accomplished musicology experts Bob and Jane each get their own place in the triplestore; e.g. <#BobsMatchDecisions> and <#JanesMatchDecisions>. 

We can decide which of these experts to trust in our query by binding their graph's name to a variable, and using that variable to specify the graph: 

{% highlight sparql %}
BIND(<#JanesMatchDecisions> as ?matchGraph).

SELECT ?uri { 
   GRAPH ?matchGraph { 
      <#a> (<#matches>/^<#matches>)* ?uri .
   }
}     
{% endhighlight %}

This query ought to work exactly like the previous one, returning <#a>, <#b>, and <#c> (assuming that they are indeed specified by Jane as matches, as in the turtle triples above). However, it turns out that [Virtuoso](http://github.com/openlink/virtuoso-opensource/), our triplestore of choice, treats zero-length edges in property paths strangely when the graph name is supplied by a bound variable. Rather than <#a>, <#b>, and <#c>, the query returns <#b>, <#c>, and <#c>. Coming to that conclusion took several hours of headscratching --- I'm very confused as to why supplying the graph name as a variable should result in this subtly different behaviour. The [bug has been reported](https://github.com/openlink/virtuoso-opensource/issues/590), and is currently being looked into by the developers. In the meantime I thought it'd be a fun thing to report as a first post to this blog --- perhaps I can save somebody else the confusion!
