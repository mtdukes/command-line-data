## Hands on: Data analysis in the command line

*IRE 2017 // Phoenix, Arizona // June 23, 11:30 a.m. // Pinnacle Peak 1*

It may not have a fancy interface, but the command line can equip journalists with tailor-made tools to analyze data. This hands-on session will build upon your basic command line skills and dive deeper into the csvkit suite of utilities to interview data, mash it up with other sources, analyze it and generate story ideas -- all with the speed and power of the command line.

**This session is good for:** People who have some basic Excel skills and a little bit of command-line experience (example: you went to the [“Command Line for Reporters”](http://ire.org/events-and-training/event/2703/3339/) session preceding this one)


#### What you'll need

**NOTE:** If you're on one of the IRE machines in Pinnacle Peak 1, what you need has been preloaded.

* [CSVkit](http://csvkit.readthedocs.io/en/latest/index.html), a suite of command-line tools for converting to and working with CSV, the king of tabular file formats
* Terminal, Apple's built-in command-line utility
* [Cambridge, Mass., snow ticket data](/snow_tickets.csv)
* [Cambridge, Mass., SeeClickFix snow removal data](/snow_complaints.csv)
* [Census block lookup table](/census_lookup.csv)

#### What you'll learn

* How to use CSVkit to work with data

#### A primer

If you haven't already, check out AJ Vicen's [excellent tutorial on using his favorite command-line tools](https://github.com/AJVicens/favcommandlinetools), which will provide the introduction to this course.

Let's begin by reminding ourselves where we are in your machine's file structure with the ```pwd``` command. You should get a result like this:


```bash
$ pwd
/Users/mtdukes/Desktop
```

**NOTE:** the ```$``` sign you may see in the instructions below is just the Terminal prompt, so don't enter it as a command. Your prompt may look slightly different:

We need to navigate to our project folder with ```cd```. This may be a slightly different process on your machine.

**NOTE:** If you're starting this tutorial from scratch or you're not using an IRE machine, you can clone the GitHub repository with:

```bash
git clone [directory]
```

#### Basic terminology

CSVkit contains a suite of **commands** we can use to work with data. **Flags** used in conjunction with these commands determine exactly how they'll function, and **parameters** will detail the file names and/or the columns and other features of the data we want to work with. We can chain multiple commands together using the ```|```, or **pipe**, character.

So with a command like this...

```bash
csvsort -c off_city snow_tickets.csv | csvlook
```
...means we're using the ```-c``` flag with the ```csvsort``` command to sort the data by the ```off_city``` column of the ```snow_report20170321fix.csv``` file. Then, using a ```|``` character, we're chaining that command with ```csvlook``` to makes the data readable in the Terminal window.

It's not necessary to memorize the individual flags, since you can look them up in the [CSVkit reference documentation](http://csvkit.readthedocs.io/en/latest/cli.html#reference) any time!

#### About the data

It's 110 degrees outside, so let's think cool.

![Think cold thoughts](https://dl.dropboxusercontent.com/u/49960384/gifs/snow-nook.gif "Think cold thoughts")

This workshop will use two spreadsheets, both obtained via public records request from the city of Cambridge, Mass., in winter 2017. The first is a listing of every fine issued by the city to violators of its ice and snow removal ordinance, which requires residents to shovel their sidewalks. The second is the database of complaints about slippery sidewalks submitted to the city's online SeeClickFix portal.

#### Taking a closer look

Let's take a closer look at the data we've got by examining a quick snapshot.

```bash
csvstat snow_tickets.csv
```

We can learn a few different things from the ```csvstat``` command, which autodetects the type of data in each field and performs a few basic calculations for each. Here, we learn a few things:

* There are 559 rows
* The ```ticket_number``` field is a unique identifier
* The ```ticket_type``` field only has one unique value -- "Snow & Ice" -- so it's not of much use to us
* Of the most repeat violators, the highest is three or four tickets
* The sum of the ```total_fine``` field shows the city has assessed **$27,900 in fines** during this two-and-a-half month period
* The ```off_city``` field has some dirty data that throws off the count because of extra spaces and other errata

So we've got some basic tasks ahead of us, including cleaning the data to make it a little easier to work with, as well as verifying some of the information we might want to use in our reporting.

First, let's cut out the ```ticket_type``` field with the ```csvcut``` command, and save the output to a new file. The ```-C``` flag (note the capital letter) tells the command we want every column _except_ ```ticket_type```.

We can also omit the extra spaces with a special flag, ```-S```.

Put everything together, and you've got the following command:

```
csvcut -C ticket_type -S snow_tickets.csv > snow_tickets_clean.csv
```

We can verify we've cut out that column by printing out the first 10 rows with:

```
csvlook snow_tickets_clean.csv | head | less -S
```

You can escape this view by pressing ```q```.

Now let's go back to that $27,900 figure we calculated before. You may have noticed one of the field contains broad categories about the status of the tickets the city has issued in the ```current_action``` field. We can look specifically at that field using ```csvstat``` with a column flag.

```
csvstat -c current_action snow_tickets_clean.cs
```

I can see there are 10 unique values here, so I want to see them all to make sure I understand the data. It might help me ask better questions later. We can use two other flags -- ```--freq``` and ```--freq-count``` -- to pull that information out.

```
csvstat -c current_action --freq --freq-count 10 snow_tickets_clean.csv
```

Aha! There are a few duplicates in here, so let's see how they add up.

```
csvgrep -c current_action -m Duplicate snow_tickets_clean.csv | csvstat -c total_fine
```

The data include $1,000 in duplicate fines, which would have thrown off our analysis. Let's remove them and save our deduplicated data to a new file.

```
csvgrep -c current_action -m Duplicate --invert-match snow_tickets_clean.csv > snow_tickets_deduped.csv
```

We can verify that we've removed those rows with

```
csvstat snow_tickets_deduped.csv
```

![Yes!](https://dl.dropboxusercontent.com/u/49960384/gifs/yes-jlaw.gif "Yes!")


Success! We can now more confidently report that the city has assessed **$26,900** in fines and dig in deeper to the other categories with more reporting.

### Joining data

CSVkit gives us the ability to join tables so we can answer bigger, more important questions. One basic question we might have: Which areas seem to get the most tickets?

One way to do that is to add in Census blocks. I've prepared a lookup table [using a U.S. Census geocoding tool](https://geocoding.geo.census.gov/geocoder/geographies/addressbatch?form), which you can download into your folder by right clicking here:

[[Download the file here]]

It contains all the unique addresses in our snow report dataset, along with the Census tracts and block numbers.

```
csvstat census_lookup.csv
```

By examining the file with ```csvstat``` we can see that both our snow report file and our lookup table have a ```location``` field. To join the tables, we'll use the ```csvjoin``` command, specifying the column by the ```location``` field.

```
csvjoin -c location snow_tickets_deduped.csv census_lookup.csv > snow_tickets_blocks.csv
```

By running ```csvstat``` on our new file, we can see that a couple of Census blocks top the list with the most tickets. We can break that down even further by looking specifically at the top-10 most ticketed blocks.

```
csvstat -c block_tract --freq --freq-count 10 snow_tickets_blocks.csv
```

What's going on here? Let's filter our table by the top-offender -- tract/block 3535002002 -- and see if we can look more closely. We can type some of those addresses [into Google Maps](https://www.google.com/maps/place/579+Franklin+St,+Cambridge,+MA+02139/@42.3682558,-71.1120149,18z/data=!4m5!3m4!1s0x89e3775bac5ce3bb:0x4ff199f0c7070371!8m2!3d42.3684286!4d-71.1123539) and narrow down our search area.