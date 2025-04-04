---
title: "Delay tree"
date: 2025-02-07T17:23:00+02:00
draft: false
mermaid: false
cover:
    image: "articles/nuget-caching/cover.jpg"
    alt: "cover"
    caption: ""
    relative: false
author:
    name: "Olivier Giniaux"
    title: "Staff Backend Engineer"
    icon: "authors/olivier-giniaux.jpg"
    link: "https://www.linkedin.com/in/oginiaux/"
summary: 
tags: 
- dotnet
- optimization
- performance
- task
- delay
- tree
- async
- cancellationtoken
---

Min max 4 levels = 2^4 = 16 values
```plantuml
@startmindmap

skinparam backgroundColor transparent

top to bottom direction

* ""root""
 * ""0***""
  * ""00**""
   * ""000*""
    * ""0000""\n"" = 0""
 * ""1***""
  * ""11**""
   * ""111*""
    * ""1111""\n""= 16""

@endmindmap
```

Insert delay of 4ms
```plantuml
@startmindmap

skinparam backgroundColor transparent

top to bottom direction

* ""root""
 * ""0***""
  * ""01**""
   * ""010*""
    * ""0100""\n"" = 4""

@endmindmap
```

Inserting another delay of 4ms results in no new node

# Taking elapsed time into account

# Finding delays between two timestamps

```plantuml
@startmindmap

skinparam backgroundColor transparent

top to bottom direction

*[#lightgreen] ""root""
 *[#lightgreen] ""0***""
  * ""00**""
   * ""001*""
    * ""0010""\n"" = 2""
  *[#lightgreen] ""01**""
   *[#lightgreen] ""010*""
    *[#lightgreen] ""0100""\n"" = 4""
    *[#lightgreen] ""0101""\n"" = 5""
   * ""01*""
    * ""0111""\n"" = 7""
 * ""1***""
  * ""10**""
   * ""101*""
    * ""1010""\n""= 10""

@endmindmap
```

Compare timestamps min max with children bits

What about value 2? Why is it left out? -> onto wrapping

# Wrapping