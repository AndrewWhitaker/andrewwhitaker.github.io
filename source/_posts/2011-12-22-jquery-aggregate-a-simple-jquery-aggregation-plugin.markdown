---
author: andrew
comments: true
date: 2011-12-22 23:39:02+00:00
layout: post
slug: jquery-aggregate-a-simple-jquery-aggregation-plugin
title: 'jQuery.aggregate: A simple jQuery aggregation plugin'
wordpress_id: 130
categories:
- JavaScript
- jQuery
- Programming
---

I've found myself needing to apply an aggregate function over a jQuery object several times. I decided to wrap the functionality in a plugin. I attempted to make `$.aggregate` and its little brother `$.sum` as close to LINQ's [aggregate](http://msdn.microsoft.com/en-us/library/system.linq.enumerable.aggregate.aspx) and [sum](http://msdn.microsoft.com/en-us/library/bb345537.aspx) as possible. This goal obviously couldn't be completely realized because of the dynamic nature of JavaScript. The biggest roadblock there is that you can't really imply a "seed" value for an aggregating operation, since arrays can contain elements of various types in JavaScript.

Let's take a look with an example:

``` javascript
    var letters = "abcdefghijklmnopqrstuvwxyz";


    var message = $.aggregate([0, 6, 6, 17, 4, 6, 0, 19, 4], function (working, element) {
        return working ? working + letters.charAt(element) : letters.charAt(element);
    });

    $("#message").text(message);
```
Here, we're building up a message from an array of numbers that correspond to letters of the alphabet. We supplied the aggregate function with a source and a function describing how we wanted to build the aggregate, based on the current value we're iterating over and the "working" aggregate (what we've built up so far).

The conditional operator inside the aggregate function is kind of awkward, so lets supply an initial "seed" value to the aggregate:

``` javascript
var letters = "abcdefghijklmnopqrstuvwxyz";


var message = $.aggregate([0, 6, 6, 17, 4, 6, 0, 19, 4], '',  function (working, element) {
    return working + letters.charAt(element);
});

$("#message").text(message);
```

Finally, we may want to supply some sort of final transformation function to our aggregated value. With the aggregate plugin, you can supply a function that will transform the aggregated value before returning it:

``` javascript
var letters = "abcdefghijklmnopqrstuvwxyz";


var message = $.aggregate([0, 6, 6, 17, 4, 6, 0, 19, 4], '',  function (working, element) {
    return working + letters.charAt(element);
}, function (value) {
    return value.toUpperCase();
});

$("#message").text(message);
```

That's pretty much it for `$.aggregate`. You should be able to get pretty creative with it.

`$.sum` is just a convenient way to call `$.aggregate`. After implementing `$.aggregate`, `$.sum` was pretty easy. `$.sum` sheds some parameters, but is much more readable if all you're doing is adding some values up:

``` javascript
var total = $.sum([0, 6, 6, 17, 4, 6, 0, 19, 4]);

$("#total").text(total);
```

Much neater right? You can also supply a transformation function that should return the item to be "summed":

``` javascript
var groceries = [
    { name: 'bread', price: 2.50 },
    { name: 'bologna', price: 4.00 },
    { name: 'cheddar cheese', price: 3.50 },
    { name: 'potato chips', price: 3.00 }
];

var total = $.sum(groceries, function () {
    return this.price;
});


$("#total").text("$" + total);
```

That's pretty much it for the "static" functions. Both `$.sum` and `$.aggregate` can take an object _or_ an array of values to aggregate.

There are also "instance" methods on jQuery objects. These methods operate on jQuery objects:

``` javascript
$(document).ready(function () {
    $("td input:text").on("change", function () {
        var total = $("td input:text").sum(function () {
            var quantity = this.value
                , cost = parseFloat($(this).closest("tr").find(".price").text(), 10) || 0;
            
            return quantity * cost;
        });
        
        $("#total").text(total);
    });
});
```

(Aggregate follows the same pattern).

I'm also toying with the idea of adding $.min and $.max.

If you want to download, view the source, or run the unit tests associated with the plugin, [I have it up on Github](https://github.com/AndrewWhitaker/jQuery-Aggregate). Minified version coming soon.
