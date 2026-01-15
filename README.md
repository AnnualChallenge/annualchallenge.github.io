# 15 Jan 2025
To understand a bit more about Jekyll, I recommend starting with the [step-by-step](https://jekyllrb.com/docs/step-by-step/01-setup/)tutorial. For the purposes of my GitHub Pages blog, the things to be aware of are:
- The Jekyll build process takes the content within the Jekyll's root directory and converts it to HTML (or copies it if there is no Front Matter) and places it into the \_site folder.
- It keeps to the directory structure you define (unless explicitly configured otherwise).
- Within each folder, if you have an index.html, it treats it as the default page for the that directory.
- If you want the build to compile the files, they need to have 'Front Matter' at the beginning of the file. If not, it is just blindly copied across.
- To define a layout, you need a \_layouts directory and then, within the 'Front Matter' you need to reference the layout file, you've added to the \_layouts directory

An example of Front Matter for a Blog Post (sample that is included as a part of the install) is:

```
---
layout: post
title:  "Welcome to Jekyll!"
date:   2026-01-15 11:04:12 +0000
categories: jekyll update
---
```

Front Matter is simply YAML that can be references within the page using Liquid notation, and used to support the Jekyll build process.

To get the rough structure for this page, I've used ChatGPT - as it is pretty good at coming up with rough templates to work from. The direction given to ChaptGPT is that it has to be simple and clean and that it needs to use Bootstrap.

Create the default template - `\_layouts/default.html`
```
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8">
    <title>{{ page.title }} · {{ site.title }}</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <!-- Bootstrap CSS -->
    <link
      href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css"
      rel="stylesheet"
    >

    <!-- Custom styles -->
    <link rel="stylesheet" href="{{ '/assets/css/custom.css' | relative_url }}">
  </head>
  <body class="d-flex flex-column min-vh-100">

    {% include navbar.html %}

    <main class="container my-5 flex-fill">
      {{ content }}
    </main>

    {% include footer.html %}

    <!-- Bootstrap JS -->
    <script
      src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/js/bootstrap.bundle.min.js">
    </script>
  </body>
</html>

```

Create the Nav Bar - `\_includes\navbar.html`
```
<nav class="navbar navbar-expand-lg navbar-light bg-light border-bottom">
  <div class="container">
    <a class="navbar-brand" href="{{ '/' | relative_url }}">
      {{ site.title }}
    </a>

    <button
      class="navbar-toggler"
      type="button"
      data-bs-toggle="collapse"
      data-bs-target="#navbarNav"
      aria-controls="navbarNav"
      aria-expanded="false"
      aria-label="Toggle navigation">
      <span class="navbar-toggler-icon"></span>
    </button>

    <div class="collapse navbar-collapse" id="navbarNav">
      <ul class="navbar-nav ms-auto">
        <li class="nav-item">
          <a class="nav-link" href="{{ '/' | relative_url }}">Home</a>
        </li>
        <li class="nav-item">
          <a class="nav-link" href="{{ '/about' | relative_url }}">About</a>
        </li>
      </ul>
    </div>
  </div>
</nav>
```

Add the footer: `\_includes/footer.html`
```
<footer class="bg-light border-top py-3 mt-auto">
  <div class="container text-center text-muted">
    <small>
      © {{ site.time | date: "%Y" }} {{ site.title }}
    </small>
  </div>
</footer>
```

Add the style sheets: `assets/css/custom.css`
```
body {
  font-family: system-ui, -apple-system, BlinkMacSystemFont, sans-serif;
}

main h1,
main h2,
main h3 {
  margin-top: 2rem;
}

main p {
  margin-bottom: 1.25rem;
}
```

Update `index.md`.
```
---
layout: default
title: Home
---

# Welcome

This is a **simple Jekyll site using Bootstrap**.

- Responsive layout
- Clean typography
- Minimal custom CSS

You can focus on content and let Bootstrap handle the structure.
```

Finally update `\_config.yml` to add the site's title.
```
title: My Bootstrap Jekyll Site
```
The output is fairly basic looking, but it is a start.
# 10 Jan 2025
To ensure I can do some development and editing without having to wait for Github Pages to update, I've installed Jekyll locally. I'm using a Mac with Brew, to install all of the Jekyll dependencies, following these instructions: [jekyllrb.com](https://jekyllrb.com/docs/). This entry takes you through the steps I've taken to install Ruby and Jekyll and to get the test server running.

Installing Ruby on the MAC using Brew doesn't override the default Ruby install already there. The Mac's version of Ruby is too old. Therefore the $PATH variable needs to be updated so the Brew-installed Ruby takes precedence. 

Installing Ruby and updating the $PATH variable:

```
$> brew install ruby
$> echo export PATH=\"$(brew --prefix ruby)/bin:\$PATH\" >> .bash_profile
```

Sorry, but I'm still using 'bash' on the Mac - not "zsh", which is now Apple's preferred shell.

Note that there are issues with Ruby >3.0.0 with Jekyll that requires the "webrick" bundle to be added. Also, versions of Ruby > 4.0.0.0, also don't come with Logger installed. To get this working, the logger gem needs to be installed and "`gem "logger"`" needs to be added to the Gemfile.

```
$> gem install jekyll bundler logger
$> jekyll new myblog
$> cd myblog
$> echo "gem \"logger\"" >> Gemfile
$> bundle add webrick
$> bundle exec jekyll serve --livereload
```

There should now be a local HTTP server running on: `http://localhost:4000`

Next steps:
- Develop a template on localhost.
- Test the template on Github Pages.
- Add the most recent blog entries to GitHub pages, built on the new Jekyll template


# 1 Jan 2025
Happy New Year! May 2026 be a happy and prosperous year for all.

Now to start my Year of Code for real!

# 30 Dec 2025

GitHub Pages offers several themes that can be applied to the static site.  

All that is needed is a config file `_config.yml` pointing to the theme to use. 

As none of the themes ‘float my boat’, I’ve decided to structure the webpage myself, using the ‘Dinky’ theme as a starting template. 

Full details of how to do this is: [here](https://github.com/pages-themes/dinky) 

Using the Dinky ‘default.html’ template, I’ve made the following high-level changes:
- I’ve got rid of the footer
- I’ve linked out to a CDN offering the Bootstrap CSS and JavaScript libraries. 
- I’ve used Bootstrap to experiment with restructuring the template. I’ll see if I can get some advertising onto the site at a later stage. Using Bootstrap should hopefully make it easier to do.

The main problem for now is the time between changing the files in the repo, and GitHub Pages updating the site. GitHub Pages is based on Jekyll, so the next step is to install it locally so I can experiment with the sites look and feel without waiting several minutes for the change to go live. 

- Modifying the Dinky layout by including a '_layouts/default.html' file
- Adding bootstrap  Javascript and CSS links.
- Starting to structure the web page - menu on the left, content in the middle and future advertising to the right, etc.

# 29 Dec 2025

For my 2026 New Year's resolution,  I've decided to set myself technical challenges and document my progress for anyone to follow. I'm starting a bit early, just to get everything set up.

A bit about me: I'm a security architect and I spend most of my working day doing stuff that can't really be considered as hands-on anymore. A typical working week consists of many meeting, lots of emails, writing documents, creating architecture diagrams, supporting business development, and (when I do have some spare time) trying to develop security-architecture strategy. As a result I'm really hankering for that technical, hands-on past.

So, for 2026, I'm planning to set myself and document technical challenges to solve real-life problems (problems for me at least).

My first challenge: **setting up a single-page blog and repo to record my fun (and progress too)**.

To keep costs low - I've done the following:
- I've registered a cheap 'dot com' vanity domain for this project.
- I've set up a Github repo to store code, config and any documentation I may produce.
- I've configured Github pages to host this Blog, as it is free and uses Markdown (so I don't need to spent time writing HTML / Javascript / CSS, or dealing with a CMS, like Wordpress).

The look and feel of Github pages is a bit shoddy. I think I'll next spend a bit of time working on the styling of it; after which, I will  look into methods to simplify the publishing process - 'git' would be the simplest approach, but for the 'fun' bit I thinking of coding something rudimentary in Python to do the publishing bit.


---

