---
layout: post
title: Playing with WebComponents
---

Thought I would have a go at migrating this jekyll based blog to use webcomponents.  This is the result, I hope it doesn't offend.

The long and the short of the story is that I got to understand jeykll much better, and hopefully haven't broken anything.  This version of the site is using the [Material Web Components](https://github.com/material-components/material-components-web-components) (MWC) that form a part of the [Polymer Project](https://www.polymer-project.org/).  These components are under active development by the polymer team and as such are not guaranteed to be functional.  If you find something broken, can you add a comment at the bottom of the page and I will try and investigate.

If it all goes horribly wrong, I also have a version of the blog that uses the mature/stable [Polymer Library](https://polymer-library.polymer-project.org/3.0/docs/about_30), or the default one that used the more standard Jekyll template.

Basic steps to make this work

- Use npm to install the packages you need
  - remember to use --save-dev, these are no longer runtime dependencies
- Create a 'sources.js' that pulled in references to the libraries we need.
- Use rollup to bundle all the needed javascript files into one
  - You need to use the rollup-plugin-node-resolve plugin to pull in files from node_modules
- Minify the bundles sources
  - I used terser for this as MWC makes use of es6+ capabilities

The whole project is available to see [here](https://github.com/m5p3nc3r/m5p3nc3r.github.io).

I'm done with this for a while, but I would like come back it at some point to see if its possible to implement the [Single Page Application](https://medium.com/a-lady-dev/single-page-applications-a-powerful-design-pattern-for-modern-web-apps-ec3590bb7e7a) design pattern on github pages.

## Conclusion

Living on the edge is fun, but can go horribly wront at times.  Lets see how this litte experiment goes!
