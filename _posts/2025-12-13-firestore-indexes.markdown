---
layout: post
title: "Turn off your Firestore automatic indexes"
date: 2025-12-13 17:00:00 +0100
tags: Firestore Software
custom_css: firestore-indexes
---

`query(blogPosts, where("id", "==", "firestore-indexes"))`

Every Firestore query that contains a `where` clause – such as the one above – requires an index. Some database engines would service a query without an index by scanning the entire table. That's slow! Not Firestore. Firestore will refuse the request if the necessary index isn't there.

To make this zero-effort for one-field queries like the one above, Firestore automatically maintains single-field indexes for each field in each document. It does so in both ascending and descending order so results can be returned in either order depending on the query’s `orderBy` clause.

Handy, right?

## Storage Size – An Interactive Example

Trouble is, those single-field index entries add up.

To see just how much, edit the JSON below and see the resulting storage usage for the automatic index entries. Calculations are based on [Firestore – Storage size calculations](https://firebase.google.com/docs/firestore/storage-size) and assume you are using the default indexing rules.

<div id="firestore-vis-container">
  <!-- Snapshot of content for non-JS readers -->
  <div><span class="firestore-index-vis-label">Document path</span><input class="firestore-index-vis-document-path-input" type="text" value="players/VtMAosUN/answers/mipUtFyB"><span class="firestore-index-vis-label">Document content</span><div class="language-js highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="p">{</span>
    <span class="dl">"</span><span class="s2">question</span><span class="dl">"</span><span class="p">:</span> <span class="p">{</span>
      <span class="dl">"</span><span class="s2">type</span><span class="dl">"</span><span class="p">:</span> <span class="dl">"</span><span class="s2">time-matching</span><span class="dl">"</span><span class="p">,</span>
      <span class="dl">"</span><span class="s2">value</span><span class="dl">"</span><span class="p">:</span> <span class="p">{</span> <span class="dl">"</span><span class="s2">hour</span><span class="dl">"</span><span class="p">:</span> <span class="mi">2</span><span class="p">,</span> <span class="dl">"</span><span class="s2">minute</span><span class="dl">"</span><span class="p">:</span> <span class="mi">25</span> <span class="p">}</span>
    <span class="p">},</span>
    <span class="dl">"</span><span class="s2">answer</span><span class="dl">"</span><span class="p">:</span> <span class="p">{</span>
      <span class="dl">"</span><span class="s2">type</span><span class="dl">"</span><span class="p">:</span> <span class="dl">"</span><span class="s2">digital</span><span class="dl">"</span><span class="p">,</span>
      <span class="dl">"</span><span class="s2">digits</span><span class="dl">"</span><span class="p">:</span> <span class="p">[</span><span class="dl">"</span><span class="s2">1</span><span class="dl">"</span><span class="p">,</span> <span class="dl">"</span><span class="s2">4</span><span class="dl">"</span><span class="p">,</span> <span class="dl">"</span><span class="s2">2</span><span class="dl">"</span><span class="p">,</span> <span class="dl">"</span><span class="s2">5</span><span class="dl">"</span><span class="p">]</span>
    <span class="p">},</span>
    <span class="dl">"</span><span class="s2">takenAtDate</span><span class="dl">"</span><span class="p">:</span> <span class="dl">"</span><span class="s2">2025-12-19T15:00:00.000Z</span><span class="dl">"</span>
  <span class="p">}</span>
  </code></pre></div></div><br><div class="firestore-index-vis-breakdown-bar" style="background-color: rgb(225, 225, 225); height: 2em; display: flex; overflow: hidden;"><div class="firestore-index-vis-breakdown-segment" style="background-color: rgb(0, 157, 255); width: 9.44056%; height: 100%;"></div><div class="firestore-index-vis-breakdown-segment" style="background-color: rgb(221, 145, 2); width: 90.5594%; height: 100%;"></div></div><div style="display: flex; align-items: center; gap: 0.5em; cursor: pointer;"><span style="width: 1em; font-size: 0.8em;">▼</span><span class="firestore-index-vis-size-label"><span class="firestore-index-vis-size-label-text" style="color: rgb(0, 157, 255); font-weight: bold;">Total document size</span><span class="firestore-index-vis-size-value">216 B</span></span></div><div style="margin-left: 2em;"><table style="border-collapse: collapse; width: 100%;"><tbody><tr><td>Name</td><td>50 B</td></tr><tr><td>Field data</td><td>134 B</td></tr><tr><td>Additional bytes</td><td>32 B</td></tr></tbody></table></div><div style="display: flex; align-items: center; gap: 0.5em; cursor: pointer;"><span style="width: 1em; font-size: 0.8em;">▼</span><span class="firestore-index-vis-size-label"><span class="firestore-index-vis-size-label-text" style="color: rgb(221, 145, 2); font-weight: bold;">Total index entries size</span><span class="firestore-index-vis-size-value">2,072 B</span></span></div><div style="margin-left: 2em;"><table style="border-collapse: collapse; width: 100%;"><tbody><tr><td><code>answer.digits</code></td><td>556 B</td></tr><tr><td><code>takenAtDate</code></td><td>320 B</td></tr><tr><td><code>question.value.minute</code></td><td>306 B</td></tr><tr><td><code>question.type</code></td><td>302 B</td></tr><tr><td><code>question.value.hour</code></td><td>302 B</td></tr><tr><td><code>answer.type</code></td><td>286 B</td></tr></tbody></table></div></div>
</div>
<script src="/assets/scripts/firestoreIndexVis.global.js"></script>
<script>
    document.addEventListener("DOMContentLoaded", function() {
      const onSizeResult = (sizeResult) => {
        if (!sizeResult) return;
        const extraDataFreeNumber = document.getElementById("extra-data-free-number");
        const multiplier = Math.round(
          sizeResult.indexSize.total / sizeResult.documentSize.total);
        extraDataFreeNumber.textContent = `${multiplier}x`;
      }
      const container = document.getElementById("firestore-vis-container");
      const breakdownContainer = document.getElementById("firestore-index-vis-breakdown-container");
      breakdownContainer.innerHTML = "";
      window.FirestoreIndexVis.renderReactIndexVis(container, breakdownContainer, onSizeResult);
    });
</script>
<style>

</style>

<div class="extra-data-free">
  <span class="extra-data-free-number" id="extra-data-free-number">10x</span>
  <span>extra data</span>
  <span>free<sup>*</sup>!</span>
  <span class="extra-data-free-note">*Storage sold separately</span>
</div>

## Why do index entries take so much storage?

<div id="firestore-index-vis-breakdown-container">
  <!-- Snapshot of content for non-JS readers -->
  <div class="firestore-index-vis-breakdown-bar" style="background-color: rgb(225, 225, 225); height: 2em; display: flex; overflow: hidden;"><div class="firestore-index-vis-breakdown-segment" style="background-color: rgb(221, 145, 2); width: 33.7838%; height: 100%;"></div><div class="firestore-index-vis-breakdown-segment" style="background-color: rgb(221, 107, 0); width: 27.7027%; height: 100%;"></div><div class="firestore-index-vis-breakdown-segment" style="background-color: rgb(202, 7, 7); width: 21.6216%; height: 100%;"></div><div class="firestore-index-vis-breakdown-segment" style="background-color: rgb(128, 0, 128); width: 10.4247%; height: 100%;"></div><div class="firestore-index-vis-breakdown-segment" style="background-color: rgb(0, 0, 128); width: 6.46718%; height: 100%;"></div></div><ul><li><span class="firestore-index-vis-size-label"><span class="firestore-index-vis-size-label-text" style="color: rgb(221, 145, 2); font-weight: bold;">Document Name</span><span class="firestore-index-vis-size-value">700 B</span></span></li><li><span class="firestore-index-vis-size-label"><span class="firestore-index-vis-size-label-text" style="color: rgb(221, 107, 0); font-weight: bold;">Parent Document Name</span><span class="firestore-index-vis-size-value">574 B</span></span></li><li><span class="firestore-index-vis-size-label"><span class="firestore-index-vis-size-label-text" style="color: rgb(202, 7, 7); font-weight: bold;">Additional Bytes</span><span class="firestore-index-vis-size-value">448 B</span></span></li><li><span class="firestore-index-vis-size-label"><span class="firestore-index-vis-size-label-text" style="color: rgb(128, 0, 128); font-weight: bold;">Field Name</span><span class="firestore-index-vis-size-value">216 B</span></span></li><li><span class="firestore-index-vis-size-label"><span class="firestore-index-vis-size-label-text" style="color: rgb(0, 0, 128); font-weight: bold;">Field Value</span><span class="firestore-index-vis-size-value">134 B</span></span></li></ul>
</div>

Each individual index entry can be surprisingly large. Entry sizes are sensitive to the length of the document path since each index entry has to reference the entire document path and the parent document path. Entries also include a non-trivial fixed overhead of 32B. The field value is often an insignificant portion of the index entry size.

And as for _how many_ entries we get for each document, remember it’s often 2 indexes (and therefore 2 index entries) per field: ascending + descending. Fields nested within map fields are indexed too. For array fields, it's an entry per element of the array, another way you can rack up more entries than you might expect.

## "Storage is cheap"

The size of the index entries in your Firestore database is also multiplied into your various backups, which you may want to have plenty of.
For us at [Polymath](https://polymath.how/), Firestore storage (including backups) was accounting for a considerable 34% of our **total** Google Cloud bill.

We were able to chop nearly 29% off our total bill solely by being more discerning with our Firestore indexes. Your mileage, of course, may vary.

## How to turn them off

<aside class="warning">
    Captain Obvious reminds you: <b>DO NOT</b> turn off indexes that are in use in production. Dependent queries will fail until the indexes are rebuilt which can take several hours.
</aside>

Once you've found fields that you're confident you won't be querying on, there are options for how you turn off the automatic indexes. I'll assume you're using a `firestore.indexes.json` file. Read more in [Cloud Firestore Index Definition Reference](https://firebase.google.com/docs/reference/firestore/indexes).

You can turn off automatic indexes for fields one by one by setting the list of indexes to an empty list:

```js
// firestore.indexes.json
{
  "fieldOverrides": [
    {
      // we can only specify collectionGroup, not the collection path
      // so this applies to e.g. both:
      // users/alice/books and libraries/cambridge/books
      "collectionGroup": "books",
      // If the field is a map
      // than nested fields inherit these settings
      "fieldPath": "blurb",
      "ttl": false,
      "indexes": []
    }
  ]
}
```

You can turn them off for all fields within a specific [collection group](https://firebase.google.com/docs/firestore/query-data/queries#collection-group-query) using the `*` wildcard and selectively enable the ones that you need:

```js
// firestore.indexes.json
{
  "fieldOverrides": [
    {
      "collectionGroup": "books",
      "fieldPath": "*",
      "ttl": false,
      "indexes": []
    },
    {
      "collectionGroup": "books",
      "fieldPath": "isbn",
      "ttl": false,
      "indexes": [
        // here we choose to only index in ascending order
        {
          "order": "ASCENDING",
          "queryScope": "COLLECTION"
        }
      ]
    }
  ]
}
```

Unfortunately, you **can’t** turn them off database-wide, which seems to be a deliberate move on Firestore's part. Trying to use the default collection group name `__default__` fails:

```js
// firestore.indexes.json
{
  "fieldOverrides": [
    {
      // "HTTP Error: 400,
      //  Configuring database-level settings is not supported."
      "collectionGroup": "__default__",
      "fieldPath": "*",
      "ttl": false,
      "indexes": []
    }
  ]
}
```

## References

- [Firestore – Storage size calculations](https://firebase.google.com/docs/firestore/storage-size)
- [Firestore – Automatic Indexing](https://firebase.google.com/docs/firestore/query-data/index-overview#automatic_indexing)
- [Firestore – Single-field index exemptions](https://firebase.google.com/docs/firestore/query-data/index-overview#single-field_index_exemptions)
- [Cloud Firestore Index Definition Reference](https://firebase.google.com/docs/reference/firestore/indexes)
- [The code that runs the above calculations and visualisation](https://github.com/jaredkhan/firestore-index-vis/blob/main/src/calculateSize.ts)
