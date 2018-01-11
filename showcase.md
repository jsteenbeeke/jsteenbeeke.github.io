---
layout: page
title: Showcase
permalink: /showcase/
---

I have a thing against resumes. As a software engineer with quite a bit of experience there are a lot of applications, frameworks, libraries and technologies I have experience with, some more than others. There are many ways I could list this in a resume, but if I want to do the ones I have the most experience with any justice, this is going to result in a very long resume. During job interviews I've noticed that people tend to go down the list, look at what they recognize, and start asking questions about that, but they also ignore those they don't know, even if they
are a significant part of my experience. As for my work history, there's [a complete and up-to-date list on LinkedIn](https://www.linkedin.com/in/jeroen-steenbeeke-1b13676/).

So instead of coming up with a list of keywords, or a tag cloud, I have instead chosen to highlight a number of open source projects that give an adequate impression of what I'm capable of.

## Example projects

There are a number of projects that I semi-actively maintain in my spare time, for a wide variety of use cases. Some of these projects are open source, and you are welcome to inspect these.

### Beholder
* **Technologies:** Docker Compose, Hibernate, HTML5 Canvas, Spring, Websockets, Wicket
* **Open source:** [Yes](https://bitbucket.org/jsteenbeeke/beholder-web/src), Affero GPL.

Beholder is a web application developed for my [Dungeons and Dragons](https://en.wikipedia.org/wiki/Dungeons_%26_Dragons) group, that adds a visual aspect to an imagination-heavy game. Dungeons and Dragons is a so-called "pen and paper roleplaying game", meaning that the basic game has no board or props, and you can play the game without these, though it is recommended to use some sort of grid, as well as miniatures, whenever a combat situation arises.

Our group traditionally ends up in a lot of these combat situation, and also does a lot of so-called "dungeon crawling", and we found the erasable grid we've been using to be somewhat lacking. Having to draw out the rooms of a dungeon as we explore was tedious, and was causing more confusion than aiding our visualization.

As a result, I developed Beholder, an application with two main interfaces:

 * The Dungeon Master interface, where the person running the game can set up maps, "fog of war", enemy icons, spell effects, and a whole lot of other things to aid in visualization. This is an application written using Apache Wicket, which uses Spring and Hibernate for business logic and persistence, respectively.
 * The TV view. We basically put a television on the table we're playing on, placed flat, and set our miniatures on top. The TV projects the map the Dungeon Master selects, and shows the enemies and other things of interest. This view is mostly written in plain HTML5 and Javascript (using the Canvas functionality), though it communicates with the master application through websockets.

### Andalite
* **Technologies:** ANTLR, Maven
* **Open source:** [Yes](https://github.com/jsteenbeeke/andalite), LGPL

Andalite is a library for analyzing and transforming Java and XML sources, that I use primarily for specific (non-automatic) code-generation. It allows you to specify a _recipe_, a specific set of transformation
steps, that can be applied to one or more source files. Example uses:

 * Add a property to a Hibernate entity, and also add the required XML to a Liquibase changeset
 * Create a new Wicket page (Java and HTML both)
 * Create a new REST service:
   * Create a JAX-RS interface in an API project
   * Create an implementation of this interface in a backend project
   * Create a proxy of this interface in a client project

As of writing this, Andalite's documentation is still being updated (check the _documentation-effort_ branch), though you can find plenty of examples in another project of mine, Hyperion.

### Hyperion
* **Technologies:** Hibernate, JAX-RS, Maven, Quartz, Retrofit, Spring, Wicket
* **Open Source:** [Yes](https://bitbucket.org/jsteenbeeke/hyperion), LGPL

Hyperion is my toolbox: a big set of libraries that help me develop applications quickly. It is primarily
inteded for Apache Wicket-based web applications, though more recent versions are also suitable for
Wicket-less applications.

Documentation in the master branch is rather sparse, which I am currently in the process of remedying in the `experimental` branch

### Project Myriad
* **Technologies:** Hibernate, JAX-RS, Retrofit, Spring, Wicket
* **Open Source:** Will eventually be released under the Affero GPL

Project Myriad is a new website for [the Tysan Clan](https://www.tysanclan.com/), a gaming community I have
 been a part of since 2001. Our current website was the first Wicket application I ever made (and is an excellent example
  of why you shouldn't hire me as a web designer), and the architecture is rather dated, so I figured I might as well replace the whole
 thing.
