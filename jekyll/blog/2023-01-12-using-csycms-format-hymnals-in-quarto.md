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

## 4. Remove non-alphanumeric characters from filenames
```bash
for i in {1..2}; do # for some reason we need to loop as it sometime misses to rename all files
    yq e   '. | keys' hymnals.yaml | while read -r linei ; do     # get all keys(hymnal shortnames)
        hymnal=$(echo $linei |cut -b 3-) # remove the leading dash and whitespace (- blog -> blog)
        hymnal=$(yq e ".$hymnal.link"  hymnals.yaml)
        find "./$hymnal" -type d -wholename "*[^[:alnum:]+._/-]*"  | sort -r | while read -r line ; do 
            echo $line
            reversed=$( echo $line|rev) # use reverse so we can replace the last occurence as the first
            newPath=$( echo $reversed | sed -e 's|[^[:alnum:]+._/-]||g' |rev ) 
            # lastDir=$(echo $newPath|rev |sed -e 's|^[^/]*/||g' |rev ) 
            # echo $lastDir
            # mkdir -p "$lastDir"
            mv "$line" "$newPath"
        done
    done
done
```

## 5. Replace some values
The `Navigation` field in the jekyll files cannot be processed by quarto. So we need to change these to `toc`. Find recursive and replace.

Replace also first occurence of `title:` with `pagetitle`. This will prevent the creation of a title and description element int he body of document.

```bash
# find . -name "*.md" | xargs sed -i "s/^Navigation:/toc:/gi"
# find . -name "*.md" | xargs sed -i "s/^title:/pagetitle:/i"
# find . -name "*.qmd" | xargs sed -i "s/^Navigation:/toc:/gi"
yq e   '. | keys' hymnals.yaml | while read -r linei ; do     # get all keys(hymnal shortnames)
    hymnal=$(echo $linei |cut -b 3-) # remove the leading dash and whitespace (- blog -> blog)
    hymnal=$(yq e ".$hymnal.link"  hymnals.yaml)
    echo $hymnal
    find "./$hymnal"  -wholename "*.md*"| while read -r file ; do 
            echo $file
            sed -i "s/^Navigation:/toc:/gi" $file
            sed -i "s/^title:/pagetitle:/i" $file
    done
    find "./$hymnal"  -wholename "*.qmd*"| while read -r file ; do 
            sed -i "s/^Navigation:/toc:/gi" $file
            sed -i "s/^title:/pagetitle:/i" $file
    done
done
```


## 6. Remove date from jekyll file paths.

We can't use regex in the name to match `\.q?md`. Since we have only one optional character, we can use or.

```bash
find . -wholename "*/[0-9][0-9][0-9][0-9]\-[0-9][0-9]\-[0-9][0-9]-*.qmd" -or -wholename "*/[0-9][0-9][0-9][0-9]\-[0-9][0-9]\-[0-9][0-9]-*.md" | while read -r line ; do 
    newPath=$( echo $line | sed -e 's|/[0-9][0-9][0-9][0-9]\-[0-9][0-9]\-[0-9][0-9]-|/|g' )
    echo $newPath
    mv "$line" "$newPath"
done
```


## 7. Rename csycms files

We start by removing the directory numbers from the paths. The strategy is to rename the directories from those deepest in the hierarchy. That's the use of the `sort` command. This way all the directories that need to be renamed will still exist by the time the loop gets to them.

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
## 8. Replace preface with index
Move `hymnal/index.md` to `hymnal/preface.md` and `hymnal/indices/index.md` to `hymnal/index.md`. We will use tags for the other indices.

```bash
yq e   '. | keys' hymnals.yaml | while read -r line ; do     # get all keys(hymnal shortnames)
    hymnal=$(echo $line |cut -b 3-) # remove the leading dash and whitespace (- blog -> blog)
    hymnal=$(yq e ".$hymnal.link"  hymnals.yaml)
    if [ -d "$hymnal/indices" ]; then # if its a hymnal, (with indices dir)
        mv "$hymnal/index.md" "$hymnal/preface.md"
        mv "$hymnal/indices/index.md" "$hymnal/index.md"
        rm -rf "$hymnal/indices"
    fi
   
done
```

## 9. Generate Sidebar Navigation Links
We group the hynmals by year in descending order. Non hymnal collections such as blog sit by themselves at the bottom. The generated toc is inserted after the last item under contents in `_quarto.yml`

```bash
echo "" > toc.txt
## Sort the hymnals by year
yq e   '. | keys' hymnals.yaml | while read -r line ; do     # get all keys(hymnal shortnames)
    hymnal=$(echo $line |cut -b 3-) # remove the leading dash and whitespace (- blog -> blog)
    link=$(yq e ".$hymnal.link"  hymnals.yaml)
    siteName=$(yq e ".$hymnal.siteName"  hymnals.yaml)
    if [ "$siteName" == "null" ]; then
        siteName=$(yq e ".$hymnal.name"  hymnals.yaml)
    fi
    if [ "$siteName" != "null" ]; then
        if [ "$link" != "null" ]; then
            year=$(yq e ".$hymnal.year"  hymnals.yaml)
            type=$(yq e ".$hymnal.type"  hymnals.yaml)
            if [ "$type" == "null" ]; then
                type="hymnal"
            fi
            if [ "$type" == "hymnal" ]; then
                echo "$year$hymnal" >> toc.txt
            fi
        fi
    fi
done

sort -r toc.txt -o toc1.txt && echo "">toc.txt

## remove year from hymnal short names
sed -i 's/^[0-9][0-9][0-9][0-9]//' toc1.txt

for hymnal in $(cat toc1.txt); do
    link=$(yq e ".$hymnal.link"  hymnals.yaml)
    siteName=$(yq e ".$hymnal.siteName"  hymnals.yaml)
    if [ "$siteName" == "null" ]; then
        siteName=$(yq e ".$hymnal.name"  hymnals.yaml)
    fi
    echo "      - href: $link" >> toc.txt
    echo "        text: $siteName" >> toc.txt
done

## remove blank lines
sed -i '/^$/d' toc.txt 
## insert into toc
sed -i '/^format:/e cat toc.txt' _quarto.yml

## do for blogs
echo "" > toc.txt
## Sort the hymnals by year
yq e   '. | keys' hymnals.yaml | while read -r line ; do     # get all keys(hymnal shortnames)
    hymnal=$(echo $line |cut -b 3-) # remove the leading dash and whitespace (- blog -> blog)
    link=$(yq e ".$hymnal.link"  hymnals.yaml)
    siteName=$(yq e ".$hymnal.siteName"  hymnals.yaml)
    if [ "$siteName" == "null" ]; then
        siteName=$(yq e ".$hymnal.name"  hymnals.yaml)
    fi
    if [ "$siteName" != "null" ]; then
        if [ "$link" != "null" ]; then
            year=$(yq e ".$hymnal.year"  hymnals.yaml)
            type=$(yq e ".$hymnal.type"  hymnals.yaml)
            if [ "$type" == "blog" ]; then
                echo "$year$hymnal" >> toc.txt
            fi
        fi
    fi
done

sort -r toc.txt -o toc1.txt && echo "">toc.txt

## remove year from hymnal short names
sed -i 's/^[0-9][0-9][0-9][0-9]//' toc1.txt

for hymnal in $(cat toc1.txt); do
    link=$(yq e ".$hymnal.link"  hymnals.yaml)
    siteName=$(yq e ".$hymnal.siteName"  hymnals.yaml)
    if [ "$siteName" == "null" ]; then
        siteName=$(yq e ".$hymnal.name"  hymnals.yaml)
    fi
    echo "      - href: $link" >> toc.txt
    echo "        text: $siteName" >> toc.txt
done

## remove blank lines
sed -i '/^$/d' toc.txt 
## insert into toc
sed -i '/^format:/e cat toc.txt' _quarto.yml
echo "" > toc.txt
## Sort the hymnals by year
yq e   '. | keys' hymnals.yaml | while read -r line ; do     # get all keys(hymnal shortnames)
    hymnal=$(echo $line |cut -b 3-) # remove the leading dash and whitespace (- blog -> blog)
    link=$(yq e ".$hymnal.link"  hymnals.yaml)
    siteName=$(yq e ".$hymnal.siteName"  hymnals.yaml)
    if [ "$siteName" == "null" ]; then
        siteName=$(yq e ".$hymnal.name"  hymnals.yaml)
    fi
    if [ "$siteName" != "null" ]; then
        if [ "$link" != "null" ]; then
            year=$(yq e ".$hymnal.year"  hymnals.yaml)
            type=$(yq e ".$hymnal.type"  hymnals.yaml)
            if [ "$type" == "null" ]; then
                type="hymnal"
            fi
            if [ "$type" == "hymnal" ]; then
                echo "$year$hymnal" >> toc.txt
            fi
        fi
    fi
done

sort -r toc.txt -o toc1.txt && echo "">toc.txt

## remove year from hymnal short names
sed -i 's/^[0-9][0-9][0-9][0-9]//' toc1.txt

for hymnal in $(cat toc1.txt); do
    link=$(yq e ".$hymnal.link"  hymnals.yaml)
    siteName=$(yq e ".$hymnal.siteName"  hymnals.yaml)
    if [ "$siteName" == "null" ]; then
        siteName=$(yq e ".$hymnal.name"  hymnals.yaml)
    fi
    echo "      - href: $link" >> toc.txt
    echo "        text: $siteName" >> toc.txt
done

## remove blank lines
sed -i '/^$/d' toc.txt 
## insert into toc
sed -i '/^format:/e cat toc.txt' _quarto.yml

## do for blogs
echo "" > toc.txt
## Sort the hymnals by year
yq e   '. | keys' hymnals.yaml | while read -r line ; do     # get all keys(hymnal shortnames)
    hymnal=$(echo $line |cut -b 3-) # remove the leading dash and whitespace (- blog -> blog)
    link=$(yq e ".$hymnal.link"  hymnals.yaml)
    siteName=$(yq e ".$hymnal.siteName"  hymnals.yaml)
    if [ "$siteName" == "null" ]; then
        siteName=$(yq e ".$hymnal.name"  hymnals.yaml)
    fi
    if [ "$siteName" != "null" ]; then
        if [ "$link" != "null" ]; then
            year=$(yq e ".$hymnal.year"  hymnals.yaml)
            type=$(yq e ".$hymnal.type"  hymnals.yaml)
            if [ "$type" == "blog" ]; then
                echo "$year$hymnal" >> toc.txt
            fi
        fi
    fi
done

sort -r toc.txt -o toc1.txt && echo "">toc.txt

## remove year from hymnal short names
sed -i 's/^[0-9][0-9][0-9][0-9]//' toc1.txt

for hymnal in $(cat toc1.txt); do
    link=$(yq e ".$hymnal.link"  hymnals.yaml)
    siteName=$(yq e ".$hymnal.siteName"  hymnals.yaml)
    if [ "$siteName" == "null" ]; then
        siteName=$(yq e ".$hymnal.name"  hymnals.yaml)
    fi
    echo "      - href: $link" >> toc.txt
    echo "        text: $siteName" >> toc.txt
done

## remove blank lines
sed -i '/^$/d' toc.txt 
## insert into toc
sed -i '/^format:/e cat toc.txt' _quarto.yml
rm toc.*
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
