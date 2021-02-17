---
layout: post
title:  "Spotify Reccommender System Project Intro and Building a Custom Spotify API Library Using C++ and Pybind11"
date:   2021-02-16 18:05:55 +0300
image:  '/assets/img/03.jpg'
tags:   Blog
---
## Introduction

In this article I hope to introduce the motivation behind building a custom reccommender system for Spotify song reccommendations, as well as a custom API wrapper library for the Spotify API. 


## Background

Back in November of 2020, I was in the process of building a new playlist on Spotify. One of Spotify's many amazing features is its reccommendation engine - while the algorithm itself has never been revealed for obvious reasons, one can safely assume that this reccommendation engine uses a mix of "Collaborative Filtering" and song attributes to produce live reccommendations. In the world of reccommender systems, there are two widely used methods for reccommendations - Collaborative and Content-Based filtering. The difference between the two lies in the subset of data used for building reccommendations:

* Collaborative Filtering - Provides reccommendations based on the ratings and activities of networks of users, i.e. if all of your Spotify friends/followers listen to indie rock, there is a good chance you will be reccommended some of their favorite indie rock bands
* Content-Based Filtering - Provides reccommendations based on the attributes of objects you are reccommending, i.e. you will receive reccommendations for songs with similar bass, tempo, melody, and other various audio features to the songs you currently listen to

For the most part, you can find the results of Spotify's reccommendation engine in three locations in the Spotify app - your 'Browse' page, your 'Discover Weekly' playlist, and the end of any one of your playlists where Spotify reccommends similar songs to add. While building out my playlist, I realized that some of the reccommendations turned out to be a bit of a letdown - the selection of songs was not particularly large and some songs even matched those in my playlists (a remix of a song in the playlist). What immediately came to mind was trying to build a reccommendation engine that could possibly match or even beat the results of Spotify's own algorithm. While I certainly don't have the resources to compete with the top minds working at Spotify, reccommender systems was one of my favorite subjects from my time at UCSD. 

_For more information on Spotify's reccommender system, check out these two Medium articles: [1](https://towardsdatascience.com/how-spotify-recommends-your-new-favorite-artist-8c1850512af0), [2](https://ericboam.medium.com/i-decoded-the-spotify-recommendation-algorithm-heres-what-i-found-4b0f3654035b)_

## Project Planning 

In terms of machine learning models, there is no doubt that machine learning models require a significantly sized data set in order to improve the quality of reccommendations. If I hoped to build something past a simple collaborative filtering based model, I would have to find a way to get song attributes. Fortunately, Spotify provides an API endpoint for collecting Spotify data. While planning the data collection phase of the project, an interesting thought came to mind - instead of using one of the many prebuilt libraries for interacting with the Spotify API, why not build a custom wrapper on top of that API and make it BETTER. And to make things more interesting - why not build that wrapper using C++? 

While C++ is not necessary a language that is typically used to interact with an API, there is no doubt that using C++ could provide improvements in performance over other popular libraries. My language of choice for the reccommender system phase of the project would be Python, so to makes interacting with my custom library easier, I would have to expose my C++ library to Python. Two of the most popular options for linking Python and C++ are the Python Boost sublibrary and Pybind11. Since Pybind11 is significantly more lightweight than Boost, I decided to use it for this project.  
