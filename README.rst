About
=====

Run the script ``py_versions_and_distros.py`` to find out which Linux distributions
have Python 3.6. It **downloads** resources data from `Distrowatch
<http://http://distrowatch.com//>`_.

I don't know if the result is accurate at all. Perhaps someone else can verify
it.

I'm assuming that a distribution has python 3.6, if in its resources text file,
it contains a package with the following patterns::

    python3 3.6*    e.g. antergos 17.6: python 3.6.1-1
    python3^3.6*    e.g. kaos 2017.07: python3^3.6.1-1
    python3-3.6*    e.g. fedora 26: python3-3.6.1-8.fc26.x86_64
    python 3.6*     e.g. gentoo unstable: python 3.6.1
    python-3.6*     e.g. arch current: python-3.6.2-1-i686.pkg.tar.xz
    python3.6*      e.g. parrotsecurity 3.7: python3.6^3.6.2~rc1-1


Setting up
==========

Requires Python 3.6.

Dependencies:

- Python 3.6
- `requests <http://docs.python-requests.org/en/master/>`_ library
- `Beautiful Soup <https://www.crummy.com/software/BeautifulSoup/>`_


Install the dependencies::

   $ python3 -m venv venv
   $ source venv/bin/activate
   $ pip install -r requirements.txt


Usage
=====

After ``venv`` is activated::

   $ python3 -m py_versions_and_distros


Output on July 17, 2017
-----------------------

::

    antergos 17.6: Python 3.6.1
    arch current: Python 3.6.1
    bluestar 4.11.5: Python 3.6.1
    debian unstable: Python 3.6.2
    debian testing: Python 3.6.2
    devuan unstable: Python 3.6.2
    fedora rawhide: Python 3.6.1
    fedora 26: Python 3.6.1
    funtoo current: Python 3.6.0
    gentoo unstable: Python 3.6.1
    kaos 2017.07: Python 3.6.1
    manjaro stable: Python 3.6.1
    manjaro 17.0.2: Python 3.6.1
    nixos 17.03: Python 3.6.0
    nixos 16.09: Python 3.6.0
    nutyx packages: Python 3.6.0
    obrevenge 2017.05: Python 3.6.1
    openbsd 6.1: Python 3.6.0
    opensuse tumbleweed: Python 3.6.1
    parrotsecurity 3.7: Python 3.6.2
    pisi 2.0-rc: Python 3.6.0
    swagarch 17.07: Python 3.6.1
    ubuntu snapshot: Python 3.6.2
    ====================================
    23 distros with Python 3.6


Output on July 23, 2017
-----------------------

::

    antergos 17.6: Python 3.6.1
    arch current: Python 3.6.2
    bluestar 4.11.5: Python 3.6.1
    debian unstable: Python 3.6.2
    debian testing: Python 3.6.2
    devuan unstable: Python 3.6.2
    fedora rawhide: Python 3.6.2
    fedora 26: Python 3.6.1
    funtoo current: Python 3.6.0
    gentoo unstable: Python 3.6.1
    kaos 2017.07: Python 3.6.1
    manjaro stable: Python 3.6.1
    manjaro 17.0.2: Python 3.6.1
    nixos 17.03: Python 3.6.0
    nixos 16.09: Python 3.6.0
    openbsd 6.1: Python 3.6.0
    opensuse tumbleweed: Python 3.6.1
    parrotsecurity 3.7: Python 3.6.2
    pisi 2.0-rc: Python 3.6.0
    swagarch 17.07: Python 3.6.1
    ubuntu snapshot: Python 3.6.2
    ====================================
    21 distros with Python 3.6


Notes
-----

``obrevenge`` was changed into ``revengeos``.  But the resources file doesn't exist.

``nutyx``'s package list moved permanently to a different URL 🤷🏻‍♀️


Background
==========

Since I found out that Fedora 26 will ship with Python 3.6, I started wondering
if there are other Linux distributions out there that also have Python 3.6.

I asked on `Twitter <https://twitter.com/mariatta/status/885704308775297024>`_
about it.

I was made aware of `distrowatch <http://distrowatch.com>`_ website,
and also about `a script <https://github.com/mlouielu/python-linux-distro-list>`_
for scraping the website to find out which Python is shipped in the Linux distros.

I took a look at the repo, and saw that `output.json <https://github.com/mlouielu/python-linux-distro-list/blob/master/output.json>`_
contains a list of Linux distros and the different Python versions shipped with them.
✨ Great!! 🎉 If only I can reduce the output to only distros with Python 3.6.

However the repo and source code does not come with any license info.

Two things:

1. I should first ask the author if I can modify their code.

2. If someone was able to scrape the site, maybe I can do it too.

I see an opportunity to write up a script that will use three
of my favorite things in Python: `f-strings`, `requests`, and `beautifulsoup4`.

I'm not going to pass up an opportunity to use f-strings 😛


Scraping
========

I need to first find out what are the available Linux distributions out there.

My first step is to download the main distrowatch website::

    def fetch_webpage(source_url, destination_filename):
        r = requests.get(source_url)
        with open(destination_filename, 'w+') as file:
            file.write(r.text)

    fetch_webpage("http://distrowatch.com", "downloaded_data/distrowatch.html")

I don't want to download the page each time I run the script, so let's download
it once per day.

::

   from datetime import datetime


   def scrape_webpage(source_url, destination_filename):
       if not os.path.exists(destination_filename):
           fetch_webpage(source_url, destination_filename)

       mod_date = datetime.fromtimestamp(
           os.path.getmtime(destination_filename))
       if mod_date.date() < datetime.today().date():
           fetch_webpage(source_url, destination_filename)


   scrape_webpage("http://distrowatch.com", "downloaded_data/distrowatch.html")


Now I have a local copy of the webpage, I can process it.

Looking at the webpage, the "Distribution" dropdown seems like a good source
for finding out all the available distributions. The HTML markup looks like
this::

   <select name="distribution">
        <option value="all">All</option>0Linux<option value="0linux">0Linux</option>
        2XOS<option value="2x">2XOS</option>
        3CX<option value="3cx">3CX</option>
        4MLinux<option value="4mlinux">4MLinux</option>
        ...
   </select>

To get all the values of the select options using beautifulsoup::

    with open("downloaded_data/distrowatch.html") as fp:
        soup = BeautifulSoup(fp, "html.parser")
        one = soup.select_one("select[name=distribution]")
        for item in filter(lambda x: x['value'], one.find_all("option")):
            yield item['value']


Next, to find out the released versions of each distributions.  Looking at
`fedora 26 <http://distrowatch.com/table-mobile.php?distribution=fedora>`_'s
distrowatch page, there is a table that compares the different releases:

=================  ==========  ==========  ==========
feature            rawhide     26          25
=================  ==========  ==========  ==========
Release Date       2017-07-23  2017-07-11  2016-11-22
...
Full Package List  rawhide     26          25
...
=================  ==========  ==========  ==========

The "Full Package List" row seems to have the info I want.  When I followed the
link there, it gives me a huge resources list all the packages available
for that distribution.

The url to the resources text file for fedora 26 is::

    http://distrowatch.com/resource/fedora/fedora-26.txt



It wasn't clear to me initially which of these items would indicate Python 3.6.
After inspecting the resources files from other distributions, I came to guess
that Python 3.6 can be listed as one of the following::

    python3 3.6*    e.g. antergos 17.6: python 3.6.1-1
    python3^3.6*    e.g. kaos 2017.07: python3^3.6.1-1
    python3-3.6*    e.g. fedora 26: python3-3.6.1-8.fc26.x86_64
    python 3.6*     e.g. gentoo unstable: python 3.6.1
    python-3.6*     e.g. arch current: python-3.6.2-1-i686.pkg.tar.xz
    python3.6*      e.g. parrotsecurity 3.7: python3.6^3.6.2~rc1-1


Now say it in Python::

    with open(resource_file_path) as file:
        for line in file.readlines():
            if line.startswith("python3 3.6") or line.startswith("python3^3.6") \
                    or line.startswith("python3-3.6") or line.startswith("python3.6") \
                    or line.startswith("python 3.6") or line.startswith("python-3.6"):
                # it has Python 3.6!
                # print it, write to csv


😅 That's all!


What I've learned
=================

Now I have a better idea of which other distributions have Python 3.6 😀

I learned how many linux distributions are out there. Before this, I've only
heard of `fedora`, `debian`, and `ubuntu` 😅😝

I didn't know how to retrieve the values from select options using beautifulsoup,
so I looked it up and learned new trick.

Writing code is easy. Writing documentation is hard. I spent an hour writing
the script. This readme file, a little more than half a day. I can choose not
to write anything, and just upload the code. But how else will I improve my
writing and communication skill?

What's next
===========

I think it will be interesting to run the script once a week (or once a month),
and see if there is any change in the output.

It will also be interesting to record the changes over time, to find out the trend
of Python 3.6 adoption.

I should learn how to use RegEx 😛

Things I will not do
====================

- Make this script backward compatible with Python < 3.6

- PEP 8 compliance

