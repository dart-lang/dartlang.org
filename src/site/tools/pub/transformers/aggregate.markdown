---
layout: default
title: "Writing an Aggregate Transformer"
description: How to write a Pub transformer to processes multiple input assets.
header:
  css: ["transformers.css"]
---

{% include toc.html %}
{% include breadcrumbs.html %}

# {{ page.title }}

<aside class="alert alert-info" markdown="1">
**Note:** Aggregate transformers are coming in 1.6, but are available
now in the [Dev Channel download](https://www.dartlang.org/tools/download.html).
</aside>

An aggregate transformer processes multiple assets in a single
pass&ndash;for example, collaging multiple images into a single image.

Most of the steps for writing an aggregate transformer are the same
as for writing a normal transformer. This document
only describes the differences specific to aggregate transformers.
If you aren't familiar with how to write a normal Pub tranformer, see
[Writing a Pub Transformer](/tools/pub/transformers).

This page uses the aggregate_transformer example which you can find
through [Writing a Pub Transformer: Examples](examples/).

## Implementing an aggregate transformer {#implementing-transformer}

An aggregate transformer extends the Dart class, [AggregateTransform][],
from the [barback][] package.

[AggregateTransform]: https://api.dartlang.org/apidocs/channels/stable/dartdoc-viewer/barback/barback.AggregateTransform
[barback]: http://pub.dartlang.org/packages/barback

### Extend `AggregateTransformer` {#extend-transformer}

In the Dart file with your transformer subclass,
extend the `AggregateTransformer` class from the barback package:

{% prettify dart %}
class MyTransformer extends AggregateTransformer { ... }
{% endprettify %}

### Claim input assets {#claim-input-assets}

An aggregate transformer claims its input assets by implementing
the `classifyPrimary` method. For an asset that you want to process,
return a value, or key. For assets that you do not care about,
return null. Pub calls the `classifyPrimary` method on every
potential input asset. Assign the same key to all assets that
you want to be processed together; these assets will be
available in the `apply` method through the `AggregateTransform`
parameter.

The following example, from aggregate_transformer,
only accepts assets whose filename ends with the string `.recipe.html`
For assets that it wants to process,
it returns the path of the source directory, which is where
it wants the output asset to be placed.

{% prettify dart %}
classifyPrimary(AssetId id) {
    // Only process assets where the filename ends with ".recipe.html".
    if (!id.path.endsWith('.recipe.html')) return null;

    // Return the path string, minus the recipe itself.
    // This is where the output asset will be written.
    return p.url.dirname(id.path);
}
{% endprettify %}

### Process input assets {#process-input-assets}

To process assets, implement the `apply()` method.
In this method, you access all of the relevant assets
through the passed-in AggregateTransform, manipulate them,
and write out the new asset.

To request a handle to all input assets, you can use
the `primaryInputs` property in AggregateTransformer.
This provides a stream of type `Asset`. Note that the assets
returned from primaryInputs have no guaranteed order and might
change each time the transformer is run, so
you might need to re-order the assets before processing.

Because the inputs are provided asynchronously,
you must write your code carefully. Functions that
return a Future must be chained together and, before creating
the output asset, use `Future.wait` to ensure that
all assets have been processed.

The following `apply` method, from AggregateTransformer,
reads the recipes, sorts them alphabetically according to
the assets' ID, and creates an output asset with
the recipes compiled into a complete HTML file.

{% prettify dart %}
Future apply(AggregateTransform transform) {
  var buffer = new StringBuffer()..write('<html><body>');

  return transform.primaryInputs.toList().then((assets) {
    // The order of [transform.primaryInputs] is not guaranteed
    // to be stable across multiple runs of the transformer.
    // Therefore, alphabetically sort the assets by ID string.
    assets.sort((x, y) => x.id.compareTo(y.id));
    return Future.wait(assets.map((asset) {
      return asset.readAsString().then((content) {
        buffer.write(content);
        buffer.write('<hr>');
      });
    }));
  }).then((_) {
    buffer.write('</body></html>');
    // Write the output back to the same directory as the source files,
    // into a file named recipes.html.
    var id = new AssetId(transform.package,
                         transform.key + "recipes.html");
    transform.addOutput(new Asset.fromString(id, buffer.toString()));
  });
}
{% endprettify %}

If you wish to request a specific secondary input, you can use the
`getInput` or `readInput` methods.

## More information {#more-info}

* [Writing a Pub Transformer](/tools/pub/transformers/)
: How to write a Pub transformer that accepts a single primary input.
* [Writing a Pub Transformer: Examples](examples/)
: Examples to get you started.
* [barback library](https://api.dartlang.org/apidocs/channels/stable/dartdoc-viewer/barback)
: API docs for the barback package.
* [Barback - Can We Build It? Yes, We Can!](https://docs.google.com/a/google.com/document/d/1juHkCRg-1YH6LvwhGPHgF2ihX-UQtR1fv-8aknO7t_4/edit?pli=1#)
: A description of the barback asset system, written by a
member of the Dart engineering team. 
