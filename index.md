---
layout: default
---
# 30 Jan 2026
In all my years of paying around with Python, whenever dealing with sockets, I've always just copied and pasted samples from the Internet, and then tweaked it a bit so it met my needs. I've always found Python socket programming a bit too involved.

This time I've decided to put in the effort to understand it. Part of that understanding will also be about understanding the various concurrency models that can be used with it. But for now, I'll stick with playing with the `socket` module. 

The sample code below demonstrates the most fundamental part of using `socket`. You first instantiate a socket and then bind it to a host and port - in this case 'localhost' and port 1066. After which you initialise it to listen for incoming connections.

The next two steps are: 1) accepting the connection; and 2) receiving the data from the client. Both default to blocking whilst waiting for something to happen. If you run the code, it will first stop at `s.accept()`; once a connection is made, it will stop at `con.recv()` (where `con` is the socket object created by a connection).

```
import time  
def main():  
    start = time.time()  
      
    # Initialise the socket and listen for a connection  
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)  
    s.bind((HOST, PORT))  
    s.listen()  
  
    # Accept a connection - socket.accept() is blocking  
    con, addr = s.accept()  
    print(f"time to accept: {time.time()-start}")  
  
    # Receive the data - socket.recv() is blocking  
    data = con.recv()  
    print(f"time to receive: {time.time()-start}")  
  
    s.close()
```

This code is pretty useless. It assumes that only one client will ever connect. Therefore there needs to be some kind of Event Loop to check for new connections and then service them.

To avoid the blocking of `accept` and `recv`, you need to set `socket.setblocking(False)`, but then the code very quickly throws a `BlockingIOError`.  

To get concurrency and handle these exceptions, the recommendation is to use either threads, asyncio or selectors.

Just for a bit of fun, I decided to do it myself - see the code below. 

Class `MySocketClass` acts as a container (with methods for new connections). As a part of `__init__`, I pass it the host and port and it then instantiates a socket. This allows me to create multiple sockets (for various ports) that are self contained within various instances of `MySocketClass`. 

There are two methods: one to check for new connections (`checkForNewConnection`) and the other to check for available data (`checkForData`). For `accept` and `recv` calls, I've used `try/except` to ignore the `BlockingIOError` exceptions.

I've created a `queue` - which is just a list - to hold all new connections that are then checked and serviced by `checkForData`.

Finally, within `main`, is my event loop, which is  an infinite `while` loop, which keeps running `checkForNewConnections` and `checkForData`.

```
import socket  
class MySocketClass():  
    def __init__(self, host_and_port):  
        self.socketObject = socket.socket(socket.AF_INET, socket.SOCK_STREAM)  
        self.socketObject.bind(host_and_port)  
        self.socketObject.listen()  
        self.socketObject.setblocking(False)  
  
        self.queue = []  
    def checkForNewConnection(self):  
        try:  
            conn, addr = self.socketObject.accept()  
            self.queue.append(conn)  
        except:  
            pass  
  
    def checkForData(self):  
        data = ""  
        for i in self.queue:  
            try:  
                data = i.recv(1024)  
                if data:  
                    print(data)  
                else:  
                    i.close()  
                    self.queue.remove(i)  
            except:  
                pass  
    def close(self):  
        self.socketObject.close()  
  
def main():  
    socket_a = MySocketClass(("localhost", 1066))  
    socket_b = MySocketClass(("localhost", 1067))  
  
    sockets = [socket_a, socket_b]  
  
    # Event loop  
    while True:  
        for i in sockets:  
            i.checkForNewConnection()  
            i.checkForData()  
  
if __name__ == '__main__':  
    main()
```

The end result works, but it is resource intensive. As can be seen in the screen-grab below, the code is consuming about 100% of a CPU's time handling the event loop. I could probably significantly drop the CPU load if I put in `time.sleep(...)`, but I'm not going to bother.

![](<assets/images/Pasted image 20260130130204.png>)

Next: I'm going to play with selectors, which is apparently what `asyncio` is built on.
# 26 Jan 2026
I've created a GitHub repo, which I'm calling [sneak](https://github.com/AnnualChallenge/sneak).A nice short name for a short project... hopefully. 

To configure `git` and synch with Github, I ran the following:

```
$ mkdir sneak && cd sneak
$ git remote add origin git@github.com:AnnualChallenge/sneak.git
$ git remote -v
$ git pull origin main
```

My preferred Python editor is PyCharm CE. Using PyCharm, I've set up a new project in the `sneak` directory.

For the design, I'm going to use object oriented code. I've defined a class called `SneakListener`. The idea is that a `SneakListener` object will be instantiated per open port. It will hopefully make the code cleaner, as everything for dealing with connections will be contained within the class.

A first step, using the `socket` and `threading` libraries, is creating a simple class that allows for a server socket to be instantiated on a selected port. Per connection to that port, it will spin up with a thread, with a handler function within the class to deal with it. This will permit multiple connections from multiple sources to that one port, without blocking per connection.

```
import socket  
import threading  
  
HOST = "localhost"   # Listen locally only  
PORT = 1066       # Listen on this port  
  
class SneakListener():  
    def __init__(self, HOST, PORT):  
        self.HOST = HOST  
        self.PORT = PORT  
  
    # handler used for each thread.  
    def handle_client(self, con, addr):  
        print(f"[CONNECTION] {addr}")  
        with con:  
            # Loop whilst connection is retained.  
            while True:  
                # Exception handling to deal with termninated-early connections (e.g. nmap SYN scans).  
                try:  
                    data = con.recv(1024)  
                    if not data:  
                        break  
                    message = data.decode()  
                    print(f"[{addr}] {message}")  
                    con.sendall(f"Echo: {message}".encode())  
                except:  
                    print("Connection terminated early - possibly nmap SYN Scan")  
  
        print(f"[DISCONNECTED] {addr}")  
  
    # Start method  
    def start_sneak(self):  
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as server:  
            server.bind((HOST, PORT))  
            server.listen()  
            print(f"[LISTENING] Server listening on {HOST}:{PORT}")  
  
            # Thread call.  
            while True:  
                conn, addr = server.accept()  
                thread = threading.Thread(  
                    target=self.handle_client,  
                    args=(conn, addr),  
                    daemon=True  
                )  
                thread.start()  
  
if __name__ == "__main__":  
    mysneak = SneakListener(HOST, PORT)  
    mysneak.start_sneak()
```

The main issue with the above code is that it blocks. The `start_sneak` function steps into a while loop and keeps going. So if I want to instantiate multiple `SneakListeners`, they won't be called as the first one is blocking.

The next step is to explore various unblocking techniques, such as using Python `asynchio` or use the non-blocking functionality that comes with the `socket` library. 

# 25 Jan 2026
First project - I thought I'd try something simple'ish first. I'm going to build a basic honey-pot with python. The primary objective is to detect suspicious behaviour in an environment; i.e. something is trying to connect to a port that shouldn't be being used.

Here's a set of initial requirements to work from.
1. The system must be capable of monitor any TCP and UDP ports.
2. The system must be able to detect TCP connection attempts (e.g. SYN packets)
3.  The system must be able to detect UDP packets being sent to any port.
4. The system must be able to operate with concurrent connections.
5. All connection attempts and ongoing connection engagements,  must be logged to a log file specified either on the command line or through the a configuration file.
6. If the log file already exists, events must be appended to the file (not overwriting the logs).
7. Log entries must contain as a minimum:
	- A timestamp.
	- The protocol (TCP or UDP).
	- Source and destination IPs
	- Source and destination ports
	- A truncated byte stream containing data send as a part of the connection or connection request (truncated to protect the log files from being saturated if large quantities of data is being sent).


# 22 Jan 2025
The challenge I have is that I'm using Obsidian (a note taking open-source application that uses `markdown`) to update the blog. This is stored in a separate folder to that of where my Jekyll site is hosted on my Mac. I want to still keep them separate, but at the same time, I don't want to have to manually copy files across, to then commit and push the changes into GitHub.

I'm going to try using hard-links `$ ln SOURCE DEST`. This will hopefully allow Git to commit the changes to the content, rather than committing a the link destination if I was just using a symbolic link.

Before executing the following line, make sure the old `index.md` is backed-up / renamed first.

```
$ ln <directory_to_obsidian_files>/myblog.md <dictory_to_blog_content>index.md
```

I've run into another problem. Obsidian doesn't add Front Matter. The local Jekyll server needs the Front Matter otherwise it breaks. Pages is also acting a little strange without it. However, manually adding the Front Matter using `vi` seems to stay put even after updating the document in Obsidian. Feels a little weak... but it is working for now.

# 20 Jan 2025
So I've created a git repo on my MAC; however, I've also got a running repo on GitHub where I was manually updating the files within GitHub. To avoid any issues I've decided to call the `main` branch `master` instead. 

First step to getting access is to create an SSH key:
```
$ ssh-keygen -t ed25519 -C "<email_address_used_for_GitHub>"
```

It should prompt for a filename (or suggest using the default `/Users/<mac_name>/.ssh/id_ed25519`). Choose a different filename if needed and then provide it with a passphrase when requested.

Edit or add the following to the file: `~/.ssh/config`.

```
Host github.com
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519
```

Run the ssh-agent in the background and add the new SSH key to the agent (including adding the pass phrase to the Apple Key Chain. Note that any previous GitHub SSH keys (if used  before) will need to be removed from `~/.ssh/known_hosts, otherwise SSH will throw a wobbly.

```
$ eval "$(ssh-agent -s)"
$ ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

Finally, add the public key to the GitHub account using the following [instructions](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)

Pushing from the local git repo, to GitHub, add the following to make sure there is a connection to GitHub:

```
$ git remote add origin git@github.com:<repo_name>
$ git remote -v
```

Then push into GitHub (note that I've pushed to master rather than main. This needs to be changed in GitHub Pages to make sure the site is coming from the master branch now):

```
$ git push -u origin master
```

# 17 Jan 2025
I managed to fix the site. I stripped out everything from `_config.yml`, apart from:
```
title: A Year Of Code
```

Seemed to work! Not sure why though.

The next step is being able to more easily publish changes without having to manually update the site. Time to remind myself as to how 'git' works.

Getting my up to speed, I've gone [W3Schools]([https://git-scm.com/book/ms/v2/Getting-Started-First-Time-Git-Setup](https://www.w3schools.com/git/default.asp?remote=github))

Setting up git
```
$ git config --global user.name "<username>"
$ git config --global user.email <email>
$ git config --global core.editor vi
```

Initialise the repo and create the `main` branch and add all the site's files.

```
$ git init
$ git add _config.yml _includes/ _layouts/ 404.html about.md index.md assets
```

Confirm that the necessary files / folders have been staged using:

```
$ git status
```

If any files / folders have been added by mistake, they can be removed from staging with the following command:

```
$ git restore --staged <file>
```

Finally, committing the staged changes and reviewing the commit log to make sure it worked.

```
$ git commit -m "First commit of the site."
```
Next steps:
- Get the changes into GitHub such that they are published into GitHub Pages.
- Do something to link my Obsidian markdown file (which is maintained in the Obsidian folder) into the git repository. I'm thinking on the lines of OSX hard links.

# 15 Jan 2025
To understand a bit more about Jekyll, I recommend starting with the [step-by-step](https://jekyllrb.com/docs/step-by-step/01-setup/) tutorial. For the purposes of my GitHub Pages blog, the things to be aware of are:
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

Create the default template - \_layouts/default.html.
```
{% raw %}
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8">
    <title>{{ page.title }} · {{ site.title }}</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <!-- Bootstrap CSS -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css"
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
    <script      src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/js/bootstrap.bundle.min.js">
    </script>
  </body>
</html>
{% endraw %}
```

Create the Nav Bar - `_includes\navbar.html`
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

Add the footer: `_includes/footer.html`
```
{% raw %}
<footer class="bg-light border-top py-3 mt-auto">
  <div class="container text-center text-muted">
    <small>
      © {{ site.time | date: "%Y" }} {{ site.title }}
    </small>
  </div>
</footer>
{% endraw %}
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

Finally update `_config.yml` to add the site's title.
```
title: My Bootstrap Jekyll Site
```
The output is fairly basic looking, but it is a start. Testing on GitHub Pages though isn't going as well... It looks like of borked it! I'm going to have to spend some time working out how to get Jekyll content developed and tested locally into Pages. 
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
