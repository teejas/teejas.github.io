---
title: 'Juko V2'
date: 2022-07-24
draft: false
tags:
  - music industry
  - music services
  - juko
  - listener services
  - music discovery
  - novel discovery
---

# Table of Contents
1. [Introduction to Juko](#introduction-to-juko)
2. [Juko V1: The Original Idea](#juko-v-the-original-idea)
3. [Juko Reimagined: Interactive Music Discovery Sessions](#juko-reimagined-interactive-music-discovery-sessions)  
	a. [Design: User Story + Figma](#design)  
	b. [Building: Django, React, Docker, GHA](#building)  
	c. [Deploying: GKE, Kubernetes, Kustomize](#deploying)  
4. [Conclusion](#conclusion)
5. [Appendix](#appendix)


# Introduction to Juko
<a name="intro"></a> 

Juko started out as a "startup idea" after I graduated from college (Dec 2019) and was struggling to find work at the start of the pandemic (a.k.a. the pando). I use the quotes because I'm not sure I ever seriously believed this idea had the legs to become anything substantial. The reason I worked on the idea at all, aside from needing something to fill my time between interviews and rejections, was because I believed it was solving a problem for me: I couldn't be bothered to skip through songs while biking, but I still wanted to find new music while biking. So in the interest of making my life easier, or so I believed, I built a mobile app using React Native which would take a user's playlist and generate recommendations from it. Then playback, which would happen in the app itself, would skip songs every 30 seconds by default. If the user interacted with the app to "Like" or "Favorite" the song, it would cancel the default skip allowing the user to listen to the song in its entirety. It was an over-engineered mess which was meant to solve an entirely made-up personal pain point, but it was mine.


# Juko V1: Discovery on-the-go
<a name="jukooriginal"></a>

The original vision for Juko, as stated before, was to provide a way to discover music while it wasn't convenient to interact with the Spotify app (or whatever streaming service you use). For example, when you were riding a bike. By skipping songs by default the user didn't have to interact at all until they came across a song they liked. Furthermore, recommendations learned from songs you liked. There are a couple obvious flaws there. Firstly, the user would inevitably **have** to interact with the app assuming they ever came across a song they liked. Secondly, as recommendations got better and the user was provided more songs they would want to hear entirely they'd have to interact more with the app meaning the app became more cumbersome as it became more useful. Not a great model.

The first issue, inevitably having to interact with the app, was the result of a core misconception in the idea, that the best solution to this issue was a mobile app. If the pain point is being solved by reducing user interaction with the listening app, you probably don't want an app at all. Just some background application which provides playback while recommendations are generated behind the scenes and appended to the queue. Then there could be some mechanical input like a button on the bike for the user to say they "Like" or "Favorite" a song. I remember my brother suggesting this exact thing when I first proposed Juko, I laughed it off at the time being bull(headed)ish on software-only solutions at the time.

The second issue, having to interact more with the app the better its recommendations got, was again a failure borne of the fact that you had to interact with an app at all. Proposing *another* app to discover music by, which still required some form of interaction was doing very little to solve the original proposed pain point. If you needed to take input it would have to be by another, more convenient means, such as the button proposed previously, or maybe by taking the "input" from somewhere else. Perhaps from an accelerometer on the bike and in order to indicate you like a song and don't want to skip it you have to speed up to a certain speed. That's an interesting idea. But that's beside the point. 

The idea to implement a solution to this problem in another mobile app was flawed to begin with. A mobile app would always require some sort of user interaction on the phone, unless it took the input some other way. It was just putting another facade on the same system, the 30-second default skip was just a gimmick. A better solution to this issue probably wouldn't have a frontend to interact with whatsoever, it would just run an application to provide playback and on the backend it would generate recommendations and queue them up. User input could be taken another way, maybe by a button on the handlebars. But building mechanical solutions like that are of little interest to me at the moment. I'd like to build something entirely from the comfort of my laptop on a cluster running in a cloud of my choosing. It's just easier to do and maintain, more fun, more in my wheelhouse, and more within my control--I can take it down whenever I want.


# Juko Reimagined: Interactive Music Discovery Sessions 
<a name="jukoreimagined"></a>

I decided to revisit Juko when someone who helped me a lot with the original iteration passed, a few years after implementing the first version. I realize that a failure of the original idea was the existence of a mobile app at all, it did little to address the pain point I had concerning hands-free music discovery. But working on that original idea I found something out, music discovery didn't have to be hands-free to be enjoyable. I was having a blast kicking up the app in the Expo demo environment and just getting recommendations fed to me in 30 second snippets until I liked something. I'd click "Like" to improve recommendations and isntantly start building a playlist of liked songs from the session. I was getting a lot more use out of Juko as a sort of session manager for music discovery. I could track songs I liked in an individual session, and everytime I sat down at the computer it felt like I was starting fresh. Future sessions weren't contingent on previous ones at all, so recommendations felt new (to begin with) every time.

This sort of segues into another idea I've been flirting with: novel discovery. I think a big thing missing from music recommendation services today is being recommended music that's way out of the norm relative to the "seed track" provided to generate those recommendations. All the music I'm recommended is too similar to the music I already know. Thinking about this graphically, if all Spotify tracks are nodes in a graph the edges can be thought of as a set of values each measuring how similar the two tracks are in a different dimension: how similar they are instrumentally, lyrically, tonally, etc. Typical recommender systems seem to try and recommend tracks that are similar across all dimensions. Novel discovery, or novel recommendation, would happen when recommended tracks are similar to the seed only in a subset of dimensions. Then you start hearing things far out of the ordinary but still related by some feature. 

Building this graph as a computing model that actually relates tracks on a set of audio features then recommends based on those relationships is possible but requires a fair amount of work itself, that's a project worth tackling separately and then incorporating into this project when it's complete. A simpler way to mimic this kind of novel discovery is to just generate the recommendations on a smaller set of seed tracks. Instead of using the whole playlist just use the most recent tracks added to it. That way the recommendations are less "fitted", having less data to train on, and you will receive recommendations you would not usually get. Of course this in some sense means the recommendations will be worse, they won't be as well-trained on the whole set of your listening tastes (as represented by the tracks in the seed playlist). But tastes change, and your current listening tastes are better represented by the songs you most recently listened to and added to the playlist. 

So that's novel discovery taken care of, or at least a spoof of it. The next thing to tackle is the interface. Now user interaction is required and encouraged. But it should still be simple. Listening is the key here, you should sit back, listen to the snippets, only have to hit a key to indicate approval ("Like" a track), and then get a summary of your listening session at the end. Maybe we can consider adding more inputs for the user--I did think a kind of superuser interface would be cool where the user can inject different parameters into the backend to influence the recommendations comming out from Spotify (i.e. increase speechiness of recommended tracks)--but we'll start by keeping it simple (stupid).

<a name="design"></a>
## Design

### User story

From the perspective of someone looking to discover new music: 

### Story &rarr; Design

Using Figma, share some screenshots here

<a name="building"></a>
## Building

### Backend with Django

#### Spotipy

### Frontend with React

#### OAuth

### Dockerfiles

### Github Actions

There is a simple Github Actions workflow (identical between `walkman-frontend` and `walkman-backend`) which builds and pushes a new image to Dockerhub with the tag `tj1997/walkman-{frontend, backend}:latest` on every push/merge to the `main` branch of the respective repo.

<a name="deploying"></a>
## Deploying

### GKE Autopilot

I used GKE Autopilot to quickly spin up a Kubernetes cluster which lets me pay-per-pod rather than pay-per-node. I thought this was preferable because a. I could get something up quick and b. I wasn't running many pods. Sure enough I was able to quickly get a cluster up and `kubectl` access, the only extra work I had to do was install and configure the [`gke-gcloud-auth-plugin`](https://cloud.google.com/blog/products/containers-kubernetes/kubectl-auth-changes-in-gke). I also set up external-dns to handle creating DNS records for my ingress resources, but I'll discuss the kube admin work in greater detail in another post.

### Kustomize

The Kubernetes resources for Juko (such as the deployment, service, and ingress for both the frontend and backend) are deployed using Kustomize. A `kustomization.yaml` file is used to define what resources to build. Then you just run `kubectl apply -k path/to/dir/containing/kustomization.yaml` to apply all those resource manifests. This is a nice way to have one manifest per Kubernetes resource, rather than having your whole service and all necessary resources defined in one messy manifest. I also could avoid setting up Helm and having to create a chart and values file. Not fun. It's also what we're migrating to at work (from Helm), so figured I'd get hands-on with it. 

### Deploying changes

Changes can be easily deployed by building a new image with the tag `:latest` and deleting the frontend and/or backend pod to force a new deployment with the latest image.


<a name="conclusion"></a>
# Conclusion 

<a name="appendix"></a>
# Appendix 

- [Juko-V2 Web App](https://juko.tejas.wtf/)
- [Juko-V1 Code](https://github.com/teejas/serendipify)
- [Juko-V2 Frontend Code](https://github.com/teejas/walkman-frontend)
- [Juko-V2 Backend Code](https://github.com/teejas/walkman-backend)


