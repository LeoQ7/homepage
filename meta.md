---
title: "Hugo Meta"
date: 2020-10-03T04:54:02+08:00
lastMod: 2020-10-15T05:26:53+0800
draft: true
hiddenFromHomePage: true
hiddenFromSearch: true
tags: [Hugo]
---

### To run hugo

```shell
hugo server -D --disableFastRender --bind 0.0.0.0 #-e production
hugo --cleanDestinationDir --ignoreCache
```

### Things remember to enable for extra functions

```toml
[params.page]
  twemoji = false
  # If have images
  lightgallery = false
  fontawesome = false
  [params.page.math]
    enable = false
  [params.page.code]
    copy = false
```

### Auto update `lastMod` for Hugo and keep correct position

One bug here maybe for Hugo, vim script does not have the correct name, it shows `Code` below.

```vim
" Auto update `lastMod` for Hugo and keep correct position
func! UpdateLastMod()
    " Only when modified and not template
    if &modified && expand('%:t') != 'default.md'
        let save_cursor = getpos(".")
        exe '%s/^lastMod:.*$/lastMod: ' . strftime('%FT%T%z') . '/e'
        call setpos('.', save_cursor)
    endif
endfunction
au FileType markdown aut BufWritePre * call UpdateLastMod()
```

### Style override for `a href`

Do I still want this?

```html
{{</* style "[theme=dark] & a{color:#dddddd;}; a{color:#777777;}" */>}}
...
{{</* /style */>}}
```

To escape the above text in Hugo, use `{?{}{?{}</* style ` and ` */>{?}}{?}}`.

To escape `{` and `}`, use `{{??}{}` and `{{??}}}`.  <!-- {?{} and {?}} -->

To escape `{{??}{}` and `{{??}}}`, use `{?{}{?{}{??}{??}{?}}{?{}{?}}` and `{?{}{?{}{??}{??}{?}}{?}}{?}}`...

### Do not show fa-kiss-wink-heart

At `layouts/partials/footer.html:51`, originally:
```
<!-- {{- $theme := .Scratch.Get "version" | printf `<a href="https://github.com/dillonzq/LoveIt" target="_blank" rel="noopener noreffer" title="LoveIt %v"><i class="far fa-kiss-wink-heart fa-fw"></i> LoveIt</a>` -}} -->
```

### More button

[https://stackoverflow.com/a/39040438](https://stackoverflow.com/a/39040438)

```html
<style>
  .details, .show, .hide:target { display: none; }
  .hide:target + .show, .hide:target ~ .details { display: block; }
</style>
<div>
  <a id="hide1" href="#hidenews" class="hide">... more history news</a>
  <a id="show1" href="#shownews" class="show">... less history news</a>
  <div class="details">
    <ul>
      <li><span class="date">Aug. 2020</span> (Bofore JHU) BORA is accepted to appear at <a href="https://sc20.supercomputing.org">SC'20</a>.</li>
    </ul>
  </div>
</div>
```

Using HTML 5.1

```html
<details open>
  <summary><strong>Overview</strong></summary>
  <ol>
    <li>Cash on hand: $500.00</li>
    <li>Current invoice: $75.30</li>
    <li>Due date: 5/6/19</li>
  </ol>
</details>
```

### Responsive design for image
```css
@media only screen and (min-width: 600px) {
    .pic { width: 250px; padding: 25px; float: right; }
}
@media only screen and (max-width: 600px) {
    .pic { width: 80%; max-width: 250px !important; display: block; margin-left: auto; margin-right: auto; }
}
```

### Favicon

[https://realfavicongenerator.net](https://realfavicongenerator.net)

https://realfavicongenerator.net/favicon_result?file_id=p1ejnsc49d19ui1m5r19ee7soudt6#.X3jGZi1h3UI

### Links

CloudLab (Do I want to add this?)

- This website settings
```
Color scheme preference: <a href="javascript:localStorage.removeItem('theme'),document.body.setAttribute('theme',window.matchMedia('(prefers-color-scheme:dark)').matches?'dark':'light'),setTimeout(function(){alert('Color scheme settings cleared.')},50)">Set back to auto</a>
```
