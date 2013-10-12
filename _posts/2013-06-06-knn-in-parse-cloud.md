---
layout: post
title: "KNN using JavaScript and Underscore"
description: ""
category: 
tags: [parse, underscore, js, ml]
---
{% include JB/setup %}
As users interact with websites, mobile devices, and the Internet of
Things, data about user behavior is being generated at tremendous scale.
And it will only continue to grow. How do we leverage this data?

One useful way to make use of the large amount of data being generated
is to use it to classify new data as it arrives. This is a form of
machine learning. For instance, email or SMS spam filtering or automatically
outputting tags that describe a body of text.

There are many ways to classify data, but one way is to use the "K
Nearest Neighbor" (KNN) algorithm.

## Machine Learning and K Nearest Neighbor

KNN is a machine learning classification algorithm that is lazy. This
means computation is deferred until classification is actually needed.
Further, KNN needs to be supervised in that an initial set of data is
required in order to "train" the algorithm. KNN is a simple,
easy-to-understand algorithm that really doesn't require any prior
knowledge of statistics.

The basic premis of KNN is simplistic and reasonable: given an instance
of data, look for the k closest neighboring instances and choose the
most popular class among them.

Imagine if a piece of data is described by only two attributes. Fruit,
for example, can be described by having weight and color. If many instances
of fruit are plotted on a X-Y graph where each axis represents one of
these attributes, there would be clusters of each fruit type in the 2D
space.

## Underscore and KNN

Because KNN is lazy, it is expensive. Optimizations can be made, but
that is an exersize for later. The psudo-code for KNN is essentially:

    load SAMPLE
    load DATASET
    sort DATASET by DISTANCE from SAMPLE
    load TOP_K from DATASET
    reduce TOP_K to frequency COUNT
    return max of COUNT

Since KNN is essentially a series of operations on collections, I wanted
to write a KNN impelemntation leveraging Underscore as much as possible.

### Distance function

KNN, at its core, needs to calculate the distance of the attributes from
the sample. In other words, triangulation using cartesian distance. In this naive
implementation, the number of attributes needs to be known up front. For
simplicity's sake, we are going to classify using only two attributes.


The distance function is simply:

    var distance = function(item1, item2) {
        a = item1[0] - item2[0];
        b = item1[1] - item2[0];
        return Math.pow((Math.pow(a,2) + Math.pow(b,2)), 0.5);
    };

This is pretty much the only function that is now based on Underscore.

### Sorting the dataset

KNN requires getting the nearest neighbors, thus, we need to sort the
training data by distance from the sample. Underscore provides the
`_.sortBy` function that can return an array where the elements are sorted
by a custom function. In this case, the distance from the sample:

    var sort = function(sample, dataset) {
        return _.sortBy(dataset, function(item) {
            return distance(item, sample);
        });
    };

### The Top K

The K in KNN is the number of closest neighbors we want to classify the
sample against. Underscore provides the `_.first` function:

    var topK = function(dataset, k) {
        return _.first(dataset, k);
    };

### Class Frequency

Next KNN needs to know what the frequency of each of the top K. This is
essentially a reduce function that takes a dataset and needs to count
the elements by a certain attribute. Underscore provides the `_.countBy`
function:

    var classCount = function(dataset) {
        return _.countBy(dataset, function(item) {
            return item[2];
        });
    };

### Most Popular in the Class of...

The last operation of KNN is to get the most frequency, or popular, class. This
single value the algorithm's output... a classification of the sample
based on the training dataset. Underscore provides the `_.max` function.
However, the result of the previous use of the `_.countBy` function is an Object
that maps the class as a key to the number of instances as a value and
`_.max` takes an array. Conveniently, the Underscore `_.pairs` function
converts an Object into an array definition:

    var classify = function(dataset) {
        return _.max(_.pairs(dataset), function(item) {
            return item[1];
        })[0];
    };

## All Together Now

Let's classify fruit across two attributes: weight and color. Weight, of course, is
a number. However, any attribute that is not inherently a number (remember we are
calculating distance here) needs to be mapped. Color nicely maps to a number
because we can create an artificial scale across:

    red    1
    orange 2
    yellow 3
    green  4
    blue   5
    purple 6

We need some sample data to train the algorithm:

    var dataset = [
      [303, 3, "banana"],
      [370, 1, "apple"],
      [298, 3, "banana"],
      [277, 3, "banana"],
      [377, 4, "apple"],
      [299, 3, "banana"],
      [382, 1, "apple"],
      [374, 4, "apple"],
      [303, 4, "banana"],
      [309, 3, "banana"],
      [359, 1, "apple"],
      [366, 1, "apple"],
      [311, 3, "banana"],
      [302, 3, "banana"],
      [373, 4, "apple"],
      [305, 3, "banana"],
      [371, 3, "apple"],
    ];

## Testing KNN

In order to test this, I deployed the following to Parse Cloud:

    var _ = require('underscore');

    Parse.Cloud.define("classify", function(req, res) {

        var sorted_dataset = sort(req.params.input, dataset);
        var top_k = topK(sorted_dataset, 3);
        var counts = classCount(top_k);
        var classification = classify(counts);

        res.success(classification);
    });

    var distance = function(item1, item2) {
        a = item1[0] - item2[0];
        b = item1[1] - item2[1];

        c = Math.pow((Math.pow(a, 2) + Math.pow(b, 2)), 0.5);

        return c;
    };

    var sort = function(unknown_item, dataset) {
        return _.sortBy(dataset, function(item) {
            return distance(item, unknown_item);
        });
    };

    var topK = function(dataset, k) {
        return _.first(dataset, k);
    };

    var classCount = function(dataset) {
        return _.countBy(dataset, function(item) {
            return item[2];
        });
    };

    var classify = function(dataset) {
        return _.max(_.pairs(dataset), function(item) {
            return item[1];
        })[0];
    };

    var dataset = [
      [303, 3, "banana"],
      [370, 1, "apple"],
      [298, 3, "banana"],
      [277, 3, "banana"],
      [377, 4, "apple"],
      [299, 3, "banana"],
      [382, 1, "apple"],
      [374, 4, "apple"],
      [303, 4, "banana"],
      [309, 3, "banana"],
      [359, 1, "apple"],
      [366, 1, "apple"],
      [311, 3, "banana"],
      [302, 3, "banana"],
      [373, 4, "apple"],
      [305, 3, "banana"],
      [371, 3, "apple"],
    ];

The sample that we are trying to classify is sent in an HTTP POST:

    curl -X POST \
        -H "X-Parse-Application-Id: XXX"            \
        -H "X-Parse-REST-API-Key: YYY"              \
        -H "Content-Type: application/json"         \
        -d '{ "input" : [303, 4] }'                 \
        https://api.parse.com/1/functions/classify

This sample is classified as a banana:

    {"result":"banana"}

It is an unriped, green banana, but KNN still classified it as a banana.

## Next Steps

You may have noted that the sample data, weight and color, are
unbalanced. The weight has more weight, if you pardon the pun. A less
naive implementation would normalize all the data before calculating the
distance from the sample.
