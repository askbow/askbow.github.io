---
layout: post
title:  "How to write long iterative configs"
date:   2015-05-12 14:23:00 +0100
categories: automation
---

> Note: I wrote this in 2015, just starting with network automation.
> Nowadays, use jinja2 or bottle

Sometimes there is a problem: you need to write a really repetitive config with hundreds of nodes. Clear example would be dial peers in Cisco routers. It is immensely hard and dull to do this by hand or copy+paste. Here\'s a simple python script to do it for you.

## Iterative config generation

I prefer to solve such problems with some scripting, and my go-to language for that is Python. Also, this script is a clear example of data-code separation. I store the data used to generate the config in a separate file (configtemplate.py) from the generator code itself (configexpand.py). Let\'s look at them closely.

### Data used to generate config

First, we need data to generate a config from. In my case, the data is divided in two types:

1. Config template - this is the part which the script will iterate
2. Data lines - a table of the parts of config that need be changed in each iteration, like IP addresses and description lines

The config template is basically a multi-line string, which stores the config almost just like you\'ll put it into your device\'s console. The only difference is that at each place where a substitution from the data table is expected, we place a special tag: {tagname}.

Here\'s an example of dial-peer configuration for CUBE used in my script:

```python
CONFIG_TEMPLATE_ITERATIVE = '''dial-peer voice {sequence} voip 
description {description} 
preference 1 
destination-pattern {pattern} 
session protocol sipv2 
session target ipv4:{ip} 
dtmf-relay rtp-nte 
codec g711alaw 
no vad'''
```

Here, the

```
CONFIG_TEMPLATE_ITERATIVE
```

variable stores the template itself. Nothing fancy, except for the tags (sequence, description, pattern, ip) mentioned above

Moreover, I used a separate Python dictionary to store static variables that get placed into config. It is a little overkill for my task at hand, but it seems to be reasonable to write it in that way for any future use. Here\'s how it looks like:

```
CONFIG_TEMPLATE_STATIC = dict(ip='192.0.2.42')
```

Here one can add any number of static tags. As I show it later, the {ip} tag in the template is replaced by the data stored here under [ip]. Also, any static stored data here will take precedence over data lines for the same tag. Aside from that, my config template stores an offset for the counter (which is filled into the sequence tag), as my task was to generate a config to be appended to the similar config already in place. The use of a counter here is obvious: there is no point in storing this number with the rest of the data used to generate the config. Additionally, not used in my example, but nevertheless present in the script (for future use) are two variables:

```
CONFIG_TEMPLATE_ADD_BEFORE = 
CONFIG_TEMPLATE_ADD_AFTER = 
```

Their meaning should be evident from variables\' names: one prepends the iterative part, and the later is appended to the end.

The data lines file is a simple CSV-table. The first line, being the table header, stores the {tags} for respective columns. In my case it looks like this:

```
pattern;description
97520.......$;IN, Indianapolis
97513.......$;IN, Smalville
96412.......$;IN, Regentsville
```

In the form of table that would be:

| pattern | description |
|---------|-------------|
| 97520.......$ | IN, Indianapolis |
| 97513.......$ | IN, Smalville |
| 96412.......$ | IN, Regentsville |

Nothing fancy here.

### The script used to generate config

The script has two main procedures:

1. main() - as the name suggests, this is the main part which reads the data file, writes the resulting config and iteratively calls the
2. configblock(dataline) procedure takes the iterative part of the template (stored in CONFIG_TEMPLATE_ITERATIVE) and formats it (in my case - replaces {tags} with respective fields from the dictionary.

The `configblock(dataline)` procedure is very simple:

```python
def configblock(dataline):
  config = CONFIG_TEMPLATE_ITERATIVE
  return config.format(**dataline)
```

That\'s all. Here the procedure just returns the result of

```python
.format()
```

method of a string variable. This could even be done in-line at the main procedure, but I sacrificed a little efficiency for a little more readability. And here\'s how the main() procedure goes (I removed the lines concerned with the little user interface the script has, as they are not crucial for the script\'s work):

```python
DataFileReader = csv.DictReader(open(r'data.csv'), delimiter=';')
```

First, the script reads the data file. The file is read into a list of dictionaries, which allows for two important things:

1. It can be easily iterated through
2. At each iteration we can find the iteration[\'tagname\'] (see further)

```python
  sequence = CONFIG_TEMPLATE_COUNTER['offset'] 
  config = CONFIG_TEMPLATE_ADD_BEFORE + \n
```

Next, the sequence number variable (used later as a counter) is filled with an offset and the config variable (used to store script\'s results) is pre-populated with the CONFIG_TEMPLATE_ADD_BEFORE data.

```python
  for line in DataFileReader:
    line['sequence'] = str(sequence) 
    line.update(CONFIG_TEMPLATE_STATIC)   
    sequence = sequence + 1
    config = config + configblock(line)
```

Now, the main part. Here the iterative generation is performed:

1. take each line from the DataFileReader - i.e. each line of the table in order
2. append sequence number from the counter
3. append any static variables from CONFIG_TEMPLATE_STATIC, replacing those with the same tags
4. increment the sequence counter
5. format new portion of iterative part of config (via configblock(line) ) and append it to the config variable

```python
  config = config + \n + CONFIG_TEMPLATE_ADD_AFTER
```

Next, the tailing part of the config is added (in my case it is an empty string, so just a new line is added).

```python
  with open(r'config.conf','wb') as resultfile:
    resultfile.write(config)
```

Lastly, the config is written to disk into the config.conf file. End of program.

## The full code of this nice iterative config generator

Below I provide the code itself and some links for those who might find this simple script usable.

### The code

configexpand.py

```python
<pre class="lang:python" title="configexpand.py">import csv, time

# to separate code and data, I load configs from this file:
from configtemplate import *

def configblock(dataline):
  config = CONFIG_TEMPLATE_ITERATIVE
  return config.format(**dataline)

def main():
  print(time.strftime('[%d/%m %H:%M:%S]',time.localtime())+' Starting')
  DataFileReader = csv.DictReader(open(r'data.csv'), delimiter=';')
  print(time.strftime('[%d/%m %H:%M:%S]',time.localtime())+' Parameter list extracted. Generating config...')
  sequence = CONFIG_TEMPLATE_COUNTER['offset'] # offset, if needed
  config = CONFIG_TEMPLATE_ADD_BEFORE + \n
  for line in DataFileReader:
    line['sequence'] = str(sequence) # in iterative configs sequence number is often needed
    line.update(CONFIG_TEMPLATE_STATIC)   # that way any static data can be added needed
    sequence = sequence + 1
    config = config + configblock(line)
  config = config + \n + CONFIG_TEMPLATE_ADD_AFTER
  print(time.strftime('[%d/%m %H:%M:%S]',time.localtime())+' Config generated. Writing to config.conf ...')
  with open(r'config.conf','wb') as resultfile:
    resultfile.write(config)
  print(time.strftime('[%d/%m %H:%M:%S]',time.localtime())+' Done.')

if __name__ == '__main__': 
  main()  
  #EOF
```

configtemplate.py

```python
# example config generates dial-peers for Cisco VoIP gateways:
CONFIG_TEMPLATE_ITERATIVE = dial-peer voice {sequence} voip
 description {description}
 preference 1
 destination-pattern {pattern}
 session protocol sipv2
 session target ipv4:{ip}
 dtmf-relay rtp-nte
 codec g711alaw
 no vad


CONFIG_TEMPLATE_COUNTER = dict(offset = 42 )# offset, if needed

# 
# this way any static data can be added if needed:
CONFIG_TEMPLATE_STATIC = dict(ip='192.0.2.42') # TEST-NET-1 per RFC5737 https://tools.ietf.org/html/rfc5737 


CONFIG_TEMPLATE_ADD_BEFORE = 

CONFIG_TEMPLATE_ADD_AFTER = 
```

### The links

The script is on GitHub in my repository for misc. scripts and other stuff: [https://github.com/askbow/cloaked-octo-ironman/tree/master/configexpand](https://github.com/askbow/cloaked-octo-ironman/tree/master/configexpand)

There, the scripts and example data files are provided.

## Conclusion

In this post I described a simple Python script used to generate iterative configs. The script is pretty simple and can be easily adapted for other similar tasks. I hope this script will be useful for other people as well.