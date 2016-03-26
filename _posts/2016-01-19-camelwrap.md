---
layout: post
title: "Automatic Word Wrapping for LongCamelCaseWords"
date: 2016-01-19 21:52:11 +0100
tags: javascript coding
---

Responsive design and Objective-C blogs don't mix well. The convention of using long descriptive names can easily break layouts on smaller devices. I fixed the problem by creating a JavaScript function to wrap long CamelCase words automatically. 

# Problem

Since mobile phones account for huge parts of todays browsing, it is advised to make your website look good on narrow screens. A lot of websites use responsive design to automatically adjust the layout to be more usable when accessed from small devices. Menus get collapsed into [hamburger buttons](https://en.wikipedia.org/wiki/Hamburger_button), sidebars get stacked vertically and images are scaled to not exceed the screen width. 

A not yet generally solved case is the handling of long words, like URLs or variable names. They are usually part of the text content and can't be scaled individually like images. Text also needs to be large enough to be readable on small screens. If not handled properly, these long words will break your design and make your website scroll sideways. 

# Solutions

The probably simplest solution is to use the CSS property `word-wrap: break-word;`. This will break words that run out of space, even if they don't contain any allowed break points. This approach has two problems: (1) A paragraph in the form of "shortWord longWord" will leave the short word on it's own line, creating a visually undesirable orphan and (2) the word will not wrap where it makes sense, but when space runs out, thus reducing readability and potentially creating widow letters that are left on their own new line.  

Optional word wraps can also be defined manually. This can be achieved by either inserting a `&#8203;` unicode character (Zero Width Space) or a `<wbr>` HTML tag (Word Break Opportunity) into the word. The unicode character will sometimes show up as a regular space when copying and pasting the word into a text editor, which is quite annoying when dealing with code. On the other hand, unicode works better in some use cases where using HTML tags is not supported. Both approaches might make the word harder to find in search engines. 

This blog is currently hosted on GitHub using [Jekyll](http://jekyllrb.com/). I write the articles in [MarkDown](https://daringfireball.net/projects/markdown/) and have little control over how the content is converted to HTML. While I can easily enter unicode characters and HTML tags in regular paragraphs, it's impossible to do so inside code segments where ampersands and angle brackets are converted HTML entities. Unfortunately, I'm occasionally writing about long Objective-C names and want to signal them being code by displaying them as such. It would work if I'd wrap them in `<code>` tags manually instead of using the regular syntax, but that would be much harder to type and the article would become dependent on its representation and template. 

Being dependent on the Jekyll setup on GitHub also limits my possibilities of server side automation. Adding additional text processing would also be rather complicated, since I could break the resulting HTML by for example modifying long words in URLs. I would need to either parse the MarkDown syntax or the resulting HTML code. 

# CamelWrap

After recognizing the layout problems of my website on my phone and evaluating the solutions mentioned above, I came up with this relatively short JavaScript. To make all CamelCase words with 18 characters or longer wrappable, I simply add `camelWrap(document.body)` into the `onload` attribute of the `<body>` tag. 

```javascript
function camelWrap(node) {
  camelWrapUnicode(node);
  node.innerHTML = node.innerHTML.replace(/\u200B/g, "<wbr>");
}
function camelWrapUnicode(node) {
  for (node = node.firstChild; node; node = node.nextSibling){
    if (node.nodeType==Node.TEXT_NODE) {
      node.nodeValue = node.nodeValue.replace(/[\w:]{18,}/g, function(str) {
        return str.replace(/([a-z])([A-Z])/g, "$1\u200B$2");
      });
    } else {
      camelWrapUnicode(node);
    }
}
```

First, the function traverses all DOM text nodes and inserts a zero width space where appropriate. This modification already produces the desired visual result, but has the aforementioned problems with copy and paste. In a second pass, it replaces the unicode characters with the HTML tag. Doing the initial substitution on the HTML code directly would break links, and inserting the tag in the first pass would require a few more lines to modify the DOM tree. Using `innerHTML` might decrease site load speeds, but it is good enough for me. 

The resulting HTML might look something like `Camel<wbr>Case<wbr>Word`. Since the modification happens client side, search machines will still be able to find the initial string. If this site still uses the same code, its effect can be seen in the title when resizing the browser window.   


