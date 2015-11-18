---
layout: post
title:  Testi
---
## Text
This is a test post!

## Quote
"Here's a quote"

## List
- thing no 1
- thingy 2
- third thing

## Code
### vim
{% highlight vim linenos %}
" local config
if filereadable(expand("~/.vimrc.local"))
    source ~/.vimrc.local
endif
{% endhighlight %}

### bash
``` bash
#!/bin/bash

size="$(mpc playlist | wc --lines )"
current="$(mpc current --format '%position%')"
[[ -z $current ]] && exit 1
mpc move $current $size
```
