---
layout: post
title:  "Spotify Reccommender System Project Intro and Building a Custom Spotify API Library Using C++ and Pybind11"
date:   2021-02-16 18:05:55 +0300
image:  '/assets/img/recc_system.jpg'
tags:   Blog
---


_For more information on Spotify's reccommender system, check out these two Medium articles: [1](https://towardsdatascience.com/how-spotify-recommends-your-new-favorite-artist-8c1850512af0), [2](https://ericboam.medium.com/i-decoded-the-spotify-recommendation-algorithm-heres-what-i-found-4b0f3654035b)_

## Project Planning 

In terms of machine learning models, there is no doubt that machine learning models require a significantly sized data set in order to improve the quality of reccommendations. If I hoped to build something past a simple collaborative filtering based model, I would have to find a way to get song attributes. Fortunately, Spotify provides an API endpoint for collecting Spotify data. While planning the data collection phase of the project, an interesting thought came to mind - instead of using one of the many prebuilt libraries for interacting with the Spotify API, why not build a custom wrapper on top of that API and make it BETTER. And to make things more interesting - why not build that wrapper using C++? 

While C++ is not necessary a programming language that is used to interact with APIs, there is no doubt that using C++ could provide improvements in performance over other popular libraries. My language of choice for the reccommender system phase of the project would be Python, so to makes interacting with my custom library easier, I would have to expose my C++ library to Python. Two of the most popular options for linking Python and C++ are the Python Boost sublibrary and Pybind11. Since Pybind11 is significantly more lightweight than Boost, I decided to use it for this project. 

## Creating a Custom Spotify API Wrapper Library

While Pybind11 makes connecting Python and C++ relatively easy, there are a couple of design caveats that make building such a library a challenge. The main hurdle in connecting the <- in progress
