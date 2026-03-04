---
date: '2026-03-03T23:59:59Z'
title: 'Elementary, My Dear Galaxy: Using LLMs to Profile Your Photo Library'
thumbnail: '/img/galaxy-llm/allinone.png'
author: "Aleš Křenek"
description: "Step by step assembly of Galaxy workflow containing LLM calls"
tags: ["Aleš Křenek", "Galaxy", "LLM", "Workflow"]
colormode: true
draft: false
---

# Step by step assembly of Galaxy workflow containing LLM calls

In this article we go step by step in assembling a Galaxy workflow from LLM calls and several conventional Galaxy tools.
The use case itself is ridiculous, its only purpose (besides making fun) 
is a demonstration how these can work together and what is the added value of the Galaxy environment
in such tasks.


## Galaxy 

> Galaxy is a free, open-source system for analyzing data, authoring workflows, training and education, publishing tools, managing infrastructure, and more. [galaxyproject.org](https://galaxyproject.org/)

And that's it, the quotation of the website summarizes everything important.
Forget about a notebook with many pages of nobody-can-remember-this commands,
and don't be afraid of forgetting how on Earth some file was generated.
At the same time, the powerful computing resources are still at hand.

### E-infra CZ installation: usegalaxy.cz

We provide a [dedicated installation of Galaxy](https://usegalaxy.cz) for the users of e-Infra CZ.
The software is the same as the "global", generally available installations 
at (usegalaxy.org)[https://usegalaxy.org] and (usegalaxy.eu)[https://usegalaxy.eu]
but the registered users get much higher storage and computing quotas.

See also [specific usegalaxy.cz documentation](https://docs.metacentrum.cz/en/docs/graphical/usegalaxy).

### Large language models at usegalaxy.cz

usegalaxy.cz is connected with our [AI as a service](https://docs.cerit.io/en/docs/ai-as-a-service/introduction)
via the LLM Hub tool (see [original anouncement](https://galaxyproject.org/news/2025-10-10-llm-hub/)).

Technically, usegalaxy.cz keeps credentials to talk to the LLM service, sending inputs (prompts) to the selected
LLM, and receiving its outputs. 
By wrapping the calls to a Galaxy tool, the LLM functionality is fully integrated, and it can be used
in manually or automatically run workflows, being combined with thousands of other maintained Galaxy tools.


## Demo workflow

Previous Galaxy experience is not strictly expected but the tutorial is still quite brief to keep it reasonably short. 
If you get lost, refer to [documentation and tutorials](https://galaxyproject.org) 
or ask your favourite LLM for help (they are fairly good in it).

{{< image src="/img/galaxy-llm/login.png" class="rounded float-end ms-3 mb-3 w-25" >}}
[Metacentrum account](https://metavo.metacentrum.cz/en/application/index.html) is required.
Once your registration is approved, proceed to [usegalaxy.cz](https://usegalaxy.cz) 
and click on the e-Infra CZ logo in the upper left corner of the login page.


The workflow described bellow looks complicated a little.
Don't be scared, we make the sequence of manual steps to show them in detail. 
They can be glued together to *Galaxy workflow* and run with a single click, as shown at the end of the section.

<div class="clearfix"></div>

### Classify an image

Let's start with a simple task. One would not need Galaxy for this but it's a necessary baby step
to show how Galaxy wraps the LLM invocations.

1. Pick a favourite photo from your phone and save it as medium sized (800x600) file, preferably in PNG format (the safest one wrt. weird format dialects of JPEG etc.).
1. Click "Upload" button in Galaxy, then "Choose local file", pick the file, and click on "Start". Once the file is uploaded ("100%"), click "Close"
1. Create a file `classes.txt` an fill it with categories you would like the images to be classified to. I used:
    ```
    music
    cat
    flower
    food
    ...
    ```
    and upload it to Galaxy in the same way as the image.
    {{< image src="/img/galaxy-llm/tool.png" class="rounded float-end ms-3 mb-3 w-50" >}}
1. Click "Tools" and search for "LLM Hub", and click on it to get the tool input dialog.
1. Fill in the dialog in a similar way: 
    - Pick "Multimodal models", choose one of them, "qwen3.5" worked for me
    - Pick `classes.txt` as the "Text context" and your photo for "Image Context"
    - Provide suitable promt to tell the model what to do. Be picky to avoid too much LLM creativity, I used:
        ```
        Classify the image using the 
        categories listed in the text input. 
        Output just one line with the class, 
        comma, and your confidence level 
        (float number, 0.0 is "don't know", 
        1.0 is "absolutely sure").
        ```
1. Click "Run Tool". It starts crunching, and depending on the load of our LLM service, 
you get the results in few seconds or minutes.

<div class="clearfix"></div>

### Classify images in a batch 

Galaxy provides specific support for handling multiple (even numerous) files in batches. 

1. Start with the upload dialog again but instead of the default "Regular" tab choose "Collection" now.
1. Click "Choose local file" and select multiple image files now (PNG, 800x600 approximately).
Stick with a dozen or two -- the more you have, the more fun will come, but also the more time you will have to wait. 
1. Click "Start" followed by "Build" (once the latter is enabled)
1. Enter "images" as the list name and click "Create list"
1. Invoke the "LLM Hub" tool again; this time, choose the folder icon for "Image Context" and pick the "images" list as the input.

Depending on the number of files and current LLM load, processing will take a while now, have coffee in the meantime.
The result is a list of single line files again, each corresponding to an image in the input list.

### Concatenate classifications into a single table

1. LLMs tend to output incomplete lines which may break the result contatenation.
This can be fixed by calling the "Text reformating" tool.
Find it in the tool column, provide the output (the whole list) of the LLM Hub (typically, Galaxy names it like "LLM Hub (qwen3.5) on collection XX"), and fill the magic line
    ```
    {print $0}
    ```
    as "AWK Program". 
1. One more hack -- LLM tools annotate their output to be "markdown" type. We need to change it to "csv" for further processing.
Click on the pencil icon ("Edit attributes") with the output of the previous step ("Text reformatting on collection ..."), 
choose "Datatypes" tab, pick "csv" and click on "Save".
1. Search for the "Concatenate datasets" tool, choose "Text reformatting on collection ...", and run. 
It produces one table with the classification results.

### Filter out low-confidence classifications


{{< image src="/img/galaxy-llm/filter.png" class="rounded float-end ms-3 mb-3 w-50" >}}
We asked LLM to express how sure it is about classification of each image.
Now it is time to exclude not so confident ones.

Run the "Filter" tool on the previous input ("Concatenate datasets ...")
specifying the condition of the filter. 
The confidence is the second column in the table, hence "c2".
Choose whatever value you find reasonable with your classified images.

<div class="clearfix"></div>


### Count the results
{{< image src="/img/galaxy-llm/groupby.png" class="rounded float-end ms-3 mb-3 w-50" >}}
Finally, count the number of distinct classifications with the "Datamash" tool.

Use the output of previous step ("Filter on dataset ...") as input, 
set "Group by fields" to "1" (the first column), and enable "Sort input" checkbox.
Leave all other as defaults, in particular "Operation to perform" to be count on column 1, and run.

<div class="clearfix"></div>

### Ask LLM what it thinks about you

{{< image src="/img/galaxy-llm/holmes.png" class="rounded float-end ms-3 mb-3 w-50" >}}
Finally, call LLM again to assess the table with the counts.
In this case, choose "Text models" and "gpt-oss-120b", pick output of the previous step for input,
and provide suitable prompt. Mine was:

```text
I asked a person to provide a random 
selection of photos from his/her phone.
The input are numbers of various objects
in those pictures. Can you hypothesize on
that person character? Use the language
and attitude of Sherlock Holmes.
```

Optionally, "Temperature" parameter among "Advanced Options" can be set above 1.0
to take the LLM high and produce more hallucinating outputs. 

<div class="clearfix"></div>


### Putting it all together

{{< image src="/img/galaxy-llm/history.png" class="rounded float-end ms-3 mb-3 w-25" >}}
In previous steps we demonstrated manual, exploratory approach to data analysis in Galaxy.
This is typical in the initial stage when the user defines the exact processing steps.

Once the procedure settles down, it is desirable to make a reproducible
record of it.
Galaxy is particularly good in it, the is "Extract Workflow" command in the
history menu in the upper right corner of the interface. 
Working with it can be a bit tricky, detailed description is beyond the scope of this article,
refer to the documentation or proceed with trial and error.

My extracted worfklow is [available here](https://usegalaxy.cz/published/workflow?id=ff5044efdb2c4f18).

<div class="clearfix"></div>

## What next?

One could extend the workflow with other image processing tools, there are dozens of them available in Galaxy.
Additional image features can be merged with classification, added as input to the final assessment, getting it 
more accurate.

Or stop playing with this artificial example and head towards serious scientific usege directly. 
We will be happy to provide support at [galaxy@cesnet.cz](mailto:galaxy@cesnet.cz).

It's all up to you, enjoy.


