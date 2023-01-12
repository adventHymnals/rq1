---
layout: post
current: post
Navigation: False
title: Using Csycms Format hymnals in Quarto
date: January 12, 2023
tags: [technical, server]
class: post-template
subclass: 'post'
author: Brian Onang'o
# subtitle: "A simple Quarto webpage with a book layout"
page-layout: full
---


[Csycms](https://github.com/csymapp/csycms-cli) is a flat file content management system for nodejs which we developed about 4 to 5 years back in 2018. It has served us well hosting both the sites for [Advent Hymnals](https://adventhymnals.org) as well as the [Sabbath School Lessons Archive](https://sabbathschool.github.io/) before we moved it to github pages. And now we have to retire it. While we could keep improving it to keep up with our current needs, that will be time and resources not wisely spent. We have now fully adopted github pages as our host for advent hymnals, and mainly because it is free. While there are several options available for us to use gh-pages, it seems to us at the moment, especially between `quarto` and `jekyll` that quarto is the better for us of the two. Here we explain how to use the the hymnals in the csycms format they are currently in in quarto. The reason for maintaining that format is so that there is just a single set of documents to easy any editing work that might be done which can be used across a wide range of tools. And the choice of using csycms means that we will also later need to explain how to create the csycms document structure for any new hymnals whichmay need to be added.


## 1. Downloading the hymnals repository
The first step is to download [the hymnals list]() available in a yaml format. The hymnals list contains also links to blogs. The blogs are available in jekyll format instead of csycms.

```
wget -O hymnals.yaml https://raw.githubusercontent.com/adventHymnals/hymnals/master/hymnals.yaml
```

## 2. Installing `yq`
Then we install yq to be able to read yaml files in bash

```bash
sudo wget https://github.com/mikefarah/yq/releases/download/v4.4.1/yq_linux_amd64 -O /usr/bin/yq &&\
sudo chmod +x /usr/bin/yq
```

## 3. Downloading the hymnals
The downloaded hymnal-list has all hymnals that we would like to have at the end of the project. However, not all hymnals are available just yet. These lack a value for `gtLink` and are ignored. All the rest are downloaded into the directory having the name given in `link`.

```bash
yq e   '. | keys' hymnals.yaml | while read -r line ; do     # get all keys(hymnal shortnames)
    hymnal=$(echo $line |cut -b 3-) # remove the leading dash and whitespace (- blog -> blog)
    # check if gtLink is available and clone into $link
    gtLink=$(yq e ".$hymnal.gtLink" hymnals.yaml)
    if [ "$gtLink" != "null" ]; then
        gtLink="https://github.com/$gtLink.git"
        link=$(yq e ".$hymnal.link"  hymnals.yaml)
        rm -rf "$link" && git clone $gtLink $link
    fi
done
```

## 4. Replace some values
The `Navigation` field in the jekyll files cannot be processed by quarto. So we need to change these to `toc`. Find recursive and replace
```bash
find . -name "*.md" | xargs sed -i "s/^Navigation:/toc:/gi"
find . -name "*.qmd" | xargs sed -i "s/^Navigation:/toc:/gi"
```


## 5. Remove date from jekyll file paths.

We can't use regex in the name to match `\.q?md`. Since we have only one optional character, we can use or.

```bash
find . -wholename "*/[0-9][0-9][0-9][0-9]\-[0-9][0-9]\-[0-9][0-9]-*.qmd" -or -wholename "*/[0-9][0-9][0-9][0-9]\-[0-9][0-9]\-[0-9][0-9]-*.md" | while read -r line ; do 
    newPath=$( echo $line | sed -e 's|/[0-9][0-9][0-9][0-9]\-[0-9][0-9]\-[0-9][0-9]-|/|g' )
    mv "$line" "$newPath"
done
```


## 6. Rename csycms files

We start by removing the directory numbers from the paths. 

```bash
find . -wholename "*/[0-9][0-9].*"  | sort -r | while read -r line ; do 
    reversed=$( echo $line|rev) # use reverse so we can replace the last occurence as the first
    newPath=$( echo $reversed | sed -e 's|\.[0-9][0-9]/|/|g' |rev ) 
    
    #last dir
    lastDir=$(echo $newPath|rev |sed -e 's|^[^/]*/||g' |rev ) 
    echo "mv $line $newPath\n$lastDir" >> lines.txt
    mkdir -p "$lastDir"
    mv "$line" "$newPath"
    # if [ -d "$newPath" ]; then
    #     echo "cp -r $line/* $newPath/" >> lines.txt
    #     cp -r "$line/* $newPath/" && rm -rf "$line"
    # else
    #     mv "$line" "$newPath"
    # fi
done
```

Then we rename `chaper.md` to `index.md`

```bash
find . -wholename "*/chapter.md"  | while read -r line ; do 
    newPath=$( echo $line | sed -e 's|/chapter\.md|/index.md|' ) 
    mv "$line" "$newPath"
done
```

Then we rename all `docs.md`
```bash
find . -wholename "*/docs.md"  | while read -r line ; do 
    newPath=$( echo $line | sed -e 's|/\([^\]*\)/docs\.md|/\1.md|' ) 
    mv "$line" "$newPath"
done
```

Then we remove all empty directories
```bash
find . -type d -empty -delete
```

The complete script is available [here](https://raw.githubusercontent.com/adventHymnals/resources/master/scripts/csycmstoquarto.sh).

Add it to the github workflow:
```yaml
- name: Download and process csycms hymnals
        run: |
          wget -O csycmstoquarto.sh https://raw.githubusercontent.com/adventHymnals/resources/master/scripts/csycmstoquarto.sh
          chmod +x csycmstoquarto.sh
          ./csycmstoquarto.sh
```
