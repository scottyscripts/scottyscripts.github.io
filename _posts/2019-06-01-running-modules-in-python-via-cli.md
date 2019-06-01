---
layout: post
title:  "Running Modules in Python via CLI: Spinning Up an HTTP Server"
date: 2019-06-01
---

## I Need A Server!

I was recently creating an up to date resume. I decided to make it from the ground up and ditch my old template. I usually create my resumes in some word processor, but I absolutely hate how difficult it is to format things the way that you want.

I did some browsing trying to find some free resume building software I could use. I found some pretty great options, but after a while I decided that I could just build out my resume on a webpage. I knew that I could very easily customize layout down to the pixel using HTML and CSS.

So I whipped up some basic HTML. Instead of opening the file directly in Chrome, I had the thought of starting a local web server. Then I had another thought, *If I had to spin up a local web server right now, how would I do it?*

<img src="https://media.giphy.com/media/lELRD773cY7Sg/giphy.gif" width="400" />

# Right on, Python

If this wasn't a new computer, I'd have some script or alias to spin up a server. In the past, I've used Node.js, Apache, and Ruby to accomplish this. I've been trying to work with more Python recently, so I decided to look into a Python solution for this problem. The other week I was reading some docs on the `http.server` module. I remembered reading something about invoking `http.server` via CLI.

```bash
python3 -m http.server ${port_number:-8000}
```

# Stepping Back: Running Python Modules Via CLI

Let's take a closer look at the `-m` flag provided by the `python` command.

```bash
$ python3 --help | grep -- -m
-m mod : run library module as a script (terminates option list)
```

Python modules can contain functions and classes, but they can also contain executable code. With the `-m` flag, we can execute the code in any Python module from the CLI. The code in the module specified with the `-m` flag gets executed as the `__main__` module.
- If the module (any Python file) is executed as the main program, the `__name__` variable is set to `'__main__'` by the interpreter.

If we look at the [source code for the `http.server` module](https://github.com/python/cpython/blob/3.7/Lib/http/server.py#L1240), we see the following.

```python
if __name__ == '__main__':
    import argparse

    # more code
```

This is the code that will be run when we run `python -m http.server`.

# Back To The Command

```bash
python3 -m http.server ${port_number:-8000}
```
As shown by my use of [shell parameter expansion](https://www.gnu.org/software/bash/manual/html_node/Shell-Parameter-Expansion.html), a port number can either be provided, or the server will run on port 8000 by default.

There are more options that can be specified for the server via flags. For example, the `-d` flag is used to specify the directory to be served. By default, the current directory is used (i.e. make sure your `index.html` file is in the current directory).

If you are curious to learn more about the command or the options you can specify, type the following in your console.

```bash
python3 -m http.server --help
```

OR checkout the [source code for the `http.server` module](https://github.com/python/cpython/blob/3.7/Lib/http/server.py#L1240).

# You Got Served

Now that it's clear how we are running the executable code in the `http.server` Python module, lets start the server!

With my current directory containing my `index.html` file, I run

```bash
python3 -m http.server
```

<img src="https://media.giphy.com/media/prUUjzUdXM7II/giphy.gif" width="300" />

Visiting localhost:8000 in my browser tells me that it worked!

# Conclusion

I found my new favorite way to run an HTTP Server! I didn't need any additional packages or scripts, only Python. This isn't a solution I would ever use in production, but it is great for running a local HTTP server for a situation like mine.

When I started to learn about everything in this post, my intention was just to write my resume. I think that really goes to show how there is always opportunities to learn in things that you are doing. Ask yourself questions and think creatively even when doing mundane tasks. You never know what problems you will think of, or the potential resulting solutions that can benefit you in the long run.
