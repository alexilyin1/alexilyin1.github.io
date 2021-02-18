## Introduction

In this article, I will discuss the motivation behind building a custom reccommender system for Spotify song reccommendations, as well as a custom API wrapper library for the Spotify API. 

## Background

In the world of machine learning algorithms, reccommender systems are one of the most common algorithms in deployment. Many of us see the results of reccommender systems in our daily internet use - whether it is on Netflix, Youtube, or Facebook, these advanced algorithms are working behind the scenes to make sure that we stay on these platforms as long as possible. In most of these cases, reccommender systems are personalized and trained to your activity and are ready to make reccommendations on the fly. In this way, you are able to constantly scroll through a particular apps "home page" and interact with seemingly endless content. With social media companies taking advantage of vast amounts of user data, they have almost perfected reccommender system. In this project, I hoped to challenge Spotify's reccommender system. 

Back in November of 2020, I was in the process of building a new playlist on Spotify. One of Spotify's many amazing features is its reccommendation engine - while the algorithm itself has never been revealed for obvious reasons, one can safely assume that this reccommendation engine uses a mix of "Collaborative Filtering" and reccommendations based on song attributes to produce live reccommendations. In the world of reccommender systems, there are two widely used methods for reccommendations - Collaborative and Content-Based filtering. In more detail:

* Collaborative Filtering - Provides reccommendations based on the ratings and activities of networks of users, i.e. if all of your Spotify friends/followers listen to indie rock, there is a good chance you will be reccommended some of their favorite indie rock bands
* Content-Based Filtering - Provides reccommendations based on the attributes of objects you are reccommending, i.e. you will receive reccommendations for songs with similar bass, tempo, melody, and other various audio features to the songs you currently listen to

For the most part, you can find the results of Spotify's reccommendation engine in three locations in the Spotify app - your 'Browse' page, your 'Discover Weekly' playlist, and the end of any one of your created playlists where Spotify reccommends similar songs to add. While building out my playlist, I realized that some of the reccommendations turned out to be a bit of a letdown - the selection of songs was not particularly large and some songs even matched those in my playlists (a remix of a song in the playlist or vice versa). What immediately came to mind was trying to build a reccommendation engine that could possibly match or even beat the results of Spotify's own algorithm. While I certainly don't have the resources to compete with the top minds working at Spotify, reccommender systems was one of my favorite subjects from my time at UCSD. 

_For more information on Spotify's reccommender system, check out these two Medium articles: [1](https://towardsdatascience.com/how-spotify-recommends-your-new-favorite-artist-8c1850512af0), [2](https://ericboam.medium.com/i-decoded-the-spotify-recommendation-algorithm-heres-what-i-found-4b0f3654035b)_

## Project Planning 

There is no doubt that reccommender systems require a significantly sized data set in order to improve the quality of reccommendations. If I hoped to build a competitive model, it would have to involve collecting a large amount of data, hopefuly not by hand. Fortunately, Spotify provides an API endpoint for collecting Spotify data. While planning the data collection phase of the project, an interesting thought came to mind - instead of using one of the many prebuilt libraries for interacting with the Spotify API, why not build a custom wrapper on top of that API and make it BETTER. And to make things more interesting - why not build that wrapper using C++? 

While C++ is not necessary a programming language that is used to interact with APIs, there is no doubt that using C++  could provide improvements in performance over other popular libraries. My language of choice for the reccommender system phase of the project would be Python, so to make interacting with my custom library easier, I would have to expose my C++ library to Python. As for linking Python and C++, I had two options: I could either by call the C++ library directly from Python, or I could create a Python package that calls the C++ library for me. The obvious downside with the first approach was that I would have to manually create a new C++ object in Python each time I wanted to use the C++ library. This seemed like an unneccessary step that would complicate the data collection process as a whole. Building a C++ library from scratch would add a significant amount of time to the project, but seemed like the logical approach in the long run. 

With the data collection medium decided, the project scope looked liked:

1. Build and test a custom Python/C++ linked library
2. Collect data that would be used to train a reccommender system
3. Use a data warehousing solution to store the data
4. Design and train an algorithm
5. "Deploy" the algorithm and run experiments (I don't have the resources to run a full fledged experiment, my study would have to be based on friends and family)
6. Publish official results

## Briefly on Technical Details of the Library 

Two of the most popular options for linking Python and C++ are the Boost Python sublibrary and Pybind11. Since Pybind11 is significantly more lightweight than Boost, I decided to use it for this project. The main challenge in building such a library is exposing C++ objects to Python. In the case of Pybind11, this exposure happens with the help of dynamic/shared library. In constrat to static libraries, dynamic libraries can be linked to any program at run-time. This happens by inserting the location of the functions in memory to the executable. In this case, if we give our Python package the location of the C++ executable/shared library, it can use this library without having to recompile and receive a new location each time we hope to use the library. Since I would be writing code in Linux, the shared library would be contained in a .so file (.dll on Windows). Pybind11 makes this extremely simple, especially since we can use CMakelists to greatly reduce the amount of typing necessary to compile all of our project files 

(I'll write about the technical details of building this library in a later article in the form of a "tutorial"). 

_I found that [this](https://www.youtube.com/watch?v=-eIkUnCLMFc) tutorial was extremely helpful in getting started with Pybind11, I will reference it heavily in my own tutorial_

## Details of the Reccommender System

A great help through both stages of this project was [Spotify's API Web Reference](https://developer.spotify.com/documentation/web-api/reference/) - both for seeing which API endpoints I would have to adjust for in my wrapper, as well as choosing which fields I would use in my reccommender system. A big hurdle in the design of a new algorithm was the fact that Spotify does not support an endpoint for finding a user's followers. One can only assume that this endpoint is not supported so projects can not mimic Spotify's social network, as it doesn't make sense that Spotify engineers don't have the capability to create it.

At first, the lack of an endpoint to get a user's followers seemed to write off the possibility of using collaborative filtering. However, I discovered a workaround that would allow me to create an arbitrary "social network" - playlists. By viewing a particular user's playlists, I could create a network amount the users which created the playlists they follow. Our network would look something like this:

* 1st Degree Connections - Users who created the playlists the current user is following
* 2nd Degree Connections - Similar logic to 1st Degree Connections, but for the User's who created those playlists

In a sense, this mimics a social network similar to the one on LinkedIn. The intuitive base of this social network design is that, if I follow a certain playlist.. clearly the interests of the user who created that playlist are reflected in the songs they added. From that perspective, following a Playlist is like a "2nd Degree Follow" between two users. 

Another addition to the design of the reccommender system would be song attributes. Fortunately, Spotify does support retrieving song attributes given a song ID, so no additional workaround such as possibly manually adding song attributes would be needed here.
