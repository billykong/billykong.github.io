---
layout: post
title:  "What is libv8 and Why its Installation Fails"
date:   2020-03-17 12:13:00 +0800
categories: ruby
comments: true
---

### Background

### In this blog, we will try to answer the following questions:
1. [Why do we need `libv8`?](#why-do-we-need-libv8)
    - a. [How is it related to `therubyracer`?](#how-is-it-related-to-therubyracer)
2. [Why does installation through `bundle install` fails?](#why-does-installation-through-bundle-install-fails)
3. [How do we fix it?](#how-do-we-fix-it)

### Why do we need `libv8`?
Before we talk about `libv8`, let's first take a look what is [V8](https://v8.dev/)  
> V8 is Google’s open source high-performance JavaScript and WebAssembly engine, written in C++. It is used in Chrome and in Node.js, among others.

So, basically, V8 is the Javascript engine in Chrome.

#### How is it related to `therubyracer`?

### Why does installation through `bundle install` fails?

### How do we fix it?



<!-- [libv8][libv8-github]{:target="_blank"} is a Javascript engine.


```
brew install v8-315

gem install libv8 -v '3.16.14.19' -- --with-system-v8
gem install therubyracer -- --with-v8-dir=/usr/local/opt/v8@315

bundle install
```

Questions:
- [ ] What is the `--` doing in `gem install`?
- [ ] What does `v8` and `therubyracer` do?
- [ ] Why does it fails in normal case when installing in OSX?


Reference
https://gist.github.com/fernandoaleman/868b64cd60ab2d51ab24e7bf384da1ca
 -->





[libv8-github]: https://github.com/rubyjs/libv8

















{% if page.comments %}
<div id="disqus_thread"></div>
<script>

/**
*  RECOMMENDED CONFIGURATION VARIABLES: EDIT AND UNCOMMENT THE SECTION BELOW TO INSERT DYNAMIC VALUES FROM YOUR PLATFORM OR CMS.
*  LEARN WHY DEFINING THESE VARIABLES IS IMPORTANT: https://disqus.com/admin/universalcode/#configuration-variables*/
/*
var disqus_config = function () {
this.page.url = PAGE_URL;  // Replace PAGE_URL with your page's canonical URL variable
this.page.identifier = PAGE_IDENTIFIER; // Replace PAGE_IDENTIFIER with your page's unique identifier variable
};
*/
(function() { // DON'T EDIT BELOW THIS LINE
var d = document, s = d.createElement('script');
s.src = 'https://billykong-github-io.disqus.com/embed.js';
s.setAttribute('data-timestamp', +new Date());
(d.head || d.body).appendChild(s);
})();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
{% endif %}