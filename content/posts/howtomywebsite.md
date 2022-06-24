---
title: "howtomywebsite"
date: 2022-06-24T20:36:40+08:00
tags: ["unpublished"]
draft: true
---

# how to my website  
  
a reference for me to run my own website.  
  
Important Notes:  
default post parameters are stored in /archetype/default.md  
ensure publishDir = "docs" (in config.toml) as github pages reads either /docs or /root/  
`hugo` to generate from content -> docs    
  
  
New Post:
hugo new posts/post.md
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