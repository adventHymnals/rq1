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

### Installing `yq`

```bash
sudo wget https://github.com/mikefarah/yq/releases/download/v4.4.1/yq_linux_amd64 -O /usr/bin/yq &&\
sudo chmod +x /usr/bin/yq
```


yq e   '. | keys' hymnals.yaml | while read -r line ; do    
   hymnal=$(echo $line |cut -b 3-) # - blog -> blog
   #echo "==>::$hymnal"
   # check if gtLink is available
   gtLink=$(yq e ".$hymnal.gtLink" hymnals.yaml)
   if [ "$gtLink" != "null" ]; then
    gtLink="https://github.com/$gtLink.git"
   
    link=$(yq e ".$hymnal.link"  hymnals.yaml)
    # clone if hymnal is available
    echo "==>::$hymnal::$link::$gtLink"
    rm -rf "$link" && git clone $gtLink $link
    fi
     #echo "==>::$hymnal$link"
done

## 2. Downloading individual hymnals




But as we work on the migration, there may be need to migrate our csycms system from one server to another. So here are the instructions incase this need should arise.

## Requirements
1. An ubuntu server
2. Node.js >=14.0.0 <18.0.0
3. Nginx. You can use Apache or any other webserver of your choice. But we only have configurations for nginx.
4. certbot

One you have set up these requirements, you are good to go. If you are not sure how to go about it you can check out the instructions elsewhere in the internet.

## Installation
First install csycms: 
```bash
npm install -g csycms
```

Then initialize it and create service files:
```bash
sudo csycms init
```

Then install advent hymnals site:
```bash
csycms site --create -n adventhymnals -p 8710 -r https://github.com/GospelSounders/adventhymnals.git -d adventhymnals.org
```

Then make changes to configurations in `/etc/csycms/sites-enabled/adventhymnals.yml`. A working configuration is found [here](https://github.com/adventHymnals/resources/blob/master/configurations/adventhymnals.yml)
```
sudo /etc/csycms/sites-enabled/adventhymnals.yml
```

Be sure to change:
- domain. Assets will not be loaded if this is wrong
- scheme. If this is wrong, assets will not be loaded properly.
- site.space
- site.title
- copyright.title
- copyright.url
- copyright.name

Now its time to setup nginx to act as a proxy to our csycms server. Sample configuration is found [here](https://github.com/adventHymnals/resources/blob/master/configurations/adventhymnals-nginxconfiguration). You can have it either an independent file in the nginx config directory or set it up as a block in the main config. Since it redirects all traffic to ssl, you will need to get some ssl certificates. You will use certbot for this.

```bash
sudo certbot certonly --nginx -d adventhymnals.org
```

Now restart both csycms and nginx:

```
sudo systemctl restart nginx
sudo systemctl restart csycms
```