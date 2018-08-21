---
layout: post
title: Creating a custom network visualization using the Scalar API Explorer, Part 1
summary: "Represent your Scalar book as a network using Python and the Scalar API Explorer"
categories: [tutorial]
image:
  feature: scalar-network/network_viz_header.jpeg
modified: 2018-08-21
comments: true
---

[Scalar](https://scalar.me/anvc/scalar/) is a unique and powerful open source publishing platform. Its strength lies in its ability to combine linear and nonlinear methods of exploring media, narratives, annotations, and scholarship within the single organizing structure of a **book**. Scalar also provides [several out-of-the-box options](http://scalar.usc.edu/works/guide2/visualizations?path=index) to visualize your data - including as an interactive network visualization.

But what happens when you want to customize your visualizations beyond what the Scalar presets allow for? I recently ran up against this issue and decided to find a way to create a network visualization "from scratch" (in reality, leveraging a number of excellent open source tools, demos, and APIs!) I used the [Scalar API Explorer](http://scalar.usc.edu/tools/apiexplorer/) to export data about our pages and tags between pages, prepared the network data with Python and various packages (NetworkX, BeautifulSoup, etc.), and wrote a custom network visualization using D3.js and Canvas.

I waded through a fair bit of code and experimentation along the way, and I'd like to share with you some notes, lessons, code, and tools that reuslted from that process. I also tried to identify several places where you may wish to deviate from my process, or to experiment further, depending on your Scalar book and your own vision of what such a visualization might look like.

This tutorial is written as two parts:

* **Part 1**: Represent your Scalar book as a network using Python and the Scalar API Explorer
* **Part 2**: Create an interactive visualization of the network data using D3.js and Canvas

In Part 1, I will introduce the goals of this process and walk through the Python code needed to prepare your data. 

<!--more-->

## Introduction

### Why a custom visualization? 

Here is an example of a [built-in network visualization generated in Scalar](http://scalar.usc.edu/works/jewish-cafes/network-visualization-of-caf-regulars).

And here is that same book reimagined as a [custom visualization using D3.js and Canvas](https://zoews.github.io/interactive-network-viz/viz2.html) – *note that as of 8/21/18 this visualization is still a work in progress, with more photography/text and reorganized tagging to come soon.*

If you want to use a pre-built network visualization in Scalar, the process is fairly simple: create a new page, and assign that page the **Connections** layout style. Now when you visit that page, Scalar will fetch all pages and tags between pages (which are expressed as "x is a tag of y" and "x tags y" in the **Relationships** tag of a page). Scalar will then use the JavaScript visualization D3.js to automatically generate a [force directed](https://en.wikipedia.org/wiki/Force-directed_graph_drawing) network visualization.

For our purposes, we encountered a number of limitations with this format. First, the network visualization takes considerable time to load a book with hundreds of items (up to 30 seconds). Behind the scenes, Scalar is assembling the data by fetching it in batches of 25 pages/tags/media items at a time, and stopping to rest between each request. This pattern of request-wait-request-wait results in a slowed-down experience for the user.

Second, the prebuilt options do not provide any options for customizing the final output. We wanted to create a network visualization that (1) excluded any "orphan" pages (pages that lack any tag relationships), (2) includes a more streamlined visual style that matches other aspects of our project, (3) populates a sidebar with names/descriptions/images from the Scalar book, and (4) allows the user to take the additional step of following a link directly to the Scalar page in focus.

### Our goal is to prepare data for a network visualization

While your ideal visualization may differ from ours, the basic process for preparing the data will be the same:

* Step 1: Download your Scalar book data using the Scalar API Explorer
* Step 2: Restructure the raw Scalar output as network data
* Step 3: Add extra network analysis indicators to the visualization, and 
* Step 4: Gather and export data as single .json file

When deciding how to prepare data for a visualization, it's helpful to have a vision of what the final output will look like. In this case, I used [this excellent code sample from Mike Bostock](https://bl.ocks.org/mbostock/ad70335eeef6d167bc36fd3c04378048) as a starting point to my design. I noticed that this network visualization relies on a single JSON file called "miserables.json" that consists of a collection of "nodes" and a collection of "links". Finding this code sample allowed me to set a goal for my data preparation: **represent all tags and pages as links and nodes within a single .json file**.

### Wait, what's a link!? What's a node!?

Okay, I realize the above paragraph may have sound quite strange if you haven't been mucking around network visualization design lately! Let me explain what's happening here.

A network visualization is a way to represent the relationships between many different individuals/items/entities all at once. It provides your user a summary view of the overall structure of a community (is it made up of totally disconnected cliques? Does everyone know one super-popular person?) Network visualizations often provide methods for exploring and investigating these connections, such as click-and-dragging, zooming, etc. Also, they tend to be [quite pretty and compelling](https://flowingdata.com/category/visualization/network-visualization/).

One way to think of a network is a set of points or "nodes" (which could be anything from people to countries to books to cities) and a set of relationships between those nodes, which are commonly called links or (less intuitively) "edges."

You can also make things more complicated: sometimes the relationships between nodes are one-way (Alma sends letters to Julie, but Julie doesn't Alma back) or unidirectional, and other times they are two-way or bidirectional. Also, some relationships might be more important than others, which is sometimes captured as the "weight" of an edge/link. 

For our purposes, we are focused on the "nodes" and "links". In the context of a Scalar book, you can think of the nodes the same way Scalar thinks of them: pages and media items. In our team's example, these pages refer to individuals, cities, cafes, photographs, drawings, etc.

We can also think about "links" in terms of tags. That is, if a page tags another page, we can think of a link existing between those two pages. In Scalar, tags are unidirectional (page A tags page B, or conversely, page A is tagged by page B), though two tags can exist back and forth between two pages, thus creating a bidirectional relationship.

Our goal will be to capture information about the "nodes" of our network (in our case, pages of specific kinds) and "links" of our network (in our case, tags recorded in the Scalar book). If you've already created a Scalar book with at least two pages and at least one tag, then congratulations! You have enough data to generate a network visualization.

## Step 1: Download your Scalar book data using the Scalar API Explorer

The Scalar team has done an excellent job anticipating that some users (like you!) may wish to move beyond the presets and limitations of the out-of-the-box platform features. So to facilitate that process, they created an easy mechanism for dumping out some or all of the data from your Scalar book.

The main tool to accomplish this task, for us, will be the [Scalar API Explorer](http://scalar.usc.edu/tools/apiexplorer/). This is a fun tool to explore in general. For today, we're going to follow a targeted approach:

* Follow the first direction under "Query" and copy and paste the URL from your book into the text box. I generally use the Welcome page in my Scalar book, but I believe this should work no matter which page you choose.
* In the options for what to export, select the final one: "All of the book's content and relationships (potentially slow, use with care)"
* Don't change any of the other options. You should be ignoring past versions and (most importantly) exporting as RDF-JSON.
* Click the big "Get API results" button. If this doesn't seem to work, try refreshing, copy and pasting a different URL, etc. You will know the API is fetching your data when the text box below fills with "Retrieving content..." and it may take a minute or longer to execute.

Here's what the results look like:

![]({{ site.url }}/img/scalar-network/api_output.png)

The Scalar API Explorer will also generate some results in the "Text" field below, but we're going to ignore this second field and focus entirely on the output in the Data text box (the Text is also included within the Data output, it's just harder to single out right now.)

To use this output, we must download it locally to our computer. Copy the entire contents of the Data text field to the clipboard. Next, open up your text editor of choice (I use Visual Studio Code for these sorts of tasks, but you may prefer Sublime Text, or [another simple text editor](https://github.com/collections/text-editors)) and paste the output into a blank file. Finally, save this file with .json extension to indicate it is a JSON file. I suggest the filename **scalar_output.json**.

[JSON](https://www.json.org/) is a format desgined to easily move collections of data between programs. For a JSON file to be read by a Python or JavaScript program, it must follow certain syntax conventions. However the formatting itself is plain text, and thus is interchangeable with a plain text (.txt) file. It becomes a .json file simply when given the .json extension.

## Step 2: Restructure the raw Scalar output as network data

Now that we have our raw data, we must re-structure it a way that will be useful for our network visualization later on.

*(Here's where you may wish to jump straight to one solution - a ready-to-run Python script called [generate_scalar_network.py](https://github.com/zoews/interactive-network-viz/blob/master/parsing-tools/generate_scalar_network.py). To download directly from GitHub, click the "Raw" button on the preview page, and save the resulting page to your computer as a Python file. Place your **scalar_output.json** file in the same directory, and run the script in the terminal. However, I recommend you go through the step-by-step breakdown below. It's likely that you will wish to customize some of the behaviors of the script in order to capture or omit elements of your Scalar book, in whatever way is most meaningful for your end goal. Plus you may wish to take the functions we describe below and further reimagine them, add new functions, etc. Feel free to choose your own adventure!)*

Python is an excellent tool for this task! Open up your text editor of choice again and save a new file as **generate_scalar_network.py** . We will be using Python 3 for our code examples here. If you need a refresher on using Python 3, such as on using pip to install Python libraries (you'll need to install the libraries below) or running Python programs from the command line, you may wish to explore [a few resources](https://www.python.org/about/gettingstarted/) before continuing on.

### Loading Scalar data into Python

Our first step is to import any libraries we'll need. At the top of your new Python program, let's add the following:

{% highlight python %}
{% raw %}
import json
from bs4 import BeautifulSoup
import networkx as nx
import math
{% endraw %}
{% endhighlight %}

Here's what the following libraries will help us with:

* json contains methods to import, parse, and export JSON in Python
* BeautifulSoup is a library for parsing HTML inn Python. This will help us extract descriptions and links from our Scalar pages.
* NetworkX contains a number of network analysis tools. We will use it to calculate betweenness centrality scores for our nodes (more later)
* Math adds extra math functionality to Python. We will use it to round floats down to the nearest integer.

Next, we need a function to read the raw JSON dump from the Scalar API Explorer into a useful Python data structure. Below is a function that accepts JSON as input and returns the data as a dictionary for Python to use


{% highlight python %}
{% raw %}
def loadScalarData(input_file):
    # loads JSON data exported from the Scalar API Explorer tool
    # and parses as a dictionary for further manipulation with Python here

    data_for_output = {}

    with open(input_file, "r", encoding="utf-8") as read_file:
        data = json.load(read_file, strict=False)
        for key in data:
            out_dict = data[key]
            
            # also add the name of the current dictionary to the dcitionary itself and assign to the key 'key'
            # (because we're outputting a list of dictionaries, the key name would be lost otherwise)
            out_dict['key'] = key
            data_for_output[key] = out_dict
    
    return data_for_output
{% endraw %}
{% endhighlight %}

### Structuring Scalar data as a network of nodes and links

Here's where things get trickier. Our goal is to take this unfiltered, raw export of Scalar data and isolate the data that will be relevant for our network visualization: nodes (in the Scalar book, pages) and links (in the Scalar book, tags). To accomplish this task, we'll use a few functions below.

*For this first function, please notes:  **the function only looks for pages that are tagged at least once somewhere in the book**. I omit any page that is completely disconnected from all others. These disconnected, floating pages are sometimes called "orphan nodes". To include orphan nodes would require a different method of generating the lists of links and nodes that I apply here, and is also outside the scope of our team's visualization. That being said, you may find it useful to include these floating nodes in your own visualization. If so, please look for a function that includes orphan nodes in Part 2 of this tutorial.*

First, here is the function to generate a list of all tags and all (tagged) nodes:


{% highlight python %}
{% raw %}
def getTagsAndTaggedNodes(reference_dict, excludeMedia=True):
    # extracts all tags and pages that are tagged somewhere in the Scalar book
    # represents each item (tags/pages) as a dictionary
    # and returns a list for each item type

    tags_for_network_links = []
    tagged_pages_for_network_nodes = []


    for key in reference_dict:
        current_item = reference_dict[key]
        if 'urn:scalar:tag:' in current_item['key']:
            body = current_item['http://www.openannotation.org/ns/hasBody'][0]['value']
            target = current_item['http://www.openannotation.org/ns/hasTarget'][0]['value']

            # ignore tags here that involve media nodes (almost always images)
            if excludeMedia:
                if 'media' in body or 'media' in target:
                    continue
            
            if body not in tagged_pages_for_network_nodes:
                tagged_pages_for_network_nodes.append(body)

            if target not in tagged_pages_for_network_nodes:
                tagged_pages_for_network_nodes.append(target)
            
            tag_dict = {
                'source': body,
                'target': target,
            }
            
            tags_for_network_links.append(tag_dict)
    
    return tags_for_network_links, tagged_pages_for_network_nodes

{% endraw %}
{% endhighlight %}

From poking around the Scalar output JSON, I figured out that the prefix indicating a tag is 'urn:scalar:tag'. I further discovered that the "body" (or starting point) and "target" (or end point) of a tag is contained in current_item['http://www.openannotation.org/ns/hasBody'][0]['value'] and current_item['http://www.openannotation.org/ns/hasTarget'][0]['value'] . The goal of this program is to extract this data into tidy tags (or pairs of 'body' and 'target' values). Then as we go, we fill up the **tagged_pages_for_network_nodes** variable with a list of every unique page we discover in our tags. At this point, we are keeping track only of the name of our pages, not any of their other attributes or content.

Note that this function includes an **excludeMedia** parameter that defaults to true. This is because, for our team's visualization purposes, we did not want to include Media pages in the same way that we include non-media pages. However, if you would like to include both types of pages in your network visualization, simply add "excludeMedia=True" when you call **getTagsAndTaggedNodes()** in the **main()** loop (see Step 4 below).

**getTagsAndTaggedNodes()**returns two lists: **tags_for_network_links** and **tagged_pages_for_network_nodes**. While the tags are 100% ready to go at this point, we need to do further parsing of the nodes to gather all of the relevant .

### Further parsing the network nodes (pages)

Our list of nodes right now is just a collection of ids. We need to actually get all of the relevant data that we'll need for our visualization. At this point, it's helpful to ask the question: **what do we actually want our users to see and/or be able to interact with?**

In our case, our team would like our users to:
* Read the name of people, cafes, and cities (stored in the name field of the relevant pages)
* See a link the user can follow to the full Scalar page (stored as the URL of the Scalar page)
* For pages with extra content like text and embedded photos, view a preview of this content on the sidebar (stored within embedded media information in the Scalar page. Note that this may require additional fiddling to work for your data).

Here is the function to gather all the data relevant for those tasks (which is the most complex of all functions we'll use in Part 1):

{% highlight python %}
{% raw %}

def parseNodes(scalar_data, pages_identified_in_links):
    # fetch all relevant data about the pages and return as list of nodes
    # including checking for additional page data (text and embedded images)
    
    # note - you may wish to modify these parameters to include significant data
    # scraping tags is more straightforward, whereas metadata categories/etc. will be specific to your Scalar book

    list_of_parsed_nodes = []
    for node in pages_identified_in_links:
        current_node_dict = scalar_data[node]
        url = node
        _id = node
        _name = ""
        imageURL = ""
        description = ""
        extraData = "no"

        # look for reference to media
        references = current_node_dict.get("http://purl.org/dc/terms/references")
        imageURL = None
        description = None
        if references:
            if "media" in references[0]["value"]:
                # if reference to media found, fetch URL and description from media page
                # using BeautifulSoup to parse the HTML descriptions
                content = scalar_data[node]["http://rdfs.org/sioc/ns#content"][0]['value']
                soup = BeautifulSoup(content, "html.parser")
                imageURL = soup.find("a")["href"]
                description = soup.text

                # ceate an abbreviated description (to encourage users to go to Scalar for full description)
                if len(description) > 80:
                    description = description[0:100] + "..."
        
        # when directing the user to the Scalar page via the "url" attribute in the visualization
        # we can ignore the extra version suffix, such as "cafe_central.4" and simply send them to
        # "cafe_central" -- otherwise Scalar may omit the page description (for some reason)
        if url[-2] == '.':
            url = url[0:-2]
        elif url[-3] == '.':
            url = url[0:-3]
        elif url[-4] == '.':
            url = url[0:-4]
        
        try:
            # make sure the node is fetching the name correctly, assign to _name
            _name = current_node_dict['http://purl.org/dc/terms/title'][0]['value']

        except:
            _name = node

        # assemble a dictionary to describe the current node
        node_out = {
            'id' : _id,
            'name' : _name,
            'url' : url,
            'colour':	"#2a2a2a"
        }

        # optional imageURL and description paramaters for the nodes, if the pages include this data
        # will also include an "extraData" parameter to indicate if either attribute exists, for use by
        # the network viz later on
        if imageURL:
            node_out["imageURL"] = imageURL
            extraData = "yes"
        if description:
            node_out["description"] = description
            extraData = "yes"
        node_out["extraData"] = extraData

        list_of_parsed_nodes.append(node_out)

    return list_of_parsed_nodes

{% endraw %}
{% endhighlight %}

This function works by iterating through the list of tagged pages. For every page, we start to construct all of the variables we'll need to describe our nodes, such as the name, the URL, etc. Each of these variables eventually gets stored in a single dictionary, **node_out**, and appended to our working variable **list_of_parsed_nodes**. After iterating through every page, we can then return the **list_of_parsed_nodes** variable with all of the data we need to describe the nodes in our network.

You may notice a few peculiar blocks of code. First, I included a block of code to deal with text descriptions and embedded images. If your page includes this kind of data, you may wish to include it along with other information about your nodes. That way, later on when you visualize the network with JavaScript, you can use the text and images to generate a nice little preview (in our case, within the sidebar). However, the text and/or image appear in the Scalar API Explorer output JSON as a string of raw HTML. This is probably overkill for our purposes - it would be better to isolate the text and isolate an embedded URL.

BeautifulSoup is a Python library that excels in just this situation: parsing website HTML in such a way that Python can quickly identify the meaningful parts and pass them along to simpler variables. We do that here in the if "media" block. **Note: this is another area you may wish to rewrite a bit, depending on how your Scalar book is structured.**

Second, I wrote a series of if statements to deal with stripping out the page version suffix of pages. In Scalar, if you continue to edit pages, the platform will automatically create new versions of the page such as "page.2", "page.3", and so on. The most recent version is included in the tags and in the names here. However! I've noticed that dropping a URL into your browser with that "page.3" ending leads the user to a strange limited version of a Scalar page. To get around this issue, we can remove the ".3" suffix entirely, and the URL of "scalar.usc.edu/blahblahblah....page" will go to the correct place.

At the end of this function call, it will return a list of nodes - but unlike the first list that contained only names, we now have a list of dictionaries packed with (almost) all of the information about the nodes we'll need for our network visualization. 

## Step 3: Add extra network analysis indicators to the visualization

At this point, we could call it a day and declare our node and link lists complete. However! I found it helpful to add one more piece of data to our network: betweenness centrality scores.

Why betweenness centrality? Consider the question: how important is any one node to the network as a whole? You might say "the node in the middle is the most important." But where exactly is the "middle"? And is the concept of "in the middle" a sufficient indicator of what makes any node important to the network as a whole?

Imagine for a moment a city with a subway or train network. You may imagine several subway lines running from the far edges of the transit network (where the stops are more isolated) into the downtown area. Closer to downtown, more and more lines may criss-cross at any given subway stop. Which of these subway stops is most important? Let's imagine one of these downtown stops is suddenly closed due to a mechanical failure. One way you might describe the degree of inconvenience of this closure is to say "how many train routes will now be impossible for a commuter to travel? Or how many will require a lengthy detour?"

Betweenness centrality is a measure of exactly that concept: of all possible routes an individual could travel from starting-point to end-point in the network, how manny of the most efficient routes (the kind we usually take to commute somewhere) travel through a given node? If the number is very high, we can say that this node has a high betweenness centrality score. If it's low, we say it has a low score. (If I've piqued your interest, here's an [interesting writeup with visualizations](https://cambridge-intelligence.com/keylines-faqs-social-network-analysis/) of BC and other social network measures.)

How do we add betweenness centrality to our network? We could wait until the JavaScript visualization step to generate these scores – but then we have to waste the user's time calculating the numbers. Instead, let's include a process now to add those scores to the nodes ahead of time.

First, we need to generate the scores. Luckily, the NetworkX network analysis package in Python has a pithy way to accomplish this goal. Below is a function using NetworkX (which we imported to our program as **nx**) to generate scores:

{% highlight python %}
{% raw %}
def generateBetweennessCentrality(final_list_links, final_list_nodes):
    # use networkx to generate betweennenss centrality for nodes and return as dict

    nodes_for_nx = [node['id'] for node in final_list_nodes]
    edges_for_nx = [(n['source'],n['target']) for n in final_list_links]

    G = nx.Graph()
    G.add_nodes_from(nodes_for_nx)
    G.add_edges_from(edges_for_nx)

    return nx.betweenness_centrality(G)
{% endraw %}
{% endhighlight %}

At this part of the process, you may wish to add even more network analysis measures. If you're curious, NetworkX also includes a rudimentary [network viz drawing tool](https://networkx.github.io/documentation/networkx-1.10/reference/drawing.html), which would help you generate a visualization directly from Python! However, D3 is much more capable at generating pretty and interactive visualizations, as you will see (I hope!) in Part 2.

We also need to write one more piece of functionality: adding the betweenness centrality scores of our nodes to our list of node dictionaries. This is slightly inconvenient given the way we've structured our node data as a list of dictionanries (in a more complex program, it might be a good time to ask the question: should I have created classes at some point?) Here, we can iterate over each node in our list, add the attribute, and reassemble and return our new list:

{% highlight python %}
{% raw %}

def addBetweennessCentralityToNodes(betweenness_centrality_scores, final_list_links, final_list_nodes):
    # add BC scores as a new parameter, 'betweeness_centrality_score'
    # as an integar between 0 and 100 to use in network viz (as font size, etc.)

    final_output = {}
    final_output['links'] = final_list_links
    final_output['nodes'] = []

    for node in final_list_nodes:
        current_bc_score = betweenness_centrality_scores[node['id']]
        # let's convnert this percentage value to an integer between 0 to 100, roundng up 
        node['betweenness_centrality_score'] = math.ceil(current_bc_score * 100)
        final_output['nodes'].append(node)

    return final_output

{% endraw %}
{% endhighlight %}

## Step 4: Gather and export data as single .json file

It's time to bring it all together! We have all of the pieces we need. Now we need to write a single **main())** function that calls the functions, stores their output as we go, and exports the single final output as a .json file. 

Note that in Python, the **main()** function is a way to encapsulate everything we wish to do in a program when we call it from the command line. When paired with an **if __name__ == "__main__" : main()** block, the **main()** function will be called when you run the Python program from the command line. This also helps prevent a common problem if you were to import **generate_scalar_network.py** as a module in another program (in the same way we imported json, NetworkX, etc.) Python will actually go ahead and immediately run anything in a module that isn't contained within a function. As a result, if we have our code dangling outside of the **main()** function, it will behave just as planned when we run it from the command line... but may do strange, out-of-order things if we import it in our program. (We can still call main() in the future even after importing the program as a module, however! We would do this by calling generate_scalar_network.main())


Here is the code we'll need to carry out our program, all packaged in a tidy **main()** function, as well as the __name__ block discussed above:

{% highlight python %}
{% raw %}

def main():

    # set filenames for input .json (from Scalar API explorer) and output .json
    input_file = "scalar_output.json"
    output_file = "clean_scalar_data.json"

    # load Scalar data .json file into Python data
    scalar_data_dict = loadScalarData(input_file)

    # generate final list of links and nodes
    final_links, linked_pages = getTagsAndTaggedNodes(scalar_data_dict)
    final_nodes = parseNodes(scalar_data_dict, linked_pages)

    # generate betweenness centrality scores for nodes using NetworkX 
    betweenness_centrality_scores = generateBetweennessCentrality(final_links, final_nodes)

    # one final step to collect links and nodes (with betweenness-centrality scores added) into a single dictionary
    # (this is assuming this is the most useful way to pass the data to a network visaulization
    # BUT you may wish to modify or rewrite this function if your visualization calls for another way to organize)
    final_output = addBetweennessCentralityToNodes(betweenness_centrality_scores, final_links, final_nodes)

    with open(output_file, "w") as write_file:
        json.dump(final_output, write_file)
        print('clean data successfully written to disk as "clean_scalar_data.json"')


if __name__ == "__main__":
    main()

{% endraw %}
{% endhighlight %}

And you're done!! If everything went as planned, when you run your **generate_scalar_network.py** script in a directory (and remember to plop in the **scalar_output.json** dump from the Scalar API Explorer beforehand) you should see a tidy **clean_scvalar_data.json** file in the same folder. 

To view your JSON, you have a few options. You can always open it up in your text editor of choice. Mozilla Firefox actually does an excellent job of visualizing JSON and letting you explore the hierarchy a bit. Here's a sample of what that looks like:

![]({{ site.url }}/img/scalar-network/JSON_in_firefox.png)

The Online JSON Viewer provides a very similar feature -- copy and paste the JSON text data into the field in the Text tab, and then switch to the Viewer tab in the top-left to visualize and explore the results.

If you'd rather download a ready-to-go version of the code excerpts above, you may also download [generate_scalar_network.py](https://github.com/zoews/interactive-network-viz/blob/master/parsing-tools/generate_scalar_network.py) using the Raw link at this GitHub page. You can also test it out on an version of our Scalar book's export data at [scalar_output.json](https://github.com/zoews/interactive-network-viz/blob/master/parsing-tools/scalar_output.json)

If something goes wrong, don't give up! It's likely means that there's something about your scalar book export that requires a little more fiddling in the way we've implemented the functions here. Feel free to shoot me an e-mail if you feel totally stuck too!

## Wrapping Up

You've now got some network data to work with! If you're comfortable getting a JavaScript excerpt up and running, you could even plug in this network JSON into the [bl.ocks network visualization example](https://bl.ocks.org/mbostock/ad70335eeef6d167bc36fd3c04378048) we mentioned above. In Part 2, we will explain in detail how to integrate these pieces and pretty radically redesign this code excerpt to work better in highlighting your Scalar data.

Congratulations for making it through! More next time. And in the meantime: happy experimenting!
