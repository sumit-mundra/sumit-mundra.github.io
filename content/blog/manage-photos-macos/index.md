+++
title = 'Declutter Photos on Macos'
date = '2024-12-29T10:03:14+05:30'
draft = false
+++

# Prelude
With advent of fast and high-res camera mobile phones, comes the need to manage your cloud storage. Memories are just a few taps away residing in your pocket, though, stored somewhere on a cloud server (most likely at Google Photos or iCloud Photos).
I have been a fanboy of Google products since childhood. And as I grow old, I am also becoming a fan of Apple products. But, the programmer within me loves everything FOSS(linux anyone?). Darn it, why can't I pick a side. Actually, I do not have to. By now, you might be wondering where am I leading to with all of this? 

We all have enjoyed free backups. We might have enjoyed fast mobile cameras too. Alas, free things don't last. Now, you need to declutter your storage every now and then. And, you need to pay to keep on continuing cloud-based photo managers. My memories are now split between worlds of Apple and Google(Android) because of their incompatible setup. Google Photos has entirely moved away from desktop versions. And, iCloud does not have smart features like Google Photos, plus a lot of my data is in Google Photos already.

## Problem Statement
With this confusing context, the actual problem I am trying to tackle here is to declutter and organise photos, but without help of any cloud provider. This should help in choosing right cloud backup provider and also allow for escaping a vendor lock-in, should you choose to keep self managed backups using the [VPN](../setup-vpn-server-part-1/) approach.

In order to reduce photos, I want to be able to
- find out photos which are very similar to each other or taken within short spans.
- group photos which were related to a life event.
- find out photos which were taken nearby to a given location (latitude, longitude).

## Approach 
I was focusing on first two problems, which made me realise,
- I often click multiple photographs in short time intervals to acheive a "best of 5" or "best of 3".
- I do not have memories spanning days very often. Mostly within an hour long span. 
- Some travel based memories can be grouped by location in order to build albums.

A [DBSCAN](https://en.wikipedia.org/wiki/DBSCAN) (density based scan) algorithm for clustering can provide unsupervised mechanism to build groups (clusters). The two parameters to the algorithm can be thought as
- distance: duration between moments when two photos were taken. So, with "Date Created" attribute, I could acheive a distance equivalent effect. 
- count: The minimum count of points to call a possible group as a cluster. Higher value leads to more dense cluster detections, but often gives larger outliers count too, for less dense points. Smaller value provides smaller clusters on evenly distributed dataset and less number of outliers.

For my usecase, a count of 2 with distance as 2min can build me photographs which were taken together within 2 minutes of each other grouped together. This can then help me reduce clutter and choose which photographs to keep.
Alternatively, if I have larger distance value and larger count, say 1 hour and 5 photos, I shall see groups resembling a travel kind of memory.

## Implementation
To implement this, I went ahead with rust to revise my previous learnings. Also, who does not love a bite of smaller memory and cpu footprint ?. Check out the codebase at [fileman](https://github.com/sumit-mundra/fileman). The command line utility fileman is available for macos target for now

``` text
➜  release git:(main) ✗ ./fileman --help
fileman is a simple tool to group files based on create_date os timestamp using dbscan algorithm. author: Sumit Mundra

Usage: fileman [OPTIONS] --input-path <INPUT_PATH> --target-path <TARGET_PATH>

Options:
  -i, --input-path <INPUT_PATH>
          the directory to scan
  -o, --target-path <TARGET_PATH>
          output directory with links to input files grouped inside cluster-wise folders
  -t, --time-interval-sec <TIME_INTERVAL_SEC>
          duration in seconds between two files to cluster them together [default: 600]
  -c, --min-cluster-size <MIN_CLUSTER_SIZE>
          minimum count of files needed to define a cluster [default: 3]
  -p, --tag-prefix <TAG_PREFIX>
          prefix to be used as tag, no tagging if empty [default: cluster]
  -h, --help
          Print help
  -V, --version
          Print version
➜  release git:(main) ✗
```

To ease out the viewing problems of groups, `fileman` provides two approaches for now, 
1. An output directory with symlinks to original file paths grouped together. Thus, you can visualise images in finder quick preview.
2. A tagging mechanism on original files to provide macos tags. With group by tags, you can then either declutter photos with such params or move files into album folders if needed.

## Results
On my MBP 2016 model(A1707), `fileman` could cluster a couple of day's worth of images (77) in under 20ms.

```
➜  release git:(main) ✗ ./fileman -i /Users/sumitmundra/Downloads/file_input_data -o /Users/sumitmundra/Downloads/out_path 
Finished in 17 ms
```
## What's next
I would add capacity to read multiple filepaths from a file and then allow to add/remove tags independent of clustering part.

> *Fun fact: The fileman currently works on all file types. It can help you sort out your downloads folder in accordance of your browsing activity ;).*