---
layout: post
title:  "Spotify Reccommender System Project Intro and Building a Custom Spotify API Library Using C++ and Pybind11"
date:   2021-02-16 18:05:55 +0300
image:  '/assets/img/recc_system.png'
tags:   Blog
---
## Introduction

In this article, I will discuss the motivation behind building a custom recommender system for Spotify song recommendations, as well as a custom API wrapper library for the Spotify API. 

## Background

In the world of machine learning algorithms, recommender systems are one of the most common algorithms in deployment. Many of us see the results of recommender systems in our daily internet use - whether it is on Netflix, Youtube, or Facebook, these advanced algorithms are working behind the scenes to make sure that we stay on these platforms as long as possible. In most of these cases, recommender systems are personalized and trained to your activity and are ready to make recommendations on the fly. In this way, you are able to constantly scroll through a particular appss "home page" and interact with seemingly endless content. With social media companies taking advantage of vast amounts of user data, they have almost perfected their recommender systems. In this project, I hoped to challenge Spotify's recommender system. 

Back in November of 2020, I was in the process of building a new playlist on Spotify. One of Spotify's many amazing features is its recommendation engine - while the algorithm itself has never been revealed for obvious reasons, one can safely assume that this recommendation engine uses a mix of "Collaborative Filtering" and recommendations based on song attributes to produce live recommendations. In the world of recommender systems, there are two widely used methods for recommendations - Collaborative and Content-Based filtering. In more detail:

* Collaborative Filtering - Provides recommendations based on the ratings and activities of networks of users, i.e. if all of your Spotify friends/followers listen to indie rock, there is a good chance you will be recommended some of their favorite indie rock bands
* Content-Based Filtering - Provides recommendations based on the attributes of objects you are recommending, i.e. you will receive recommendations for songs with similar bass, tempo, melody, and other various audio features to the songs you currently listen to

For the most part, you can find the results of Spotify's recommendation engine in three locations in the Spotify app - your 'Browse' page, your 'Discover Weekly' playlist, and the end of any one of your created playlists where Spotify recommends similar songs to add. While building out my playlist, I realized that some recommendations turned out to be a bit of a letdown - the selection of songs was not particularly large and some songs even matched those in my playlist (a remix of a song in the playlist or vice versa). What immediately came to mind was trying to build a recommendation engine that could possibly match or even beat the results of Spotify's own algorithm. While I certainly don't have the resources to compete with the top minds working at Spotify, recommender systems was one of my favorite subjects from my time at UCSD. 

_For more information on Spotify's recommender system, check out these two Medium articles: [1](https://towardsdatascience.com/how-spotify-recommends-your-new-favorite-artist-8c1850512af0), [2](https://ericboam.medium.com/i-decoded-the-spotify-recommendation-algorithm-heres-what-i-found-4b0f3654035b)_

## Project Planning 

There is no doubt that recommender systems require a significantly sized data set in order to improve the quality of recommendations. If I hoped to build a competitive model, it would have to involve collecting a large amount of data, hopefully not by hand. Fortunately, Spotify provides an API endpoint for collecting Spotify data. While planning the data collection phase of the project, an interesting thought came to mind - instead of using one of the many prebuilt libraries for interacting with the Spotify API, why not build a custom wrapper on top of that API and make it BETTER. And to make things more interesting - why not build that wrapper using C++? 

While C++ is not necessary a programming language that is used to interact with APIs, there is no doubt that using C++  could provide improvements in performance over other popular libraries. My language of choice for the recommender system phase of the project would be Python, so to make interacting with my custom library easier, I would have to expose my C++ library to Python. As for linking Python and C++, I had two options: I could either call the C++ library directly from Python, or I could create a Python package that calls the C++ library for me. The obvious downside with the first approach is that I would have to manually create a new C++ object in Python each time I wanted to use the C++ library. This seemed like an unnecessary step that would complicate the data collection process as a whole. Building a C++ library from scratch would add a significant amount of time to the project, but seemed like the logical approach in the long run. 

With the data collection medium decided, the project scope looked liked:

1. Build and test a custom Python/C++ linked library
2. Collect data that would be used to train a recommender system
3. Use a data warehousing solution like AWS to efficiently store the data
4. Design and train an algorithm
5. "Deploy" the algorithm and run experiments (I don't have the resources to run a full fledged experiment, my study would have to be based on friends and family)
6. Publish official results on this blog

## Briefly on Technical Details of the Library 

Two of the most popular options for linking Python and C++ are the Boost Python sublibrary and Pybind11. Since Pybind11 is significantly more lightweight than Boost, I decided to use it for this project. The main challenge in building such a library is exposing C++ objects to Python. In the case of Pybind11, this exposure happens with the help of dynamic/shared library. In contrast to static libraries, dynamic libraries can be linked to any program at run-time. This happens by inserting the location of the functions in memory to the executable. In this case, if we give our Python package the location of the C++ executable/shared library, it can use this library without having to recompile and receive a new location each time we hope to call functions from the library. Since I would be writing code in Linux, the shared library would be contained in a .so file (.dll on Windows). Pybind11 makes this extremely simple, especially since we can use Makefiles or CMakelists, which make it easy to compile a large amount of files. 

(I'll write about the technical details of building this library in a later article in the form of a "tutorial"). 

_I found that [this](https://www.youtube.com/watch?v=-eIkUnCLMFc) tutorial was extremely helpful in getting started with Pybind11, I will reference it heavily in my own tutorial_

## Details of the Recommender System

A great help through both stages of this project was [Spotify's API Web Reference](https://developer.spotify.com/documentation/web-api/reference/) - both for seeing which API endpoints I would have to build into my wrapper library, as well as choosing which fields I would use in my recommender system. A big hurdle in the design of a new algorithm was the fact that Spotify does not support an endpoint for finding a user's followers. One can only assume that this endpoint is not supported so projects can not mimic Spotify's social network, as it doesn't make sense that Spotify engineers don't have the capability to create it.

At first, the lack of an endpoint to get a user's followers seemed to write off the possibility of using collaborative filtering approach. However, I discovered a workaround that would allow me to create an arbitrary "social network" - playlists. By viewing a particular user's followed playlists, I could create a network among the users which created the playlists followed by a particular user. Our network would look something like this:

* 1st Degree Connections - Users who created the playlists the current user is following
* 2nd Degree Connections - Similar logic to 1st Degree Connections, but for the User's who created those playlists

In a sense, this mimics a social network similar to the one on LinkedIn. The intuitive base of this social network design is that, if I follow a certain playlist, clearly the interests of the user who created that playlist are reflected in the songs they added. From that perspective, following a Playlist is like a "2nd Degree Follow" between two users. They may not be directly following eachother, but a user that follows a certain playlist is clearly interested in that person's musical tastes.

Another interesting API endpoint available through the Spotify API is the "Personalization API". This API allows you to see your own top artists and tracks over the existence of your Spotify account, or a specific time period. This API could be used to establish a base for reccommendations.

Another addition to the design of the recommender system would be song attributes. Fortunately, Spotify does support retrieving song attributes given a song ID, so no additional workaround such as possibly manually listening to thounsands of songs and adding song attributes would be needed here (just kidding I would never do this).

At this point, my plan for data collection looked something like:

1. Use the "Search" and "Browse" APIs to generate a list of albums/playlists/artists to begin branching out from
2. Use other endpoints to view albums/playlists/artists related to the original list from Step 1, and expand the original list
3. Once a large set of albums/playlists/artists is selected, expand from within the list i.e. get albums/artists from playlists
4. Extract all the songs from objects collected in the 3 steps above
5. Use the AWS Python connecter to store the data in a AWS Redshift instance

## Wrapping Up

As of 2/16, this project is still in progress. I've made enough progress on my library to make a separate post detailing how I linked Python and C++ in a Python package. That post should be coming soon. Thank you for reading and I'm excited to share more details soon. Check out my work so far [here](https://github.com/alexilyin1/CPPotify)
