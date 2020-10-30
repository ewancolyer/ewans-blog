---
layout: post
title:  "Auditing installed chrome extensions"
date:   2020-10-30 11:00:00 +0000
categories: chrome security
---
# About
So, you want to work out what chrome extensions your users have installed, but don't want to use <a target="_blank" href="https://support.google.com/chrome/a/answer/9116814?hl=en">Google's Chrome Cloud Managment</a> (free for 1 admin user, but paid after that), what do you do?

This isn't a sound way to get all extensions installed on a device, it is intended to help guage the impact on moving to a model where you have a blocklist or an allowlist for chrome extensions.

I think it is often underestimated how much a chrome extension can make a employees life easier and going in all guns blazing moving to an allowlist without working out the impact might lead to an <a target="_blank" href="https://youtu.be/w8KQmps-Sog?t=162">uprising</a>, or your employees just trying to avoid these mesures.

# What do we need?

<li> A list of installed extensions on macs</li>
<li> A way to process that list into data that is easy to interpret.</li>


# The extension attribute

Thanks to <a target="_blank" href="https://stackoverflow.com/questions/17377337/where-to-find-extensions-installed-folder-for-google-chrome-on-mac">this</a> post, I found that the default location of installed chrome extensions is in `~/Library/Application\ Support/Google/Chrome/Default`.

So what do we do from here? my first thought is that we can just `ls` the directory, chuck commas between each extension ID and echo it into an extension attribute in your chosen MDM.

Here is how I did it, its not necessarily the neatest, or best way to do it. But its the way that worked for me.

{% highlight zsh %}
#!/bin/zsh

currentUser=`ls -l /dev/console | cut -d " " -f4`
userHome=$( dscl . read /Users/$currentUser NFSHomeDirectory | awk '{print $NF}' )


# checking that files dont exist before starting
if [[ -f /tmp/extensions.txt ]]
then
    rm /tmp/extensions.txt
else
    echo "extensions.txt does not already exist"
fi

if [[ -f /tmp/extensionsNew.txt ]]
then
    rm /tmp/extensionsNew.txt
else
    echo "extensions.txt does not already exist"
fi


# gets names of all files in the extensions dir
if [[ -d $userHome/Library/Application\ Support/Google/Chrome/Default/Extensions ]]
then
    echo "Extensions dir exists"
    
    ls $userHome/Library/Application\ Support/Google/Chrome/Default/Extensions/ > /tmp/extensions.txt 
    tr '\n' ',' < /tmp/extensions.txt > /tmp/extensionsNew.txt
    extensions=$(cat /tmp/extensionsNew.txt)

    echo "<result>"$extensions"</result>"

    rm /tmp/extensions.txt
    rm /tmp/extensionsNew.txt

    exit 0
else
    echo "extensions dir does not exist"
    
    exit 0
fi
{% endhighlight %}

Any suggestions on how the above could be better done are appreciated.

So now we have our list of extensions into our MDM, we have 2 options:

<li> Use their API</li>
<li> Export a CSV</li>

> note, the script that I have written expects only 2 columns and will have to be adapted to expect more than that.

I decided to use a CSV for this, mostly becuase having not used my MDM's API before it would be quicker to import it for this one off task.


# What does the script do?

<li>Loads the CSV and manipulates the data into a json array</li>
<li>Checks if the data exists in the extensionsMapper array, if it does add to a known array, if not add to a unknown one</li>
<li>If the ID already exists in one of the arrays add 1 to the count</li>
<li>Then repeat this for the whole of the origional array</li>
<li>Finally write both arrays to file</li>


The array that is spat out by the csvLoader() will look something like the below which will also be written into the `results/` directory.

{% highlight json %}
["id1","id2","id3","id2","id3","id1"]
{% endhighlight %}

Your `input/extensionsMapper` file should looks like the below, obiously you would duplicate this object for each extension, the script then iterates through this array for every extension and works out whether it should add it to the known or unknown list.

{% highlight json %}
[
    {
        "id":"exampleChromeExtensionID",
        "extensionName":"extensionName",
        "publisher":"nameOfPublisher"
    },
]{% endhighlight %}

If the extension is found in the extensionsMapper array it is referred to as a "Known extension", otherwise a "Unknown extension". Below you will see an example of an object that is in the known list:

{% highlight json %}
[
    {
        "id": "ggjhpefgjjfobnfoldnjipclpcfbgbhl", 
        "extensionName": "My Apps Secure Sign-in Extension", 
        "publisher": "Microsoft", 
        "count": 1
    }
]
]
{% endhighlight %}

and this is one in the Unknown list:

{% highlight json %}
[
    {
        "id": "exampleID", 
        "count": 1
    }
]
{% endhighlight %}

Hopefully the info in these 2 arrays will give you enough info for you to start making an assesment as to the impact of blocking an extension in your enviroment.


# Plans for the future/issues:

I have come across a fair few extensions that can not be found in the chrome store, so I am thinking of adding a "KnowUnknown" array to take these out of the unknown list to make it easier to spot new extensions on devices.

I had a quick go at trying to scrape the chrome store in order to automatically generate the extensionMapper array, but after a quick go it I didn't have any more time to spend on it, I might revisit this in the future at some point as this would save **a lot** of time getting this script boot strapped!

Want to go look at this project, you can find it on <a target="_blank" href="https://github.com/ewancolyer/chrome-extension-tools">GitHub</a>