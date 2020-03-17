---
layout: post
title:  "What is libv8 and Why its Installation Fails"
date:   2020-03-17 12:13:00 +0800
categories: ruby
comments: true
---

### Background
When I cloned a project and tried to run `bundle install`, I receive the error
```
ERROR: Failed to build gem native extension.
...
fatal error: 'climits' file not found #include <climits>
```

The error is raised when `bundler` tries to require `libv8`, which is in turn required by `therubyracer`.   

If you are reading this article, I believe I can assume that you are having a similar problem. I am going to share my findings here.
<br>
<br> 

### In this blog, we will try to answer the following questions:
1. [Why do we need `libv8`?](#why-do-we-need-libv8)
    - a. [How is it related to `therubyracer`?](#how-is-it-related-to-therubyracer)
2. [Why does installation through `bundle install` fails?](#why-does-installation-through-bundle-install-fails)
3. [How do we fix it?](#how-do-we-fix-it)
<br>
<br>

### Why do we need `libv8`?
Before we talk about `libv8`, let's first take a look what is [V8](https://v8.dev/)  
> V8 is Google’s open source high-performance JavaScript and WebAssembly engine, written in C++. It is used in Chrome and in Node.js, among others.

So, basically, V8 is the Javascript engine in Chrome. And `libv8` is the ruby interface for the V8 engine used by `therubyracer`.
<br>

#### How is it related to `therubyracer`?
The `therubyracer` gem [embeds the V8 Javascript Interpreter into ruby][github-therubyracer]. With it, we can call Javascript functions or use Javascript objects directly in ruby runtime.
<br>
<br>

### Why does installation through `bundle install` fails?
The installation failed as `therubyracer` gem calls the `libv8` gem, and it fails to build a native extension, i.e. `v8`. I am able to repeat the same bug with:   

```bash
$ gem install therubyracer
Building native extensions. This could take a while...
ERROR:  Error installing therubyracer:
    ERROR: Failed to build gem native extension.

    current directory: /Users/billykong/.rvm/gems/ruby-2.6.5/gems/libv8-3.16.14.19/ext/libv8
...
Compiling v8 for x64
Using python 2.7.16
Using compiler: c++ (clang version 11.0.0)
Unable to find a compiler officially supported by v8.
It is recommended to use GCC v4.4 or higher
Beginning compilation. This will take some time.
...
```
The error in your case may be different but still there is one interesting finding here. **The `therubyracer` gem installation script is trying to compile `v8` from source!** I am not saying it is impossible, but I got a feeling that the compilation is the cause of issues for many of us. Just the first few posts I found when I google "libv8 error":    

- [https://stackoverflow.com/questions/19673714/error-installing-libv8-error-failed-to-build-gem-native-extension](https://stackoverflow.com/questions/19673714/error-installing-libv8-error-failed-to-build-gem-native-extension){:target="_blank"}
- [https://gist.github.com/simonqian/f67c862d06c94fe315d0ba97b711571f](https://gist.github.com/simonqian/f67c862d06c94fe315d0ba97b711571f){:target="_blank"}
- [https://stackoverflow.com/questions/19577759/installing-libv8-gem-on-os-x-10-9?noredirect=1&lq=1](https://stackoverflow.com/questions/19577759/installing-libv8-gem-on-os-x-10-9?noredirect=1&lq=1){:target="_blank"}

Compiling something requires all the dependencies to be present. Since `v8` is written in C++ and probably many of us don't have a C++ development environment setup ready for use. Perhaps it is easier just to download a suitable binary of v8 instead of building it from source.
<br>
<br>

### How do we fix it?
```bash
# 1. Install v8 ourselves
$ brew install v8-315
# 2. Install libv8 using the v8 binary we just installed
$ gem install libv8 -v '3.16.14.19' -- --with-system-v8
# 3. Install therubyracer using the v8 binary we just installed
$ gem install therubyracer -- --with-v8-dir=/usr/local/opt/v8@315
# 4. Install the remaining dependencies
$ bundle install
```
Note: The `--` before the `--with-system-v8` and `--with-v8-dir=/usr/local/opt/v8@315` is necessary when we supply library paths for native extensions, i.e. using specified libraries/binaries instead of building them.
<br>
<br>

### Conclusion
`therubyracer` or some other gems may requires native extensions to function properly. While some tries to build the extension from source code, it often fails because the dependencies required by the native extensions are not readily installed. We may solve the issues by supplying our own binaries.  

Hope that you have find the information useful. If you find any error or mistake in the above content, please do feel free to leave a comment below.
<br>
<br>

### Reference
1. [https://v8.dev/docs](https://v8.dev/docs){:target="_blank"}  
2. [https://github.com/rubyjs/libv8](https://github.com/rubyjs/li bv8){:target="_blank"}
3. [https://github.com/rubyjs/therubyracer](https://github.com/rubyjs/therubyracer){:target="_blank"}
4. [https://gist.github.com/fernandoaleman/868b64cd60ab2d51ab24e7bf384da1ca](https://gist.github.com/fernandoaleman/868b64cd60ab2d51ab24e7bf384da1ca){:target="_blank"}
5. [https://guides.rubygems.org/command-reference/](https://guides.rubygems.org/command-reference/){:target="_blank"}
6. [https://stackoverflow.com/questions/19673714/error-installing-libv8-error-failed-to-build-gem-native-extension](https://stackoverflow.com/questions/19673714/error-installing-libv8-error-failed-to-build-gem-native-extension){:target="_blank"}
7. [https://gist.github.com/simonqian/f67c862d06c94fe315d0ba97b711571f](https://gist.github.com/simonqian/f67c862d06c94fe315d0ba97b711571f){:target="_blank"}
8. [https://stackoverflow.com/questions/19577759/installing-libv8-gem-on-os-x-10-9?noredirect=1&lq=1](https://stackoverflow.com/questions/19577759/installing-libv8-gem-on-os-x-10-9?noredirect=1&lq=1){:target="_blank"}
<br>
<br>


[libv8-github]: https://github.com/rubyjs/libv8
[github-therubyracer]: https://github.com/rubyjs/therubyracer




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