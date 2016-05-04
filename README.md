# Champion Affinity ([live site](http://champion-affinity.getforge.io/))

For [Riot's API Challenge (2016)](https://developer.riotgames.com/discussion/announcements/show/eoq3tZd1), we decided to explore how the mastery data provided by the Riot API can model the relationships between champions. For example, if a summoner enjoys playing Garen, how does that predict whether they would enjoy playing Annie? This sort of data could be useful to find new champions to expand your champion pool with, or for designers, identify champions that don't have an obvious progression to another champion. We've generated a few metrics to explore this idea, using an initial population of Masters/Challenger players. You can read more about how each metric is calculated below (or on the site's [about](http://champion-affinity.getforge.io/about) page)

In this README, we'll go through:

- How to run the application on your own (in various forms)
- The overall structure of the GitHub repository
- How the backend scripts work and the dependencies required to run them
- How the visualizations work
- How the site was built and the dependencies required to deploy it

Champion Affinity was designed and created by [Chris](http://theisencr.github.io/) ([Ellipson (NA)](http://na.op.gg/summoner/userName=Ellipson)) and Rob ([T3lrec (NA)](http://na.op.gg/summoner/userName=t3lrec))

Chris designed the metrics, did the development work, and wrote most of the words you're reading here and on the site itself.

Rob acted as a sounding board for ideas (including many late nights in Discord/Hangouts), helped design the color scheme, wrote text content, and provided proofreading and feedback on the final application.

##I Just Want To Run This Thing!

Fair enough! Here's the step by step instructions for several use cases of the app. If you're confused about what some of these steps mean, refer farther down in the document. Otherwise, feel free to reach out to ask questions!

####I just want to see the final product!

Easy! We have a live version of the app up at [http://champion-affinity.getforge.io/](http://champion-affinity.getforge.io/). Go nuts!

####I want to deploy my own version (locally) with your dataset!

Dependency: Python 3.X

- Grab the codebase using <code>git clone https://github.com/theisencr/lol-mastery-analysis.git</code>
- Navigate to the <code>web</code> directory on a command line.
- Run the following command: <code>python -m http.server 8000</code>
- Navigate to <code>localhost:8000</code> on your web browser of choice.

####I want to deploy my own version (remotely) with your dataset!

We recommend deploying the site to [Forge](http://getforge.com), but any static site host will do. These instructions will be for Forge.

- Download <code>web.zip</code> in the GitHub directory however you choose to do so.
- Navigate to [Forge](http://getforge.com) and set up an account (if you don't have one already)
- Follow the instructions for setting up your own site: choose the "deploy from .zip" option
- Drag and drop (or select using a menu) <code>web.zip</code>.

Your copy of Champion Affinity should be deployed at your chosen address!

####I want to generate a new dataset and deploy!

TBD

##GitHub Structure

First, we'll go over the structure of the files in our repository.

###Root

Root contains the following:

- The <code>README</code> you're reading now!
- The python scripts that makeup the backend
- Our verification text file
- The deployable <code>web.zip</code> file of our public-facing app

###Data

This folder contains the files that support the creation and evaluation of our metrics. The files on GitHub represent mastery data from Masters/Challenger players and metric calculations from April 25th, 2016. These files can be regenerated using the backend scripts at any time, though some differences should be expected in the final results.

###Static_Data

This folder contains a single file, <code>mobafire-icon-lookup.csv</code>. This file provides some mappings that were generated manually: 
- <code>Name -> Graph_ID</code> for mapping to nodes on our eventual visualizations
- <code>Name -> Mobafire_Name</code> and <code>Name -> Mobafire_ID</code> for generating links to each champion's [Mobafire](http://www.mobafire.com) page
- <code>Name -> Image_ID</code> for looking up each champion's icon.
These mappings are referenced by our backend scripts for generating our data for our final public-facing site.

###Tests

This folder, along with <code>test_controller.py</code> in the root directory, was used to test API calls and make sure the API requests were formatted properly. Originally, we planned on building a unit test for each individual API endpoint we were referencing, just to explore how the built-in <code>unittest</code> module in Python works. The need to ship something for the contest won out, however.

###Web

Web contains our stand-alone static website that displays our final metrics. You can see the latests version of our site at [http://champion-affinity.getforge.io/](http://champion-affinity.getforge.io/). You can also deploy your own version out of the web directory; we'll discuss deployment options later on in the <code>README</code>.

##Backend (Data Collection and Metric Calculations)

Now we'll go through the meat of our application; the backend Python scripts that pull data from the API calculate our metrics, and create the .csv/.tsv/.json files that our visualizations rely on.

###Dependencies

All of our backend scripts are written in Python, using version 3.5.1. Any 3.X version should work.

The clustering algorithm we use, [Markov Clustering Algorithm (MCL)](https://github.com/koteth/python_mcl), requires the following (also on their GitHub page):
- [numpy](http://www.numpy.org/)
- [networkx](https://networkx.github.io/documentation/development/install.html) (install with pip or directly from the downloads section)
 
###How it works

There are four "types" of scripts at root:
- data
- generate
- utility
- test

####"data_" Scripts

The data scripts perform our API requests and calculates our champion affinity metric, dropping the results in the <code>data</code> folder. The scripts, in the order they are run:

- <code>data_import_champion_list.py</code> - Imports the list of current champions and their ID's to <code>champion_list_api.csv</code>.
- <code>data_merge_mobafire_icon_api.py</code> - Merges <code>data/champion_list_api.csv</code> and <code>static_data/mobafire-icon-lookup.csv</code> into <code>data/champion_list.csv</code> for later scripts. Sets a default "no meta champions" for a new <code>MCL</code> column.
- <code>data_import_top_players_master_challenger.py</code> - Imports all current Masters and Challenger summoner ID's to <code>player_id_list.csv</code>.
- <code>data_import_champion_mastery_data.py</code> - Imports all Mastery Level data for all summoner ID's in <code>player_id_list.csv</code>, and outputs the result to <code>list_of_masteries.csv</code>. File assumes the "personal" rate limit from the API spec; if you have a production API key, you can decrease the timeout in the file.
- <code>data_import_create_affinity_table.py</code> - Generates the normalized affinity table (the Champion Affinity) from <code>list_of_masteries.csv</code> and outputs the result to <code>affinity_table_normalized.csv</code>. We'll discuss how the affinity metric was generated in a later section.

####"generate_" Scripts

The generate scripts transform our <code>affinity_table_normalized.csv</code> and <code>champion_list.csv</code> into the champion centrality metric, along with readable formats for our visualization tools in our webkit. The results of these scripts are dropped into the <code>web/data</code> folder, which our visualizations access on our site.

##Visualizations

##Site Design
