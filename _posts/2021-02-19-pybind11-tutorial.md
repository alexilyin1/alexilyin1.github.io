---
layout: post
title:  "Building a C++ API Wrapper and Pybind11 Tutorial"
date:   2021-02-19 18:05:55 +0300
image:  '/assets/img/pybind11.png'
tags:   Blog
---
_As mentioned in my previous post, [this](https://www.youtube.com/watch?v=-eIkUnCLMFc) Youtube series was a massive help in getting started with Pybind11. My project structure is heavily based on the example from that series. The [Pybind11 documentation](https://pybind11.readthedocs.io/) is awesome and is another good place to start_

# Contents
- Pybind11 Tutorial
  * [Introduction](#introduction)
  * [Libraries Needed](#libraries-needed)
    * [Pybind11](#pybind11)
    * [Libcurl](#libcurl)
  * [Pybind11 Project Structure](#pybind11-project-structure)

## Introduction

The existence of libraries known as "API Wrappers" make interacting with API endpoints particularly easy. Instead of directly making HTTP API calls every time you want to pull data from an API, these wrapper libraries allow you to interact with APIs through easy-to-use functions. To illustrate this point, here is the code for making a HTTP request in C++ and code for making a HTTP request using a wrapper library:

```
// Directly making HTTP API requests

CURL *curl;
std::string res;

curl = curl_easy_init();
if(curl) {
    try {
        curl_easy_setopt(curl, CURLOPT_TCP_NODELAY, 0);
        curl_easy_setopt(curl, CURLOPT_URL, "https://api.spotify.com/v1/playlists/37i9dQZF1DX0XUsuxWHRQd");
        curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, this->WriteCallback);
        curl_easy_setopt(curl, CURLOPT_WRITEDATA, &res);
        
        struct curl_slist *authChunk = nullptr;            
        authChunk = curl_slist_append(authChunk, "Accept: application/json");
        authChunk = curl_slist_append(authChunk, "Content-Type: application/json");
        authChunk = curl_slist_append(authChunk, ("Authorization: Bearer " + this->TOKEN).c_str());

        curl_easy_setopt(curl, CURLOPT_HTTPHEADER, authChunk);

        curl_easy_perform(curl);
        curl_easy_cleanup(curl);
    }
    catch (const char* Exception) {
        std::cerr << Exception << std::endl;
    }
}
```

```
//Using a wrapper 

cppotify.get_playlists("37i9dQZF1DX0XUsuxWHRQd")
```

The two code snippets above perform the same task - retrieving playlist information for a Spotify playlist with ID '37i9dQZF1DX0XUsuxWHRQd'. The first snippet uses the cURL library in C++ to call the API, while the second snippet uses a wrapper function. Technically, both snippets are using the same code, however the second snippest is clearly simpler for extended use. 

This article will cover the technical details of my Spotify API wrapper library, and hopefully will serve as a tutorial for building a pybind11 library. I built this library in Linux, for this reason instructions in this tutorial will only be relevant for Linux systems. 

## Libraries Needed

![Libcurl](/assets/img/pybind11_tutorial/curl.png)

### Pybind11

As mentioned in my previous post, I hoped to build my reccommender system in Python. This meant that I would have to expose my newly built library to Python. I chose Pybind11 to do this, as between Pybind11 and Boost, Pybind11 is a lighter than Boost. 

There are a couple of options for installing Pybind11. Out of the options that I tried, using Pybind11 as a Git submodule proved to be the most straightforward approach. To accomplish this, I used the following CMake functions:

```
set(EXTERNALS "${PROJECT_SOURCE_DIR}/externals")

add_subdirectory(${EXTERNALS}/pybind11-2.6.1)
```
(Folder prefixes will make more sense as we continue with the tutorial)

After adding the library to a Git repo, we'll see that the submodule shows up like 

![Submodule](/assets/img/pybind11_tutorial/git_submodule.png) 

Adding Pybind11 to a C++ project is as simple as that. 

### Libcurl

_The next set of steps involve setting up libcurl in C++, they can be skipped if your project does not involve using libcurl_

As mentioned in my [previous](https://alexilyin.me/2021/02/16/C++-Python-Project/) post, C++ is not generally thought of as a language used to make HTPP requests. However, the existence of the [libcurl](https://curl.se/libcurl/) library makes this is a possibility. While there are existing libcurl wrappers such as [curlcpp](https://github.com/JosephP91/curlcpp), I decided to limit the amount of prebuilt libraries used for this project and went with the approach of directly using libcurl. 

To install libcurl, use the following command:

```
apt-get install libcurl4-openssl-dev
```

The command above will allow us to use the curl.h header file in C++, where the functions used in the code snippet above are stored. To make sure the curl library works, we have to include the lines 

```
pybind11_add_module (
    pybind11module 
)

target_link_libraries(
    pybind11module
    curl
)
```

in our CMakelists file. The first lines use the pybind11 "pybind11_add_module" CMake function to create a pybind11 module, while the next lines link the newly created pybind11module with the curl library we just installed. 

Once we have the curl library installed, we can begin setting up the Pybind11 project itself.

## Pybind11 Project Structure 
