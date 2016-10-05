---
layout: page
title: Tech Books
author: Marc Cobos
permalink: /books/
cover:  "/assets/header_books.jpg"
---

<div class="about-header-container {% if page.cover %}has-cover{% endif %}" {% if page.cover %}style="background-image: url({{ page.cover | prepend: site.baseurl }});"{% endif %}>
  <div class="scrim {% if page.cover %}has-cover{% endif %}">
    <header class="about-header">
      <h1 class="title">{{ page.title }}</h1>
    </header>
  </div>
</div>
<br/>
In our world there are hundreds and hundreds of books to learn from. Here you have my list of books that I recommend.
<br/>

My esencial tech books (must-read books):
<br><br>
- <a href="https://www.amazon.es/gp/product/0596007124" target="_blank">Head First Design Patterns</a>

Easy read book to learn about the Design Patterns. They are necessary if you don't want to reinvent the wheel to solve common software problems. The single biggest benefit of design patterns in my opinion is that it gives developers a common vocabulary to talk about software solutions.<br/><br/>
- <a href="https://www.amazon.es/Refactoring-Improving-Design-Existing-Technology/dp/0201485672" target="_blank">Refactoring:Improving the Design of Existing Code</a><br/>

A good book to learn how to refactor your big ball of mud. Refactoring is a controlled technique for improving the design of an existing code base. Its essence is applying a series of small behavior-preserving transformations, each of which "too small to be worth doing". However the cumulative effect of each of these transformations is quite significant. By doing them in small steps you reduce the risk of introducing errors.<br/>
- <a href="https://www.amazon.es/gp/product/0321125215" target="_blank">Domain-Driven Design:Tackling Complexity in the Heart of Software </a><br/>

"In order to create good software, you have to know what that software is all about. You cannot create a banking software system unless you have a good understanding of what banking is all about, one must understand the domain of banking."

From: Domain Driven Design by Eric Evans.<br/><br/>
- <a href="https://leanpub.com/ddd-in-php" target="_blank">Domain-Driven Design in PHP</a><br/>

Real examples written in PHP showcasing DDD Architectural Styles, Tactical Design, and Bounded Context Integration
