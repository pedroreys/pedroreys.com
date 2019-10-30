Today I was following the instructions to upgrade to the latest version of the Twitter Bootstrap.

It&#8217;s pretty straight forward. It is really awesome, actually. All you have to do is: Open the terminal, pull the changes, run `make`.

But, Twitter Bootstrap uses Less.css, so one of the steps of the update script is to compile the `.less` files into `.css`. I didn&#8217;t have less compiler installed on my MacBook Pro, so instead of successfully upgrading the Bootstrap, I got this on my terminal:

![lessc: command not found](http://f.cl.ly/items/33232j3z201O3B0s2q3q/Screen%20Shot%202012-02-02%20at%208.16.59%20PM.png) 

So, my first reaction was to try installing less using Homebrew:

![Homebrew doesn't have a formula for less.css](http://f.cl.ly/items/260U2t3b1L3Q392G1q1h/Screen%20Shot%202012-02-02%20at%208.19.46%20PM.png) 

Dang it. There is not Homebrew formula for Less. I&#8217;ll have to open my browser to install a software, damn it. So I headed to [Less.css](http://lesscss.org/) website and saw that the easiest way to install the compiler to be used on the server side is via the node package manager, [npm](http://npmjs.org/ "package manager for node"). Then I did: `npm install less`

![npm installed less](http://f.cl.ly/items/043Q440w3e2w2q3o2K2f/Screen%20Shot%202012-02-02%20at%208.27.12%20PM.png) 

All right, now let&#8217;s try running make again.

![less not found again](http://f.cl.ly/items/3Q363v1l0l2I0Y0a0v3B/Screen%20Shot%202012-02-02%20at%208.28.58%20PM.png) 

Ok. It&#8217;s still not found. Maybe installing it globally will solve the problem: `npm install less --global`

![npm install less with global option](http://f.cl.ly/items/1o061I3E461t2S2t1D0Z/Screen%20Shot%202012-02-02%20at%208.36.59%20PM.png) 

All right, now that less is installed globally, let&#8217;s try running `make` again to update Twitter Bootstrap.

![Bootstrap upgraded succesfully. Yay!](http://f.cl.ly/items/0O2u2w0O3h1g1t0d0U2t/Screen%20Shot%202012-02-02%20at%208.38.25%20PM.png) 

Yay, it worked. The `--global` option did the trick.