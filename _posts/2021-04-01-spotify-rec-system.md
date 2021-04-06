---
layout: post
title:  "Using my Spotify API Library to Build a Spotify Recommender System"
date:   2021-04-04 18:05:55 +0300
image:  '/assets/img/spotify_rec_system/spotify-icon.jpg'
tags:   Blog
---

# Contents

- Spotify Recommender System

  * [Introduction](#introduction)
  
  * [Creating a Dataset of Spotify Songs](#creating-a-dataset-of-spotify-songs)
  
  * [Defining a Custom Recommender Algorithm](#defining-a-custom-recommender-algorithm)
  
  * [Writing the Algorithm](#writing-the-algorithm)
  
  * [Testing Some Recommendations](#testing-some-recommendations)
  
  * [Conclusion and Possible Improvements](#conclusion-and-possible-improvements)


## Introduction

In this post I'll be wrapping up my "series" on my Spotify recommender system and Python/C++ shared library. To conclude the project, I'm going to talk through the thinking behind the recommender system that I've built, as well as the results. While the goal of this phase of the project was to create a model that could be deployed and used in real time, I wasn't able to achieve this result with the dataset that I had retrieved using the API. However, I can say that I am happy with the results of the model themselves. More on that later.

## Creating a Dataset of Spotify Songs 

While datasets of Spotify tracks and audio features exist on sites such as [Kaggle]([https://www.kaggle.com/yamaerenay/spotify-dataset-19212020-160k-tracks](https://www.kaggle.com/yamaerenay/spotify-dataset-19212020-160k-tracks)), my motivation for building a Spotify API wrapper was to 'mine' Spotify data myself. In order to make the wrapper development process pay off, I aimed to build a dataset larger than the one available on Kaggle. 

To pull this off, I came up with a multi-phased approach to build out a set of tracks and their audio features. This would be a time consuming process, and in order to ensure that data would not be lost in case of a dropped internet connection (this happened. A LOT) or any other issues, I created a script to periodically create data checkpoints in S3. 

The approach to gathering the data looked like this:

* Use the following options from the ['Browse']([https://developer.spotify.com/documentation/web-api/reference/#category-browse](https://developer.spotify.com/documentation/web-api/reference/#category-browse)) API endpoint
	* All playlists from 'Categories'
	* All artists and albums from 'New Releases'
	* For dates ranging from 2013 - 2021, get all playlists from 'Featured Playlists'
* For the playlists collected above, iterate through each playlist's tracks and gather all albums, artists and tracks
* For all artists collected above, iterate through each artist and find all ['Related Artists']([https://developer.spotify.com/documentation/web-api/reference/#endpoint-get-an-artists-related-artists](https://developer.spotify.com/documentation/web-api/reference/#endpoint-get-an-artists-related-artists))
* After collecting artists and their 'related' artists, grab all albums created by those artists
* Grab all tracks from the albums that have been collected
* For each track, find its ['Audio Features']([https://developer.spotify.com/documentation/web-api/reference/#endpoint-get-audio-features](https://developer.spotify.com/documentation/web-api/reference/#endpoint-get-audio-features)) (this will serve as the basis for the content based filtering part of our recommender system)

After waiting for a couple of days (this could have gone on longer but I was tired of waiting), I built up a dataset of ~200k tracks, just beating out the dataset found on Kaggle. 

## Defining a Custom Recommender Algorithm 

[TLDR here](#algorithm-definition-tldr)

As mentioned in [my previous post]([https://alexilyin.me/2021/02/16/C++-Python-Project/](https://alexilyin.me/2021/02/16/C++-Python-Project/)), the two most widely accepted forms of recommender systems are

* Collaborative Filtering
* Content-Based Filtering

This section will discuss the 'Collaborative' filtering approach (with modifications) that I took. 

In order to attempt to improve on Spotify's current recommendations, I decided to build out an algorithm that would be a mix of Collaborative and Contest-Based filtering.

![Spotify Follow Friends](/assets/img/follow-friends-on-spotify.jpg)

While Spotify does provide a feature to follow your friends, the Spotify _does not_ provide an API endpoint for accessing your followers. To mimic a network of friends and followers, I used the [Playlists]([https://developer.spotify.com/documentation/web-api/reference/#endpoint-get-a-list-of-current-users-playlists](https://developer.spotify.com/documentation/web-api/reference/#endpoint-get-a-list-of-current-users-playlists)) endpoint to get all of the current user's (my) playlists. From this list of playlists, I would keep only playlists that were not created by the current user, and grab the tracks from the remaining playlists. In this way, I would have a set of tracks that users that I follow listen to. This handles the 'collaborative' filtering aspect of my algorithm.

The next step is finding the audio features of each track in the remaining set, and comparing it with the audio features of the track the that user is currently listening to. In order to reduce compilation times and make sure that comparisons are only happening against relevant tracks, the set of tracks that remain is filtered for songs that have genres related to the genres of the current track. 

![Jaccard Distance Formula](/assets/img/spotify_rec_system/jacc_sim.png)

Initially, I thought that I would be able to complete a simple comparison of song genres by directly comparing them. However, upon examination of some Spotify genres, I realized that this would not work. It turns out that Spotify genres can be extremely specific:
```
[portland hip hop]
```

or can be specific with multiple values:

```
[hip hop, pop rap, rap, underground hip hop]
```

To create a workaround, I decided to incorporate the famous set distance metric **Jaccard Distance**.  However, with such specific values, applying Jaccard Distance directly to the genre sets would not provide accurate results. For example, for these two sets of genres:

```
[east coast hip hop, hip hop, pop rap, rap]
[chicago rap, conscious hip hop, hip hop]
```

The Jaccard Distance would be 0.16666666666666666. It is clear that these tracks are from similar genres, however, the computed Jaccard Distance tells us that these tracks are probably not from similar genres. An additional workaround that I implemented was taking the set union of each genre with sets of genres. For example:

```
[east coast hip hop, hip hop, pop rap, rap]
```
Would become:
```
{pop, coast, rap, hip, east, hop}
```

After taking the union of set of genres above, the new Jaccard Distance would be 0.5. This is closer to the answer we would expect. 

Back to our new dataset of tracks from our followed playlists: once we apply Jaccard Distance to the current track's genres and the rest of the tracks, we can filter out tracks that are not similar. Since the goal is to reduce compilation times while also keeping only tracks that are similar, I had to choose a similarity threshold for filtering the dataset. After trial and error, I decided to use the threshold 0.4 - I found that any value below 0.4 returned tracks/genres that were not as similar as needed. Additionally, when filtering for distance less than 0.4, the dataset reached a length at which compilation times would not be optimal (at this point I was still aiming to create a recommender algorithm that could be run in real time). 

After obtaining the final set of tracks, the next step would be to obtain the audio features for each song in the remaining set. This is where the dataset I collected in the previous set would come into play - if a song is already in our dataset, not additional computation is needed, as we can simply take the audio features we previously obtained. However, if the track is not in our ~200k track dataset, we will have to take the time to call the 'Audio Features' Spotify API endpoint to obtain audio features for that track.

#### Algorithm Definition TLDR

The set of steps mentioned above can be summarized as:

1. Start with the song the user is currently listening to
2. For this particular user, obtain a complete list of their playlists - playlists they created and playlists they follow
3. For the set of playlists that they follow, create a set of tracks contained in those playlists
4. For each track in the playlist, take the Jaccard Distance between the genres of the current track, and each track's genres in the set above
5. Filter the above set to tracks having Jaccard Distance >= 0.4
6. Find the audio features of the remaining tracks

![Cosine Similarity Formula](/assets/img/spotify_rec_system/cosin_sim.png)

Now that we have a 'final' set of similar tracks, we are left with finding a way to obtain the similarity of each track to the track the user is currently listening to. Among popular similarity metrics for recommender systems, Cosine Similarity is one that stood out - this metric will return the cosine of the angles between two vectors, both of which have been normalized to have lengths of 1. Using this metric, two parallel lines will have Cosine Similarity of 0. However, as the distance between two lines decreases, so does their cosine value/Cosine Similarity. 

Here is an example of a set of audio features for a particular track: 
```
""0.468","0.462","7","-17.189","1","0.0262","0.0199","4.48E-6","0.584","0.588","95.99","318000","4"
```

In such a case, we can assume that Jaccard Distance or other similarity metrics would not provide us an optimal measure of similarity.

## Writing the Algorithm

This section will cover the 'Content-based' filtering section of my algorithm. I'll be walking through the Python code chunks I wrote (some parts of the algorithm were covered in the previous section). 

Here are the three functions used to produce a set of recommended songs ([Github link]([https://github.com/alexilyin1/SpotifyRecSystem/tree/master/rec_system](https://github.com/alexilyin1/SpotifyRecSystem/tree/master/rec_system))).

```python
# List Union Function - For a list of lists, return the union of all the lists 
def list_union(set_list: list)->set:
	start = set(set_list[0])
	for others in set_list[1:]:
		start = start.union(others)
	return start


def get_song_artist_df(followed_playlists):
	# Take the playlists that a user follows. For each playlist in that list, create a dictionary having keys set to each unique track ID, with values set to
	# the artists that are featured on that track.

	followed_tracks_artists = {}

	for playlist_id in followed_playlists:

		try:
			res = cpp.get_playlists(False, '', playlist_id, 'tracks')
			for track in res['items']:
				if track['track']['id'] not  in followed_tracks_artists.keys():
					followed_tracks_artists[track['track']['id']] = [art['id'] for art in track['track']['artists']]
				else:
					for artist in track['track']['artists']:
						followed_tracks_artists[track['track']['id']].append(artist['id'])
						
		# Skip tracks that produce an error 
		except  Exception  as e:
			print(e)
			pass

	# Create a DataFrame containing with each row containing a unique artist-track combinations

	followed_tracks_df = pd.DataFrame(columns=['track', 'artist'])

	  

	for track in followed_tracks_artists.keys():
		for row in followed_tracks_artists[track]:
			followed_tracks_df = followed_tracks_df.append({'track': track, 'artist': row}, ignore_index=True)

	# Now that we have a DataFrame of unique artist-track combinations, we want to find the genres for each unique artist. Spotify does not release track genre information  
	# through its API, for this reason we have to find the genre of the artist. We will use this genre information to filter the final DataFrame to match genres with the song/artist
	# that is currently being listened to

	 
	artist_genre = {}
	for artist_id in followed_tracks_df['artist'].unique():
		if artist_id not in artist_genre.keys():
			artist_genre[artist_id] = cpp.get_artists(artist_id)['genres'] if cpp.get_artists(artist_id)['genres'] != [] and  'genres'  in cpp.get_artists(artist_id).keys() else  None

	# For the newly created artist_genre dictionary above, match each artist ID in our previously created DataFrame with its genre in the dictionary. Use this to create a new 'genre'
	# column in the DataFrame. Filter out any artists that do not have genre information

	followed_tracks_df['genres'] = [artist_genre[id] for  id  in followed_tracks_df['artist']]
	followed_tracks_df = followed_tracks_df[followed_tracks_df['genres'].notna()]

	# For the newly created 'genre' column, split the genres into words and create a union of those words for each artist. The reason we do this is that Spotify contains a lot of unique
	# genres. Since we have a rather small dataset, we want to split more rare genres such as 'german hip hop' into a set ('german', 'hip', 'hop') so that this song will match with other
	# german or hip-hop tracks

	followed_tracks_df['genre_words'] = [list_union([word for word in [genre_string.split(' ') for genre_string in genre_list]]) for genre_list in followed_tracks_df['genres'].values]

	return followed_tracks_df


def current_song_info(song_id)->tuple:
	# For the current song, get the song's artists and the genres for those artists
	
	track_info = cpp.get_tracks(song_id)
	current_artists = [artist['id'] for artist in track_info['artists']]
	
	current_genres = {}
	for artist in current_artists:
		try:
			current_genres[artist] = cpp.get_artists(artist)['genres']
		except  Exception  as e:
			current_genres[artist] =  None
	return current_genres

def  get_audio_features(tracks, existing_data):
	# Create a dictionary of tracks and their audio features. If the song exists in the current dataset, retrieve that information. If it doesn't, retrieve the information from the
	# Spotify API

	audio_features = {}
	for id in tracks:
		if id in existing_data['track_id'].values:
			audio_features[id] = existing_data[existing_data['track_id']==id].iloc[:, np.r_[5:16, 21:23]].values.tolist()
	else:
		try:
			audio_features[id] =  list({k:v for k,v in cpp.get_tracks(id, 'audio-features').items() if k in existing_data.columns[np.r_[5:16, 21:23]].values}.values())
		except:
			pass

	return audio_features


def song_rec(user_id, get_playlists_results, current_song, stored_songs):
	# Given a user_id, find all playlists that the user follows. Using this logic, we can create a set of tracks listened to by users that created these followed playlists.
	# Since the Spotify API does not let us find followed users, this is our best bet for collaborative filtering

	followed_playlists = [playlist['id'] for playlist in json.loads(get_playlists_results)['items'] if playlist['owner']['id'] != user_id]

	# Call the two functions above to get information for tracks contained in 'followed playlists', as well as track information for the current song	  

	followed_tracks_df = get_song_artist_df(followed_playlists)

	current_genres = current_song_info(current_song)

	# For the genres of the current song/artist(s), create a union of genre words

	current_genres_words = list_union([word for word in [dict_values.split(' ') for all_genres in current_genres.values() for dict_values in all_genres]])

	# Find the set Jaccard distance between the genres of the current song/artist(s) and all the songs in our followed tracks dataset. Since Spotify has unique genre naming conventions,
	# it will be harder to find exact matches for sets of genres

	followed_tracks_df['jac_dist'] = [round(jaccard_similarity(np.array(current_genres_words), np.array(followed_genre)), 2) for followed_genre in followed_tracks_df['genre_words']]

	# Keep songs which genre sets have Jaccard distance greater than or equal to 0.4

	followed_tracks_df = followed_tracks_df[followed_tracks_df['jac_dist'] >=  0.4]

	# Get the audio features for the songs in the remaining dataset

	followed_tracks_audio_features = get_audio_features(followed_tracks_df['track'].unique(), stored_songs)

	# Set the audio_features retrieved above as a new column in the dataframe

	followed_tracks_df['audio_features'] = [followed_tracks_audio_features[track] if track in followed_tracks_audio_features else  None  for track in followed_tracks_df['track']]

	followed_tracks_df = followed_tracks_df[followed_tracks_df['audio_features'].notna()]

	# Get the audio_features of the current song. We will be calculating the cosine similarity of the audio features of the current song to each song in our dataframe. Keep only the audio feature values from the total set of returned columns

	af_list_values =  list({k:v for k,v in cpp.get_tracks(current_song, 'audio-features').items() if k in stored_songs.columns[np.r_[5:16, 21:23]].values}.values())

	# Now that we have the audio features of the current song and the songs with closely matching genres, we can calculate cosine similarity. Sort the
	# DataFrame by cosine similarity and Jaccard distance. Then return the DataFrame

	followed_tracks_df['cosine_sim'] = [cosine_similarity(np.array(af_list_values), stored_songs.iloc[x, np.r_[5:16, 21:23]].values) for x in  range(len(followed_tracks_df))]
	followed_tracks_df = followed_tracks_df.sort_values(['jac_dist', 'cosine_sim'], ascending=False).drop_duplicates('track').drop('genres', axis=1)

	return followed_tracks_df
```

Now that we've defined our functions, lets test some output to see if it matches our expected results.

## Testing Some Recommendations

To test the results of the algorithm, I'll pick two random tracks from my ~200k track dataset

### Song 1  - Madonna by Drake 

Using this first song, the first 20 recommendations sorted by Jaccard Distance and Cosine Similarity looked like:

```
'Bad Boy (with Young Thug)',
'Cali Love',
'Case Closed',
'Dangerous World (feat. Travis Scott & YG)',
'Good Vibes (Za)',
'Haute (feat. J Balvin & Chris Brown)',
'I Just Wanna Party',
'Light It Up (with Tyga & Chris Brown)',
'Neighbors (feat. BIG30)',
'New Freezer (feat. Kendrick Lamar)',
'SHELTER ft Wyclef Jean, ft Chance The Rapper',
'Shabba (feat. A$AP Rocky)',
'Still Brazy',
'Taste (feat. Offset)',
'Wavy (feat. Joe Moses)',
'Why You Always Hatin?',
'Wolves',
'pick up the phone'
```

Based on knowledge about the original artist (Drake) and the genre of the original track (rap/hip-hop) we can say that the first half of our recommender system worked (selecting similar songs). After listening to the songs in the above list, most have the same tempo and a similar beat. This tells us that a person that is listening to the original track is at least being recommended similar songs. Lets try to drive this point home with an additional example.

### Song 2 - 'Piano Concerto No. 2 in C Minor, Op. 18: I. Moderato' - Sergei Rachmaninoff - performed by Khatia Buniatishvili

Here are the recommended songs from this track:

```
['Mandolin Concerto in C Major, RV 425: II. Largo', 'Bela Banfalvi'],
 ['Symphony for Strings and Continuo in B Major, RV 169 "Al santo sepolcro": I. Adagio molto',
  'Bela Banfalvi'],
 ['Symphony for Strings and Continuo in G Major, RV 149: I. Allegro molto',
  'Budapest Strings'],
 ['Concerto in E-Flat Major K. 495: II. Romance - Andante Cantabile',
  'Wolfgang Amadeus Mozart'],
 ['The Four Seasons - Concerto No. 3 in F Major, RV 293 "Autumn": I. Allegro',
  'Karoly Botvay'],
 ['The Four Seasons - Concerto No. 3 in F Major, RV 293 "Autumn": II. Adagio molto',
  'Bela Banfalvi'],
 ['Symphony for Strings and Continuo in G Major, RV 149: II. Andante',
  'Karoly Botvay'],
 ['Symphony for Strings and Continuo in G Major, RV 149: III. Presto',
  'Bela Banfalvi'],
 ['Symphony for Strings and Continuo in E Major, RV 132: II. Andante',
  'Budapest Strings'],
 ['The Four Seasons - Concerto No. 4 in F Minor, RV 297 "Winter": I. Allegro non molto',
  'Bela Banfalvi'],
 ['The Four Seasons - Concerto No. 3 in F Major, RV 293 "Autumn": III. Allegro',
  'Karoly Botvay'],
 ['Symphony for Strings and Continuo in B Major, RV 169 "Al santo sepolcro": II. Allegro ma poco',
  'Budapest Strings'],
 ['Symphony for Strings and Continuo in C Major, RV 112: I. Allegro',
  'Budapest Strings'],
 ['Klavierkonzert Nr. 21 C-Dur, K. 467, "Elvira Madigan": 2. Andante (Elvira Madigan): II. Andante',
  'Wolfgang Amadeus Mozart'],
 ['Symphony for Strings and Continuo in E Major, RV 132: I. Allegro',
  'Budapest Strings'],
 ['Carnival of the Animals, R. 125: The Swan', 'Camille Saint-Saëns'],
 ["Piano Quintet in E-Flat Major, Op. 44: II. In modo d'una marcia. Un poco largamente - Agitato",
  'Robert Schumann'],
 ['The Four Seasons - Concerto No. 2 in G Minor, RV 315 "Summer": I. Allegro non molto',
  'Budapest Strings'],
 ['Symphony for Strings and Continuo in C Major, RV 112: III. Presto',
  'Karoly Botvay'],
 ['Oboe Quartet in F Major, K. 370/ 368b: II. Adagio',
  'Wolfgang Amadeus Mozart'],
 ['2 Arabesques, L. 66: No. 1 in E Major', 'Claude Debussy'],
 ['Valse "La plus que lente"', 'Claude Debussy'],
 ['Dawn - From "Pride & Prejudice" Soundtrack', 'Jean-Yves Thibaudet'],
 ['The Four Seasons - Concerto No. 1 in E Major, RV 269 "Spring": I. Allegro',
  'Budapest Strings'],
 ['Il barbiere di Siviglia: Overture (Sinfonia)', 'Sir Neville Marriner'],
 ['Violin Partita No. 3 in E Major, BWV 1006: III. Gavotte en rondeau (Arr. J. Milone for Violin & Orchestra)',
  'Academy of St. Martin in the Fields'],
 ['Symphony for Strings and Continuo in E Major, RV 132: III. Allegro',
  'Bela Banfalvi'],
 ['Orfeo ed Euridice: Dance Of The Blessed Spirits from Orfeo ed Euridice',
  'Christoph Willibald Gluck'],
 ['Mandolin Concerto in C Major, RV 425: I. Allegro', 'Budapest Strings'],
 ['The Four Seasons - Concerto No. 2 in G Minor, RV 315 "Summer": II. Adagio e piano - Presto e forte',
  'Budapest Strings'],
 ['Concerto Per Oboe, Archi E Continuo In Re Minore: II. Adagio',
  'Venice Baroque Orchestra'],
 ['Violin Concerto No.1 Mvt II:Adagio (opening)', 'Philharmonia Orchestra'],
 ['Clarinet Quintet in A Major, K. 581: IV. Allegretto con variazioni',
  'Wolfgang Amadeus Mozart'],
 ['Clarinet Quintet in A Major, K. 581: II. Larghetto',
  'Wolfgang Amadeus Mozart'],
 ['The Ruins Of Athens, Op. 113 - Turkish March', 'Ludwig van Beethoven'],
 ['Nocturne No. 2 in C Minor', 'John Field'],
 ['The Four Seasons - Concerto No. 2 in G Minor, RV 315 "Summer": III. Presto',
  'Bela Banfalvi'],
 ['The Four Seasons - Concerto No. 1 in E Major, RV 269 "Spring": III. Allegro',
  'Karoly Botvay'],
 ['Sonata No. 11 in A Major for Piano, K. 331, "Turkish March": III. Rondo alla Turca: Allegretto',
  'Ingrid Haebler'],
 ['The Four Seasons - Concerto No. 1 in E Major, RV 269 "Spring": II. Largo e pianissimo sempre',
  'Budapest Strings'],
 ['Für Elise, WoO 59', 'Ludwig van Beethoven'],
 ['Menuet en Sol Mineur, Transcription de Wilhem Kempff d’après Le Menuet de la Suite en Si Bémol Majeur No. 1, HWV 434, 1er Cahier',
  'Anne Queffélec'],
 ['Fantaisie Impromptu in C-Sharp Minor, Op. 66', 'Henrik Måwe'],
 ['Septet in E-Flat Major, Op. 20: II. Adagio cantabile', 'Péter Szabó'],
 ['Clarinet Quintet in A Major, K. 581: III. Menuetto',
  'Wolfgang Amadeus Mozart'],
 ['Concerto "Il gardellino" RV428: II. Cantabile', 'Jean-Christophe Spinosi'],
 ['Flute Quartet in A Major, K. 298: I. Theme (Andante) and Variations',
  'Wolfgang Amadeus Mozart'],
 ['Symphony No. 40 in G Minor, K. 550: Allegro molto',
  'Wolfgang Amadeus Mozart']
```

## Conclusion and Possible Improvements

Based on the examples above, it seems that the recommender algorithm I built accomplishes its task - providing recommend

One issue with this particular recommender system is that it largely depends on the kind of playlists you follow. If the song variety of your playlists is lacking (i.e. you follow a lot of rap/hip-hop related playlists) and you are listening to songs from a different genre, it is quite likely the algorithm will not return any recommendations, simply because it will not find songs from that genre. 

With two main goals to this project (1. Create a track recommender system 2. Create an algorithm that could work in real time), at this point I can only say I accomplished #1. With the algorithm taking around 10 minutes to produce a result, deploying in real-time would not produce acceptable results for end users. That being said, possible improvements include:

* Creating a larger dataset of audio features - having a larger dataset reduce the amount of real-time compilations needed to make a recommendation
* More refined collaborative filtering approach 

Thank you for reading, I hope you found this information useful!

