---
layout: post
title: Using virtual environments to separate project and OS Python software
---

Some of the operating system software on your computer is written in
Python.  If you need a package that's part of your OS that didn't get
installed at install time, you can install them later with `apt` or
`dnf`.

The software (and which versions of this software) is chosen and tested
by the OS bulders to work together well.  But let's say you want to
install a Python package that isn't part of the OS, or perhaps your
project needs a version that's different to that provided by the OS.
Traditionally you could use `pip` to install different Python software
into the OS's main place for Python packages, typically
`/usr/lib64/python3.11/site-packages` or similar, depending on your OS
and main Python version.

The problem is that installing not-included-in-the-OS packages can mess
up the software that's there as part of the OS.  Similarly, your
project may need an updated version of a package, but this updated
version is incompatible with the version your OS needs.  Upgrading may
break your OS!

In order to prevent this happening, when you install
not-included-in-the-OS (or updated) Python code, you're now strongly
encouraged to install it into a "virtual environment", rather than
where the OS puts stuff.  "Virtual environment" is just a directory
somewhere (usually nestled in with your project files) where Python
packages can be installed into.  Then, when you "activate" that virtual
environment and import something within your Python code, Python will
look there first, and if it can't find it there, then it'll search the
OS place.

This also means you no longer need to be root to install software
(because it gets installed into somewhere in your home directory), and
you can delete this virtual env without compromising the OS.

Now, it is possible to get `pip` to install into where the OS puts its
Python code, but really, just don't.  The move to virtual environments
is a _good thing_!  Let me give a practical demonstration:  I like
writing web apps in Django, which is an application framework written
in Python.

So I'm logged into my computer as "mjd", and my home directory is
`/home/mjd`:

```
[mjd@xiaomao ~]$ pwd
/home/mjd
```

I'll `cd` into where my web app is, then `cd` into my `pyproj` dir
inside that:

```
[mjd@xiaomao ~]$ cd git/afork.com/
[mjd@xiaomao afork.com]$ ls
pyproj  README.md
[mjd@xiaomao afork.com]$ cd pyproj/
[mjd@xiaomao pyproj]$ ls
authuser  conf  db.sqlite3  guestbook  manage.py  requirements.txt  static
```

Note there's a requirements.txt file that says the python packages that
need to be installed:

```
[mjd@xiaomao pyproj]$ cat requirements.txt 
Django==5.0.2
django-dotenv==1.4.2
dj_database_url==2.1.0
pytz==2024.1
# requests==2.31
```

And also that Django is not currently installed: 

```
mjd@xiaomao pyproj]$ ./manage.py check
ModuleNotFoundError: No module named 'django'

ImportError: Couldn't import Django. Are you sure it's installed and available on your PYTHONPATH environment variable? Did you forget to activate a virtual environment?
```

So, if I type `python`, which Python do I get, and where does it load
packages from?

```
[mjd@xiaomao pyproj]$ type python
python is /usr/bin/python
[mjd@xiaomao pyproj]$ python -c 'import sys; print(sys.path)'
['', '/usr/lib64/python311.zip', '/usr/lib64/python3.11', '/usr/lib64/python3.11/lib-dynload', '/usr/local/lib/python3.11/site-packages', '/usr/lib64/python3.11/site-packages', '/usr/lib/python3.11/site-packages']
```

Let's create a virtual environment (or "venv" for short), which is
stored in a directory called `env`, in the current directory:

```
mjd@xiaomao pyproj]$ python -m venv env
[mjd@xiaomao pyproj]$ ls -l
total 164
drwxr-xr-x. 4 mjd mjd   4096 Feb 20 13:59 authuser
drwxr-xr-x. 3 mjd mjd   4096 Feb 27 13:43 conf
-rw-r--r--. 1 mjd mjd 135168 Feb 20 14:57 db.sqlite3
drwxr-xr-x. 5 mjd mjd   4096 Feb 27 13:51 env
drwxr-xr-x. 5 mjd mjd   4096 Feb 23 18:35 guestbook
-rwxr-xr-x. 1 mjd mjd    660 Feb 20 13:35 manage.py
-rw-r--r--. 1 mjd mjd     88 Feb 20 13:35 requirements.txt
drwxr-xr-x. 3 mjd mjd   4096 Feb 20 14:48 static
```

Inside `env`, there's a directory called `bin`, that has versions of
`pip` and `python` that will load packages from this virtual environment:

```
[mjd@xiaomao pyproj]$ ls env/bin
activate      activate.fish  pip   pip3.11  python3
activate.csh  Activate.ps1   pip3  python   python3.11
```

In order to access this virtual environment, we have to "activate" it.
This will alter our path so `python` picks up the Python that's in the
venv's `bin` dir, and pip will pick up the `pip` that's in the `bin` dir:

```
[mjd@xiaomao pyproj]$ . env/bin/activate
(env) [mjd@xiaomao pyproj]$
```

Notice the `(env)` in our command-line prompt?  That's our cue that
we're running inside the virtual environment.  So, let's see which
Python we get when we run `python`, and where it looks for packages:

```
(env) [mjd@xiaomao pyproj]$ type python
python is /home/mjd/git/afork.com/pyproj/env/bin/python
(env) [mjd@xiaomao pyproj]$ python -c 'import sys; print(sys.path)'
['', '/usr/lib64/python311.zip', '/usr/lib64/python3.11', '/usr/lib64/python3.11/lib-dynload', '/home/mjd/git/afork.com/pyproj/env/lib64/python3.11/site-packages', '/home/mjd/git/afork.com/pyproj/env/lib/python3.11/site-packages']
```
 
That's great, we can see that python will now look for packages in our
venv. Now that our virtual env is activated, we can run `pip`, which
will install into the venv:

```
(env) [mjd@xiaomao pyproj]$ pip install -r requirements.txt 
Collecting Django==5.0.2
  Using cached Django-5.0.2-py3-none-any.whl (8.2 MB)
...
Successfully installed Django-5.0.2 asgiref-3.7.2 dj_database_url-2.1.0 django-dotenv-1.4.2 pytz-2024.1 sqlparse-0.4.4 typing-extensions-4.10.0
(env) [mjd@xiaomao pyproj]$
```

So what happened when we did the `pip`?  Let's have a look into where
the venv `python` will look for packages:

```
(env) [mjd@xiaomao pyproj]$ ls -l env/lib64/python3.11/site-packages/
total 200
drwxr-xr-x.  3 mjd mjd   4096 Feb 27 13:52 asgiref
drwxr-xr-x.  2 mjd mjd   4096 Feb 27 13:52 asgiref-3.7.2.dist-info
drwxr-xr-x.  3 mjd mjd   4096 Feb 27 13:51 _distutils_hack
-rw-r--r--.  1 mjd mjd    151 Feb 27 13:51 distutils-precedence.pth
drwxr-xr-x. 18 mjd mjd   4096 Feb 27 13:52 django
drwxr-xr-x.  2 mjd mjd   4096 Feb 27 13:52 Django-5.0.2.dist-info
drwxr-xr-x.  2 mjd mjd   4096 Feb 27 13:52 django_dotenv-1.4.2.dist-info
drwxr-xr-x.  3 mjd mjd   4096 Feb 27 13:52 dj_database_url
drwxr-xr-x.  2 mjd mjd   4096 Feb 27 13:52 dj_database_url-2.1.0.dist-info
-rw-r--r--.  1 mjd mjd   3669 Feb 27 13:52 dotenv.py
drwxr-xr-x.  5 mjd mjd   4096 Feb 27 13:51 pip
drwxr-xr-x.  2 mjd mjd   4096 Feb 27 13:51 pip-22.3.1.dist-info
drwxr-xr-x.  5 mjd mjd   4096 Feb 27 13:51 pkg_resources
drwxr-xr-x.  2 mjd mjd   4096 Feb 27 13:52 __pycache__
drwxr-xr-x.  4 mjd mjd   4096 Feb 27 13:52 pytz
drwxr-xr-x.  2 mjd mjd   4096 Feb 27 13:52 pytz-2024.1.dist-info
drwxr-xr-x.  8 mjd mjd   4096 Feb 27 13:51 setuptools
drwxr-xr-x.  2 mjd mjd   4096 Feb 27 13:51 setuptools-65.5.1.dist-info
drwxr-xr-x.  5 mjd mjd   4096 Feb 27 13:52 sqlparse
drwxr-xr-x.  2 mjd mjd   4096 Feb 27 13:52 sqlparse-0.4.4.dist-info
drwxr-xr-x.  2 mjd mjd   4096 Feb 27 13:52 typing_extensions-4.10.0.dist-info
-rw-r--r--.  1 mjd mjd 117599 Feb 27 13:52 typing_extensions.py
```

Knowing that these packages are now installed in the venv, we can now
run our Django program without error:

```
(env) [mjd@xiaomao pyproj]$ ./manage.py check
System check identified no issues (0 silenced).
```

So let's try it out.  We'll run Django's testing webserver on port
8000, but before that, some trickery: Let's start a job that will sleep
for 10 seconds, then use `curl` to request the top level page from the
site and show the first 4 lines of the HTML.  (The page doesn't exist
and will 404, but that's ok).  The `()` bundles the sleep and curl, and
the `&` runs the job in the background:

```
(env) [mjd@xiaomao pyproj]$ (sleep 10; curl -s 'http://localhost:8000/' | head -4) &
```
 
And we can see that job is running in the background: 

```
(env) [mjd@xiaomao pyproj]$ jobs
[1]+  Running                 ( sleep 10; curl -s 'http://localhost:8000/' | head -4 ) &
```

Now, while the sleep is happening, we can start the web server, which is dutifully listening on port 8000: 

```
(env) [mjd@xiaomao pyproj]$ ./manage.py runserver
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```
 
Ten seconds after we started the curl job, the request is made (and 404s, as expected), and we can see the first four lines of the returned HTML: 

```
Not Found: /
[27/Feb/2024 13:55:34] "GET / HTTP/1.1" 404 2279
<!DOCTYPE html>
<html lang="en">
<head>
  <meta http-equiv="content-type" content="text/html; charset=utf-8">
```

Finally, I can press _Ctrl-C_ to break out of the web server, and
deactivate the virtual environment:

```
^C
(env) [mjd@xiaomao pyproj]$ 
(env) [mjd@xiaomao pyproj]$ deactivate 
[mjd@xiaomao pyproj]$ 
```

(Deactivating is optional) 

By using a virtual environment, we've been able to install Python
software without interfering with (or destabilising) the Python
software that comes with the OS.  We also didn't need to log in as root
to do this, and it's valid and useful to have several virtual
environments, one per project, which may contain the same packages, but
where the versions of the packages is different.
