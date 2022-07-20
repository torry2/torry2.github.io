---
title: "howtomywebsite"
date: 2022-06-24T20:36:40+08:00
sort: ["unpublished"]
draft: true
---

# how to my website  
  
a reference for me to run my own website.  
  
Important Notes:  
default post parameters are stored in /archetype/default.md  
ensure publishDir = "docs" (in config.toml) as github pages reads either /docs or /root/  
`hugo` to generate from content -> docs    
  
  
New Post:
hugo new index/post.md
write in md  
  
draft: false  
  
hugo  
  
git add .  
git commit -m "post"  
git push  
  
  
Using Images/Files:  
store files in /static/  
static content will reflect in /docs/  
  
testing website locally:  
hugo  
hugo server  
  
  
  
markdown guide: https://www.markdownguide.org/cheat-sheet/
hugo docs: https://gohugo.io/documentation/




**** i changed posts -> index and tags -> sort
identifiers changed and some modifications in themes/etch/_default/index.html /partials/index.html and /?
en.toml leave as posts otherwise the time breaks 

/content/_index.md is bio