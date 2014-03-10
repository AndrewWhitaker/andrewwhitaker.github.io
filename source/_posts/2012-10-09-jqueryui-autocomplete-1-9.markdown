---
author: andrew
comments: true
date: 2012-10-09 01:37:33+00:00
layout: post
slug: jqueryui-autocomplete-1-9
title: jQueryUI Autocomplete 1.9
wordpress_id: 164
categories:
- JavaScript
- jQuery
- jQueryUI
- Programming
---

As you might have realized from my previous post, I have an affinity for the jQueryUI autocomplete widget. With 1.9, which was [recently released](http://jqueryui.com/changelog/1.9.0/), autocomplete [got a little love](http://jqueryui.com/upgrade-guide/1.9/#autocomplete), which is what I'll focus on in this post. I'll go over each change and what practical implications it has.

## The `response` event

Previously, it wasn't possible to determine when a search had completed unless results were returned (in which case the `open` event was triggered). There are [several ways to get around this limitation](http://stackoverflow.com/a/4719848/497356), but none of them use autocomplete's API. This limitation made it hard to perform actions if the search returned zero results.  

**Here's what you would have had to do in jQueryUI 1.8:**

``` javascript Detecting no results in jQueryUI 1.8 http://jsfiddle.net/qz29K/
    var availableTags = [/* array of items */];

    $("#auto").autocomplete({
        source: function (request, response) {
            var results = $.ui.autocomplete.filter(availableTags, request.term);
            
            if (!results.length) {
                $("#no-results").text("No results found!");
            } else {
                $("#no-results").empty();
            }
            
            response(results);
        }
    }); 
```

**Here's what you can do in 1.9:**

``` javascript Detecting no results in jQueryUI 1.9 http://jsfiddle.net/andrewwhitaker/AYRhE/
    var availableTags = [/* array of items*/];

    $("#auto").autocomplete({
        source: availableTags,
        response: function(event, ui) {
            if (!ui.content.length) {
                $("#no-results").text("No results found!");
            } else {
                $("#no-results").empty();
            }
        }
    });
```

Much cleaner. This works for an autocomplete-enabled input with a remote source as well:

``` javascript Detecting no results with a remote source http://jsfiddle.net/andrewwhitaker/J5rVP/1/
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
            }).done(function(data) {
                if (data.items) {
                    response($.map(data.items, function(item) {
                        return item.display_name;
                    }));
                } else {
                    response([]);
                }
            });
        },
        delay: 500,
        minLength: 3,
        response: function(event, ui) {
            if (!ui.content.length) {
                $("#message").text("No results found");
            } else {
                $("#message").empty();
            }
        }
    });
```

## Synchronous `change` event

This is fixing a subtle, but important, limitation in 1.8's autocomplete implementation. The `change` event used a small timeout right after the `blur` occurred. Most of the time this didn't cause a problem, but if you wanted to validate that the user selected an item from the suggestion menu, the user could actually submit the form before the `change` event fired. This is best seen with an example (click the **result** tab inside the fiddle):

<iframe style="width: 100%; height: 300px" src="http://jsfiddle.net/andrewwhitaker/qz29K/83/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>
        
If you select an item from the menu after searching, then click out of the field (enabling the submit button), then focus the input field again and change the input's value to something _not_ in the suggestion list, you can submit the form.

In 1.9, this works much better and you can prevent the user from submitting the form entirely:

<iframe style="width: 100%; height: 300px" src="http://jsfiddle.net/andrewwhitaker/AYRhE/1/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>
        
Try following the steps for the 1.8 example, and you should not be able to submit the form.

## Support for `contentEditable`
        

This enhancement allows you to attach autocomplete to a `contentEditable` element. This functionality was not possible in 1.8. This has some very cool applications that I'll explore in a later blog post, but here's a simple example:

<iframe style="width: 100%; height: 300px" src="http://jsfiddle.net/andrewwhitaker/J5rVP/2/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>   

## Blurring a suggestion no longer changes the input's value
        
This one is hard to explain, but follow the steps outlined in the 1.8 fiddle below to see the problem:

<iframe style="width: 100%; height: 300px" src="http://jsfiddle.net/andrewwhitaker/qz29K/84/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>
        
Now, follow the same instructions in 1.9:

<iframe style="width: 100%; height: 300px" src="http://jsfiddle.net/andrewwhitaker/AYRhE/2/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

See the difference? In 1.9, the input's value is not reset when the menu item is hovered over.
    
## Added experimental `messages` option for accessibility
        
The jQueryUI folks explain this one better than I can show with an example:

> We now use ARIA live regions to announce when results become available and how to navigate through the list of suggestions. The announcements can be configured via the messages option, which has two properties: noResults for when no items are returned and results for when at least one item is returned. In general, you would only need to change these options if you want the string to be written in a different language. The messages option is subject to change in future versions while we work on a full solution for string manipulation and internationalization across all plugins. If you're interested in the messages option, we encourage you to just read the source; the relevant code is at the very bottom of the autocomplete plugin and is only a few lines.

I'm not an accessibility expert, so I had to look up what _ARIA live regions are_. [MDN has a great explanation](https://developer.mozilla.org/en-US/docs/Accessibility/ARIA/ARIA_Live_Regions):

> In the past, a web page change could only be spoken in entirety which often annoyed a user, or by speaking very little to nothing, making some or all information inaccessible. Until recently, screen readers have not been able to improve this because no standardized markup existed to alert the screen reader to a change. ARIA live regions fill this gap and provide suggestions to screen readers regarding whether and how to interrupt users with a change.

So how does this apply to the autocomplete widget? Well, now when you search for an item, if you have a screen reader installed it will read you something like _"1 result is available, use up and down arrow keys to navigate."_. Pretty cool, huh?

I've outlined each enhancement to jQueryUI autocomplete in the 1.9 release. There are some exciting possibilities, specifically with the `contentEditable` support and the exposure of the Menu widget as a first-class widget. I'll be sure to follow up on those topics in a subsequent post.
