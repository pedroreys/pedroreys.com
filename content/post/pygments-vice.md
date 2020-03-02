---
title: "Vice - A dark and colorful pygments style"
date: 2020-03-02T08:47:11+01:00
author: Pedro
type: post
url: "pygments-vice"
draft: false

tags:
  - python
---

My current project is a data sciency project in python which means I've been using [IPython](https://ipython.org/) and [Jupyter](https://jupyter.org/) a lot.

For a while now, my go to and favorite colorscheme/theme for coding is [vim-vice](https://github.com/bcicen/vim-vice). As I was working in this python project, I found out that IPython uses [pygments](https://pygments.org/) for styling and syntax-highlighting. After looking for a pygments port of vim-vice and not founding one I decided to do the port myself: [pygments-vice](https://github.com/pedroreys/pygments-vice)

![screencap][screencap]

## Porting from vim

The port itself wasn't too hard, but required a few steps, mostly because I couldn't bring myself to do it manually and had to find some tools to make it the "easy way". 

I started using [vim2pygments](https://github.com/honza/vim2pygments) to convert the vim colorscheme to a pygments style. But, unfortunattelly, that script didn't work with the way `vim-vice` was written, using functions and what not, as it expected raw `hi` vimscript commands in the colorscheme file. That meant taking a step back and and tweaking `vim-vice` to generate the actual `hi` commands and save that to a file.


With the vim colorscheme in the right format, I was able to use vim2pygments to convert it and used that as my starting point. Then, it took a few iterations of manually tweaking the styles defitions until I got to a point that I was happy with.


## Adding to pygments

After using the style myself for some time, I decided to share it as an official [pypi package](https://test.pypi.org/project/pygments-vice).

You can install it using pip:

```
pip install pygments-vice
```

## Using in the Web

After the style was available for pygments I was able to generate the [css style](https://gist.github.com/pedroreys/e7055625de7db5a6782af23aee264e3c) for it using `pygmentize`

```shell {lineos=false}
pygmentize -S vice -f html -a .highlight > vice_highlight.css
```
This is the css I'm using for color highlighting here in my blog. Here is an example

```python
import logging
import requests

log = logging.getLogger(__name__)

class ArticleNotFound(RuntimeError):
    """ Article query returned no results """

class Client(requests.Session):
    """ Mediawiki API client """

    def __init__(self, lang="en"):
        super(Client, self).__init__()
        self.base_url = f'https://{lang}.wikipedia.org/w/api.php'

    def fetch_page(self, title, method='GET'):
        """ Query for page by title """
        params = { 'prop': 'revisions',
                   'format': 'json',
                   'action': 'query',
                   'explaintext': '',
                   'titles': title,
                   'rvprop': 'content' }
        r = self.request(method, self.base_url, params=params)
        r.raise_for_status()
        pages = r.json()["query"]["pages"]
        # use key from first result in 'pages' array
        pageid = list(pages.keys())[0]
        if pageid == '-1':
            raise ArticleNotFound('no matching articles returned')

        return pages[pageid]
```

Enjoy!


[screencap]: https://i.imgur.com/jt6TthK.png "vice"
