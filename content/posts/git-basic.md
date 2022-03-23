---
title: "Git Basic"
description: "Git fundamental"
tags: ["git"]
categories: ["git", "basic"]
date: 2020-05-15T15:55:52+08:00
---

### Problem Solving

#### Authentication

+ Clear saved account & password
  
  Error: `fatal:Authentication failed for 'remote-url'`
  
  `git config --system --unset credential.helper`

+ Save account & password
  
  `git config --global credential.helper store`

#### Remote

+ Check remote url
  
  `git remote -v`

+ Update remote url
  
  `git remote set-url origin newRepoUrl`
