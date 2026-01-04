---
type: post
title: JESI, or how to fill the gap between frontends and backends
date: 2017-01-08
hero: "/json-edge-side-include/dan-burton-ukRioxowrhg-unsplash.jpg"
tags:
  - Web
  - Golang
  - Hypermedia
  - JSON
authors:
  - Yutaka Ichibangase
---

There's a huge gap between frontend engineers and backend engineers. People just don't realize it or ignore it as if it were a fate.

# Aggregation Problem: who aggregates data and when?

Let's begin with a story: Furiosa is a talented frontend developer and Max is an experienced backend developer. They're a team working on a movie database web app.

Their app has a feature to show movie details with actors appeared in it. It's backed by an API endpoint `/movies/:id`:

```
$ curl http://api.example.com/movies/2 | jq .
{
  "title": "Pulp Fiction",
  "year": 1994,
  "actors": [
    {
      "name": "John Travolta",
      "role": "Vincent Vega"
    },
    {
      "name": "Samuel L. Jackson",
      "role": "Jules Winnfield"
    }
  ]
}
```

Now they want a feature to show actor details with movies they appeared in.

# how to solve the aggregation problem
 approaches from the frontend
 multiple requests to the backend
 approaches from the backend
 specialized endpoints for specific frontends
 backends for frontends
approaches from the middle!?
 RPC
 GraphQL
 Hypermedia
 JESI
 
# ad hoc aggregation for the rescue
  - how frontend engineers work with JESI
  - how backend engineers work with JESI
