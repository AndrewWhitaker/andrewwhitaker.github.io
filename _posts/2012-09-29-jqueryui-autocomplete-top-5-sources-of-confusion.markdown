---
author: andrew
comments: true
date: 2012-09-29 18:31:58+00:00
layout: post
slug: jqueryui-autocomplete-top-5-sources-of-confusion
title: 'jQueryUI Autocomplete: Top 5 sources of confusion'
wordpress_id: 151
categories:
- JavaScript
- jQuery
- jQueryUI
- Programming
---

I've been answering [jQueryUI autocomplete](http://www.jqueryui.com/demos/autocomplete) questions on StackOverflow now for over two and a half years and I've noticed that there are a few things that are _always_ coming up. This post will attempt to clear up those sources of confusion.
<!-- more -->
## jQuery Autocomplete vs. jQueryUI autocomplete 
        
[jQuery autocomplete](http://bassistance.de/jquery-plugins/jquery-plugin-autocomplete/) is jQueryUI autocomplete's predecessor. Despite the multiple messages and disclaimers on the legacy plugin's page, there is _still_ confusion about the documentation, functionality, and status of this plugin. For some reason, some folks do not notice the following message on Jörn Zaefferer's documentation page for the plugin:
            
> This plugin is deprecated and not developed anymore. Its [successor is part of jQuery UI](http://jqueryui.com/demos/autocomplete/), and [this migration guide](http://www.learningjquery.com/2010/06/autocomplete-migration-guide) explains how to get from this plugin to the new one. This page will remain as it is for reference, but won’t be updated anymore.


The link to the migration guide explains how to migrate your old code using the original, deprecated plugin with the new one provided by jQueryUI.
        
## What format does my data need to be in?
        
Another source of confusion is exactly _what_ format does the widget's data source need to take? This is easy to see after viewing a few examples and reading the overview tab on the documentation page.

To summarize, the data that you send to the widget needs to be:
        
1. An array of strings
2. An array of objects. Each object should have a either a `label` property, a `value` property, or both.
   
An important point on #2 is that the object can contain other properties (besides `label` or `value`). This comes in handy when doing custom things with data.
    
## I am using a server-side source and my data isn't being filtered!

When you use a server-side resource, _you_ are responsible for doing the filtering. Usually this occurs by building up a database query based on the term that the user searched for.
    
If you don't want to filter your data with server-side code (which I highly recommend), you could retrieve all of the possible results via AJAX and then let the widget do the filtering. Here's an example using StackOverflow's API:

``` javascript Letting jQueryUI do the filtering http://jsfiddle.net/andrewwhitaker/ZBmM8/light/
$.ajax({
    url: "http://api.stackexchange.com/2.1/users",
    data: {
        pagesize: 100,
        order: 'desc',
        sort: 'reputation',
        site: 'stackoverflow'
    },
    dataType: "jsonp"
}).success(function (data) {
    console.dir(data);
    var source = $.map(data.items, function (user) {
        return user.display_name;
    });
    
    $("input").autocomplete({
        source: source
    });
});
```

This may not be ideal depending on the size of the data you are processing. You may want to handle the filtering on the server rather than bogging the browser down with filtering through thousands of results.
        
## I'm using a server-side resource that isn't returning data in the format that autocomplete is expecting. What should I do?
        
You can supply a callback function to the `source` parameter of autocomplete. This allows you to use virtually any source as long as you format it correctly before passing it to the `response` function that autocomplete uses to populate the results. Here's another example using the StackOverflow API:

``` javascript Using a function with the "source" option http://jsfiddle.net/andrewwhitaker/MGTKm/
$("input").autocomplete({
    source: function(request, response) {
        $.ajax({
            url: "http://api.stackexchange.com/2.1/users",
            data: {
                pagesize: 10,
                order: 'desc',
                sort: 'reputation',
                site: 'stackoverflow',
                inname: request.term
            },
            dataType: 'jsonp'
        }).success(function(data) {
            var results = $.map(data.items, function(user) {
                return user.display_name;
            });

            response(results);
        });
    }
});
```

The key here is to use `$.map` to transform the results into the format that the widget expects (described above).
        
## I want to implement tagging functionality. How can I go about that?
        
This is the most complex of the 5, but it is doable. Check out the [multiple values demo](http://jqueryui.com/demos/autocomplete/#multiple) for one way to do this.
        
I've demonstrated a more complex Google-plus like functionality in the answer to [this question](http://stackoverflow.com/q/7089406/497356). Here's an updated fiddle using the 2.1 version of the API.

``` javascript Tagging functionality http://jsfiddle.net/andrewwhitaker/LHNky/41/light/
function split(val) {
    return val.split(/@\s*/);
}

function extractLast(term) {
    return split(term).pop();
}

function getTags(term, callback) {
    $.ajax({
        url: "http://api.stackexchange.com/2.1/tags",
        data: {
            inname: term,
            pagesize: 5,
            order: 'desc',
            sort: 'popular',
            site: 'stackoverflow'
        },
        success: callback,
        dataType: "jsonp"
    });    
}

$(document).ready(function() {
    $("#tags")
    // don't navigate away from the field on tab when selecting an item
    .bind("keydown", function(event) {
        if (event.keyCode === $.ui.keyCode.TAB && $(this).data("autocomplete").menu.active) {

            event.preventDefault();
        }
    }).autocomplete({
        source: function(request, response) {
            if (request.term.indexOf("@") >= 0) {
                $("#loading").show();
                getTags(extractLast(request.term), function(data) {
                    response($.map(data.items, function(el) {
                        return {
                            value: el.name,
                            count: el.count
                        };
                    }));
                    $("#loading").hide();                    
                });
            }
        },
        focus: function() {
            // prevent value inserted on focus
            return false;
        },
        select: function(event, ui) {
            var terms = split(this.value);
            // remove the current input
            terms.pop();
            // add the selected item
            terms.push(ui.item.value);
            // add placeholder to get the comma-and-space at the end
            terms.push("");
            this.value = terms.join("");
            return false;
        }
    }).data("autocomplete")._renderItem = function(ul, item) {
        return $("<li>")
            .data("item.autocomplete", item)
            .append("<a>" + item.label + "&nbsp;<span class='count'>(" + item.count + ")</span></a>")
            .appendTo(ul);
    };
});
```

This demo is also useful because it shows a custom display of each tag including the count. Type @j to see tags starting with "j".
        
jQueryUI autocomplete's API may look simple at first glance, but this is a very extensible and capable plugin. Most people's questions revolve around the `source` parameter for the plugin. Remember to look at your AJAX requests in Firebug or similar just to make sure the data you're supplying to the widget is what you expect.
