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
  * [CMakeLists](#cmakelists)
  * [Testing](#testing)
  * [Conclusion](#conclusion)

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

The two code snippets above perform the same task - retrieving playlist information for a Spotify playlist with ID '37i9dQZF1DX0XUsuxWHRQd'. The first snippet uses the cURL library in C++ to call the API, while the second snippet uses a wrapper function. Technically, both snippets are using the same code, however the second snippet is clearly simpler for extended use. 

This article will cover the technical details of my Spotify API wrapper library, and hopefully will serve as a tutorial for building a pybind11 library. I built this library in Linux, for this reason instructions in this tutorial will only be relevant for Linux systems. 

## Libraries Needed

### Pybind11

As mentioned in my previous post, I hoped to build my recommender system in Python. This meant that I would have to expose my newly built library to Python. I chose Pybind11 to do this, as between Pybind11 and Boost, Pybind11 is a lighter than Boost. 

There are a couple of options for installing Pybind11. Out of the options that I tried, using Pybind11 as a Git submodule proved to be the most straightforward approach. To accomplish this, I used the following CMake functions:

```
set(EXTERNALS "${PROJECT_SOURCE_DIR}/externals")

add_subdirectory(${EXTERNALS}/pybind11-2.6.1)
```
(Folder prefixes will make more sense as we continue with the tutorial)

After adding the library to a Git repo, we'll see that the submodule shows up like 

![Submodule](/assets/img/pybind11_tutorial/git_submodule.png) 

Adding Pybind11 to a C++ project is as simple as that. 

![Libcurl](/assets/img/pybind11_tutorial/curl.png)

### Libcurl

_The next set of steps involve setting up libcurl in C++, they can be skipped if your project does not involve using libcurl_

As mentioned in my [previous](https://alexilyin.me/2021/02/16/C++-Python-Project/) post, C++ is not generally thought of as a language used to make HTTP requests. However, the existence of the [libcurl](https://curl.se/libcurl/) library makes this is a possibility. While there are existing libcurl wrappers such as [curlcpp](https://github.com/JosephP91/curlcpp), I decided to limit the amount of prebuilt libraries used for this project and went with the approach of directly using libcurl. 

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

[Git Repo](github.com/alexilyin1/CPPotify)

As mentioned above, the structure of this project follows the structure presented in the video tutorial I linked. The overall structure looks like:

```
+-- /externals
|   +-- /pybind11
|
+-- /source
|   +-- /app
|   +-- /module
|   +-- /py
|
|   CMakeLists.txt
```

#### Breakdown of folders

* externals - This folder will contain any submodules we add to our project, in this case Pybind11
* source - This folder contains the source code for the project:
    * app - Contains the main.cpp file
    * module - Contains the code for the C++ module we will be linking to Python
    * py - Contains the Python code that calls the C++ wrapper
* cmake-build-debug - Files created when we compile the files 

With this structure in mind, we can begin populating the necessary files. We'll add our 'main.cpp' into the 'app' folder. This file isn't necessarily important to building our library, but can be used for testing it in C++. 

```
+-- /source
|   +-- /app
|       +--main.cpp
|
|   +-- /module
|       +--CPPotify.cpp
|       +--CPPotify.h
|       +--authControl.cpp
|       +--authControl.h
|
|   +-- /py
|   
```
Into the module folder, we'll add two sets of header/source files - 'CPPotify.cpp' is the important file in this case - it actually uses Pybind11 to create a 'module'. Here is the relevant code:

```
#include <pybind11/pybind11.h>

namespace py = pybind11;
```

In this code snippet, we include the pybind11.h header file and declare a 'py' namespace that will hold the necessary pybind11 functions. 

After writing the class and functions we want to expose to Python, we need to formally declare them as a Pybind11 module:

```
PYBIND11_MODULE(pybind11module, cpp) {
    cpp.doc() = "CPPotify Module - Python Spotify API using C++";
    py::class_<CPPotify>(cpp, "CPPotify")
            .def(py::init<std::string, std::string>())
            ... _Rest of your functions_ ...
            .def("getToken", &CPPotify::getToken);
};
```

Here is where naming convention comes into play - we want to make remember the name we pass into the first argument of the PYBIND11_MODULE function call, as it must match the name we use in our CMakeLists.txt file.

After you've added the class functions you want your users to be able to call from Python, we can begin populating the CMakeLists.txt file, which will tie our project together. 

## CMakeLists

Before starting to write any actual CMake functions, we can set some filepath variables to reduce repetitive filepath copy/pasting. We'll also add the subdirectory containing Pybind11 and 'find' another external library used:

```
project("pybind11module_CPPotify")

set(APP_SOURCE "${PROJECT_SOURCE_DIR}/source/app")
set(MODULE_SOURCE "${PROJECT_SOURCE_DIR}/source/module")
set(EXTERNALS "${PROJECT_SOURCE_DIR}/externals")

add_subdirectory(${EXTERNALS}/pybind11-2.6.1)
find_package(nlohmann_json 3.2.0 REQUIRED)
```

(${PROJECT_SOURCE_DIR} is created automatically once you create a project, no need to define it)

We can start with creating a target library for our Pybind11 module. Instead of using the 'add_library' function, we'll use the 'pybind11_add_module' function which is quite literally a wrapper for the former:

```
pybind11_add_module (
    pybind11module 
    ${MODULE_SOURCE}/CPPotify.cpp
    ${MODULE_SOURCE}/authControl.cpp
)
```

WARNING - The first argument to that function must match the name we used to create the Pybind11 module in our C++ function.

Now that we have a target library, we can use the 'target_link_libraries' function to link the external libraries we are using, in this case libcurl

```
target_link_libraries(
    pybind11module
    PRIVATE curl
    PRIVATE nlohmann_json::nlohmann_json
)
```

WARNING - If you don't include PRIVATE before the libraries you are using, you will get the following error

```
[cmake] CMake Error at CMakeLists.txt:21 (target_link_libraries):
[cmake]   The keyword signature for target_link_libraries has already been used with
[cmake]   the target "pybind11module".  All uses of target_link_libraries with a
[cmake]   target must be either all-keyword or all-plain.
[cmake] 
[cmake]   The uses of the keyword signature are here:
[cmake] 
[cmake]    * externals/pybind11-2.6.1/tools/pybind11Tools.cmake:153 (target_link_libraries)
[cmake]    * externals/pybind11-2.6.1/tools/pybind11Tools.cmake:180 (target_link_libraries)
```

That keyword before the library tells us what kind of dependency our shared library will have with that external library - if the dependency is PRIVATE, that means we are including headers in our source files but NOT our header files - this is the case for this shared library. I found [this SO post](https://stackoverflow.com/questions/26037954/cmake-target-link-libraries-interface-dependencies) helpful in understand this. 

At this point, we have successfully complete the necessary steps for creating a Pybind11 module, the full CMakeLists looks like:

```
cmake_minimum_required(VERSION 3.16)
project("pybind11module_CPPotify")
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_PREFIX_PATH /usr/include/)

set(APP_SOURCE "${PROJECT_SOURCE_DIR}/source/app")
set(MODULE_SOURCE "${PROJECT_SOURCE_DIR}/source/module")
set(EXTERNALS "${PROJECT_SOURCE_DIR}/externals")

add_subdirectory(${EXTERNALS}/pybind11-2.6.1)
find_package(nlohmann_json 3.2.0 REQUIRED)

pybind11_add_module (
    pybind11module 
    ${MODULE_SOURCE}/CPPotify.cpp
    ${MODULE_SOURCE}/CPPotify.h
    ${MODULE_SOURCE}/authControl.cpp
    ${MODULE_SOURCE}/authControl.h
)

target_link_libraries(
    pybind11module
    PRIVATE curl
    PRIVATE nlohmann_json::nlohmann_json
)

target_include_directories (
    pybind11module 
    PRIVATE ${MODULE_SOURCE}
)

add_executable (
    pybind11app
    ${APP_SOURCE}/app.cpp
    ${MODULE_SOURCE}/CPPotify.cpp
    ${MODULE_SOURCE}/CPPotify.h
    ${MODULE_SOURCE}/authControl.cpp
    ${MODULE_SOURCE}/authControl.h
)

target_include_directories (
    pybind11app 
    PRIVATE ${APP_SOURCE}
    PRIVATE ${MODULE_SOURCE}
)

target_link_libraries(
    pybind11app
    pybind11::embed
    curl
    nlohmann_json::nlohmann_json
)
```

## Testing

The easiest was to import your module in Python is to navigate to the new 'build' folder that will be created in your main project folder. This folder will contain the dynamic library .so file mentioned in my previous post. We can either cd into this folder or launch a terminal from this window and launch Python from the terminal

``` 
$cd ~/CPPotify/build && python3.8
Python 3.8.5 (default, Jul 28 2020, 12:59:40) 
[GCC 9.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import pybind11module
>>>
```

_Its important to specify which Python version you are using to import the module - if your default Python version is 3 but you're using 3.8 for Pybind11, Python 3 will not find the module!_

If you are able to import your module using the name you selected when creating it, everything has worked as intended! Great job!

## Conclusion

I hope this tutorial has served as a good introduction into building a shared library in C++. As mentioned, my full source code can be found [here](github.com/alexilyin1). This is the first programming related tutorial I have written, please leave feedback and critiques where you feel the tutorial was not effective. Thanks!

