---
layout: post
title: "Prettifying old tests"
date: 2017-09-22
tag: testing
---

Ever heard of [FitNesse](http://fitnesse.org/)?  If not, imagine what the grandfather of [SpecFlow](http://specflow.org/) would look like and how it might be used.  

If you imagined HTML tables filled with wiki-style syntax to perform assertions on the data placed in them you're on the right track.

A small example from the [FitNesse Two Minute Example page](http://www.fitnesse.org/FitNesse.UserGuide.TwoMinuteExample):

![fitnesse-example](\assets\images\posts\enhancing-old-tests\fitnesse-example.png)

And the backing markup that creates the table:

![fitnesse-wiki-markup](\assets\images\posts\enhancing-old-tests\wikimarkup.png)

This seems relatively simple for straightforward tests.  While maybe not an ultra-modern way of writing tests, enabling testers who aren't necessarily able to write code with the ability to write automated tests is a good thing.  

These tests can be executed in the browser and are backed by code in a similar fashion to SpecFlow.  Users don't need to install anything at all in order to get started; only go to a url.

Recently, I came across a scenario where a small suite of these tables were being used to verify data coming from a 3rd party vendor.  These tests were necessary as we need to make sure our data arrives in the expected format and expected value.  

The problem was some of these FitNesse tables had over 90 columns.  This reduces the readability of the tests to almost zero since the only way someone reading the results could find a specific failure is manually searching the results or CTRL+F on the results page.  Any web page with a 90+ column table isn't navigable using this format.   

Be that as it may, these types of tests have been around for a while and simply porting them to something other than FitNesse wasn't really an option.  This poses a problem to us as these types of tests hold a key role in ensuring the provider services our apps integrate with are working properly.  The search was on for a better way to visualize these results.

### D3.js

Enter [d3.js](https://d3js.org).  I encourage you to check out the link, but essentially d3.js specializes in rendering complex graphs of data in html (often as `SVG`s).  After doing some research on non-tabular wasy to present tabular data I chose a [Sunburst diagram](https://datavizcatalogue.com/methods/sunburst_diagram.html).

Visualizing these large FitNesse results tables as a zoomable sunburst diagram allows readers to view all tables at once without needing to scroll.  The sunburst diagram also offers a high-level view of the results ("Did any tests fail?") as well as the ability to zoom in to a specific table for more in-depth analysis.  Placing additional details for results at specific layers on the diagram eases readers into the mass amount of info presented with these results.

Ideally, the highest level uses a unique color for each table and red/green colors for each database column to show fail/pass, respectively.  Hovering over a 'table' section section reveals database table-specific info and hovering over a specific section for a single test shows fail/pass details, the database table and column under test.

Here's an example of the generated diagram:

![sunburst](/assets/images/posts/enhancing-old-tests/sunburst.png)

First, the FitNesse test results page is scraped for these database column test results and proper JSON structure is built.  My technique was nothing magical - just some Regex and selectors to get at the necessary HTML elements with the test result data I needed.  According to [this example by Mike Bostock](https://bl.ocks.org/mbostock/4348373) the necessary JSON for a sunburst diagram in d3.js would look like this:

```json
{
    "name": "name",
    "children": [
        "name": "name",
        "children": [
            "name": "name",
            "children": [
                ...and so on
            ]
        ]
    ]
}
```

Each 'layer' inside the `children` property represents another layer in the sunburst diagram.  The first layer is the root and contains overarching information about the system under test.  The second layer of `children` nodes each contain a database table with information about that table.  The third layer of `children` contains database column information specific to the parent table and test result information.  Referring to the image shown above, one can see how this JSON structure is rendered.

(*It's worth mentioning that JSON is not the only format d3.js accepts, I just found it to be the easiest for this experiment.*)

The last step is generating the sunburst diagram using the scraped and parsed FitNesse results data.  This ended up being very similar to the [previously mentioned example sunburst diagram by Mike Bostock](https://bl.ocks.org/mbostock/4348373).  Here's an simplified example of what that may look like:

```js
svg.selectAll("path") //get all the path elements in the SVG.  At thsi point it's empty so there will only be one.  https://github.com/d3/d3-selection
    .data(nodes) //nodes is the parsed FitNesse results JSON data.  Give the data to d3 and begin the process of generating the SVG.
    .enter().append("path") //iterates nodes, creating a new path element for each.  https://github.com/d3/d3-selection#selection_enter
    .on("click", click) //wire events for each create path element that contains a node.  Click will cause the diagram to zoom in on the clicked node
    .on("mouseover", mouseover) //show info about the cell in question
    .on("mouseleave", mouseleave) //clear shown info
    .attr("d", arc) //determine the measurements of the generated path for this node
    .style("fill", function(d){
        if(d.data.passed != null){ //set the color of the arc based on whether or not the node is a test and passed/fail or the node is another non-test piece of data like a db table
            if(d.data.passed = true){
                return testPassedColor;
            } else{
                return testFailedColor;
            }
        }
        if(d.data.name === 'root'){
            return rootColor;
        }
        return colors((d.children ? d : d.parent).data.name); //ensure unique colors are used for tables
    })
```

Here's how it all comes together: 
<br/>
<br/>
<img src="/assets/images/posts/enhancing-old-tests/example.gif" style="width:800px;"/>

The dashboard pieces shown in the gif like the breadcrumb trail and combobox are designed to additionally ease the process of finding the info readers need.  As a result, readers can delve directly into the results instead of having to scroll through them.  The comboboxes used are from [select2.js](https://github.com/select2/select2) and contain autocomplete functionality so users can start typing the name of a database table or column they would like to see and filter results as needed.

### Results

All of this is aimed towards making test results easier to digest for consumers.  Ultimately, readability is one of the most important features of test suites.  If users have a hard time analyzing test results they will start to ignore them as it becomes too tedious to work through them over time.

### Expansion

FitNesse uses a fairly generic structure for test tables, so resuing the code to create JSON from results on other test results would theoretically be straightforward.

By way of a disclaimer, I am in no way a FitNesse expert.  In the event of a more efficient, alternative method existing that would allow us to obtain the raw data from the FitNesse wiki test results tables as JSON then that should be used to eliminate a step in the process.  I did come across a url parameter that can be used to return the test results as XML, but it appeared to return the test results inside a 'table' of sorts rather than just containing the raw data.

It's also worth noting that while this proof of concept code is geared towards creating a sunburst diagram, any type of visualization is possible.  If another type of diagram is desired, I highly recommend taking a look at the [d3.js gallery](https://github.com/d3/d3/wiki/gallery) to get ideas.