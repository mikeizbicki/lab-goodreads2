# Lab: AI Review Summaries

Amazon recently launched a new service where [AI models summarize product reviews](https://www.aboutamazon.com/news/amazon-ai/amazon-improves-customer-reviews-with-generative-ai).
For example, here's the AI generated review for [this package of uranium ore](https://www.amazon.com/dp/B000796XXM):

<img src=amazon-review-uranium.png width=100%>

In this lab, you will create your own version of this service.
You will use AI to summarize book reviews from the website <https://www.goodreads.com/>.
We'll use [a public dataset](https://mengtingwan.github.io/data/goodreads.html) that contains all activity on this website between 2006-2017.
It's approximately 30GB of data, and contains 15.7 million user reviews of 2.3 million books.

The hard part of this project will be to find the reviews for the particular books we're interested amidst all of this data.
To accomplish this, we will review basic shell commands.
<!--
Dataset URL at https://mengtingwan.github.io/data/goodreads.html from 2017

https://datarepo.eng.ucsd.edu/mcauley_group/gdrive/goodreads/goodreads_reviews_dedup.json.gz

**Learning Objectives:**

1. give you an overview of the bigdata course,
1. review how to work with csv and json files,
1. review basic python, terminal, and sql programming.
-->

## Part 0: Exploring the reviews.

In this lab, we're going to dive right into the full dataset.
The file `/data-fast/goodreads/goodreads_reviews_dedup.json.gz` contains the full set of reviews.
Whenever you're working with a new file,
there's three common tasks that you should always do:
(a) measure the size of the file,
(b) count the number of data points,
(c) manually inspect the data.
Let's see how to do each of these in turn.

### Part 0.a: File size

The "standard" way to get the size of a file is with the `ls -l` command.
(The `-l` stands for displaying the "long" format.)
```
$ ls -l /data-fast/goodreads/goodreads_reviews_dedup.json.gz
-rw-rw-r-- 1 mizbicki mizbicki 5381053528 Oct 30 08:48 /data-fast/goodreads/goodreads_reviews_dedup.json.gz
```
In the output above,
the number `5381053528` is the number of bytes in the file.
This is not a particularly human-readable number, however, so I usually prefer to use the `du -h` command.
(`du` stands for "disk usage", and `-h` stands for "human readable output".)
This gives the result:
```
$ du -h /data-fast/goodreads/goodreads_reviews_dedup.json.gz
5.1G	/data-fast/goodreads/goodreads_reviews_dedup.json.gz
```
This is large enough that we won't be able to easily work with the file in memory and will need to use $O(1)$ memory algorithms.

### Part 0.b: Number of Lines

Our file is in [JSON lines](https://jsonlines.org/) format,
where each line has a single JSON object that represents one data point.
Therefore, counting the number of lines will tell us the number of book reviews in our dataset.

In the first part of this lab,
you already completed an exercise to count the number of lines in this file.
You should have written a command that looks something like:
```
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | wc -l
15739967
```
That's 15.7 million reviews.

We won't want to be passing all of these reviews into an AI model at once.
Instead, we'll need to search through these 15.7 million reviews to find just the subset of reviews about the book we're writing a summary for.

### Part 0.c: Inspect the Data

The most important task when confronting any new file is to manually inspect your file.
I've too many students (and professional data scientists!) skip this step.
This results in them writing code for data that they don't understand,
which makes the code not work,
which means they've wasted hours of work.
SO ALWAYS MANUALLY INSPECT YOUR DATA BEFORE DOING ANY ANALYSIS!!!

Recall that we saw before how to use the `zcat` and `head` commands to manually view the first few lines of a csv file.
The same commands will work to visualize a json file,
but the output can be a bit overwhelming.

Try running the following command:
```
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | head
```
You should get a huge wall of text that is difficult to read.

In order to understand this data better,
we'll need to clean up the output a bit using two new techniques:

1. Passing the `-n1` parameter to the head command.
    (This tells `head` to print only the first line of its input.
    Since this data file has one line per data point, this will result in printing the first data point.)

1. Piping the data point to python's `json.tool` pretty printer.
    (`json.tool` is a python module, and we can activate any python module from the command line with the command `python3 -m modulename`.)

Putting it all together, we get the incantation
```
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | head -n1 | python3 -m json.tool
```
Which gives output like
```
{
    "user_id": "8842281e1d1347389f2ab93d60773d4d",
    "book_id": "24375664",
    "review_id": "5cd416f3efc3f944fce4ce2db2290d5e",
    "rating": 5,
    "review_text": "Mind blowingly cool. Best science fiction I've read in some time. I just loved all the descriptions of the society of the future - how they lived in trees, the notion of owning property or even getting married was gone. How every surface was a screen. \n The undulations of how society responds to the Trisolaran threat seem surprising to me. Maybe its more the Chinese perspective, but I wouldn't have thought the ETO would exist in book 1, and I wouldn't have thought people would get so over-confident in our primitive fleet's chances given you have to think that with superior science they would have weapons - and defenses - that would just be as rifles to arrows once were. \n But the moment when Luo Ji won as a wallfacer was just too cool. I may have actually done a fist pump. Though by the way, if the Dark Forest theory is right - and I see no reason why it wouldn't be - we as a society should probably stop broadcasting so much signal out into the universe.",
    "date_added": "Fri Aug 25 13:55:02 -0700 2017",
    "date_updated": "Mon Oct 09 08:55:59 -0700 2017",
    "read_at": "Sat Oct 07 00:00:00 -0700 2017",
    "started_at": "Sat Aug 26 00:00:00 -0700 2017",
    "n_votes": 16,
    "n_comments": 0
}
```
The JSON object above represents the first review in our dataset.
This object contains a variety of information about the review,
but notice that the title of the book is not stored in the JSON object.
The only information we have about the book is the `book_id` field.
A different file `goodreads_books.json.gz` associates each `book_id` with information about the book, like the title, author, and publication year.
<!--
Storing this information in a separate file is called *normalizing* the dataset.
We will talk later in this class about the various advantages of normalizing and denormalizing datasets,
but the main idea is that normalized datasets can save a lot of disk space by not repeating the same information multiple times.
For example, we will see in a bit that goodreads stores a lot of information about each book.
In addition to the title, there is 
-->
In order to figure out which book reviews correspond with which titles,
we will have to *join* the `goodreads_reviews_dedup.json.gz` and `goodreads_books.json.gz` files together.
Joining datasets is a notoriously difficult and time consuming process.
We will spend a considerable amount of time in this class discussing how to join correctly and efficiently.
For this lab, we will use a simple, manual, and slow method.
Later in the course, we'll learn more complicated, automatic, and faster methods.

## Part 1: Convert a book id into a title.

The first step is to familiarize ourselves with the `goodreads_books.json.gz` file.

> **Exercise:**
>
> Recall that three good steps to familiarize yourself with a new file are: (1) check the size of a file, (2) count the number of lines in the file, and (3) and manually inspect that file.
> Write one line bash commands to complete each of these tasks.
> You can use the commands from Part 0 above as a guide.

<!--
```
$ du -h /data-fast/goodreads/goodreads_books.json.gz
2.1G	/data-fast/goodreads/goodreads_books.json.gz
$ zcat /data-fast/goodreads/goodreads_books.json.gz | wc -l
2360655
$ zcat /data-fast/goodreads/goodreads_books.json.gz | head -n1 | python3 -m json.tool
{
    "isbn": "0312853122",
    "text_reviews_count": "1",
    "series": [],
    "country_code": "US",
    "language_code": "",
    "popular_shelves": [
        {
            "count": "3",
            "name": "to-read"
        },
        {
            "count": "1",
            "name": "p"
        },
        {
            "count": "1",
            "name": "collection"
        },
        {
            "count": "1",
            "name": "w-c-fields"
        },
        {
            "count": "1",
            "name": "biography"
        }
    ],
    "asin": "",
    "is_ebook": "false",
    "average_rating": "4.00",
    "kindle_asin": "",
    "similar_books": [],
    "description": "",
    "format": "Paperback",
    "link": "https://www.goodreads.com/book/show/5333265-w-c-fields",
    "authors": [
        {
            "author_id": "604031",
            "role": ""
        }
    ],
    "publisher": "St. Martin's Press",
    "num_pages": "256",
    "publication_day": "1",
    "isbn13": "9780312853129",
    "publication_month": "9",
    "edition_information": "",
    "publication_year": "1984",
    "url": "https://www.goodreads.com/book/show/5333265-w-c-fields",
    "image_url": "https://images.gr-assets.com/books/1310220028m/5333265.jpg",
    "book_id": "5333265",
    "ratings_count": "3",
    "work_id": "5400751",
    "title": "W.C. Fields: A Life on Film",
    "title_without_series": "W.C. Fields: A Life on Film"
}
```
-->

You should see a large number of JSON fields,
but for our purposes the most important are the `title` and `book_id` fields.
In order to find the title of the book that corresponds to the book review we found above,
we need to search through the entire `goodreads_books.json.gz` file for a line with the appropriate `book_id` and extract the `title` field from this entry.
We will use the `grep` tool to search and a new command `jq` to extract.

### Part 1.a: Searching with `grep`

<!--
Our goal is to search through the `goodreads_books.json.gz` to find the `book_id` that corresponds to *The Name of the Wind*,
then we will search through `goodreads_reviews_dedup.json.gz` to find the corresponding reviews.
-->

`grep` is the standard tool for searching files on the command line.

> **Note:**
>
> There are many competing implementations of the `grep` tool.
> Linux machines traditionally use the GNU grep implementation,
> which is particularly efficient.
> GNU grep uses the [Boyer-Moore](https://en.wikipedia.org/wiki/Boyer%E2%80%93Moore_string-search_algorithm) string searching algorithm,
> which has the nice property that it doesn't even need to examine every byte in the input stream!
> The author has a [mailing list post from 2010](https://lists.freebsd.org/pipermail/freebsd-current/2010-August/019310.html) where he describes the implementation details that I recommend (but don't require) you to read.

`grep` has no knowledge of JSON and works on any text file.
It takes a regular expression as a command line argument,
and removes all lines in its input that don't satisfy that regular expression.
In order to find the book with id `24375664`,
we will use the following regular expression to match the way this information is stored in the JSON object: `"book_id": "24375664"`.
The final incantation is
```
$ zcat /data-fast/goodreads/goodreads_books.json.gz | grep '"book_id": "24375664"'
```
The command above should output a very large JSON object.

> **NOTE:**
>
> Pay close attention to the quotation marks in the previous paragraph.
> Since the regex contains a space, it must be enclosed in quotation marks for the entire regex to be considered a single parameter to `grep`.

> **Note:**
>
> `grep` and `zcat` are *streaming* tools.
> This means they output their results as they find them, not when the program ends.
> You will therefore see the JSON object get printed to the screen before the programs end and return control to the terminal.
> You will know that the programs have ended when the command prompt `$` is printed to the screen.
> Streaming tools can be confusing at first, but they are the reason for the $O(1)$ memory performance.

### Part 1.b: Parsing the JSON

The JSON object associated with each book is large.
Even if we use `python3 -m json.tool` to pretty print that output,
it will be annoying to manually search through the entire object to find the `title` field.
We will use the `jq` command to extract the title field for us automatically.

`jq` is a popular command line tool for parsing JSON objects,
but it has a [notoriously complicated syntax](https://jqlang.github.io/jq/manual/).
Fortunately extracting just a single field from the JSON object is not too bad:
all you have to do is pass the field name prepended with a dot.
For example, `.title` will extract the `title` field from the json object and print it to the screen.

Combining `zcat`, `grep`, and `jq` gives us the following 1-liner for finding our book title.
```
$ zcat /data-fast/goodreads/goodreads_books.json.gz | grep '"book_id": "24375664"' | jq '.title'
```
You should get that the title for our first book review was
```
"The Dark Forest (Remembrance of Earth’s Past, #2)"
```

## Part 2: Get the reviews for a title

In the previous sections, we saw how to extract a review from `goodreads_reviews_dedup.json.gz` and then join with the file `goodreads_books.json.gz` to get the title of the book that was reviewed.
What we're really interested in, however, is going in the opposite direction.
We need to start with a title, and then find all of the reviews for that title.
This will turn out to be quite a bit trickier due to some messiness in the data.
As a running example, we will use the best selling fantasy novel *The Name of the Wind* (NOTW for short).

First, we'll see that there are multiple `book_id`s for NOTW.
Then we'll see how to perform our join using multiple `book_id`s at once.

### Part 2.a: Getting the `book_id`(s) for our Title

Consider the following command.
```
$ zcat /data-fast/goodreads/goodreads_books.json.gz | grep '"title": "The Name of the Wind"' | wc -l
```
This counts all of the entries in the `goodreads_books.json.gz` file with a `title` field equal to `The Name of the Wind`.
It turns out that there are 4 different entries in `goodreads_books.json.gz` for the title `*The Name of the Wind*, each with their own `book_id`.
These 4 entries correspond to different editions of the book (there's a hardcover, a softcover, an audiobook, and a "Tenth Anniversary Edition" hardcover).

Our command to count the number of entries took a long time to run because it needed to loop over the entire dataset.
To save time in the future, it is often good practice to store the results of intermediate steps into a file.
We can do this using the output redirection `>` operator.
The following command will take all of the entries for our book and store them in a file called `books-notw.json`.
```
$ zcat /data-fast/goodreads/goodreads_books.json.gz | grep '"title": "The Name of the Wind"' > books-notw.json
```
Now we can quickly count the number of books:
```
$ cat notw.json | wc -l
4
```
And we can do other processing on these books much more efficiently.
For example, we can extract the title from each of these books with the command
```
$ cat notw.json | jq '.title'
```
and notice that they are all the same.
The differences between each entry are in the `format` and `edition_information` fields:
```
$ cat notw.json | jq '.format'
"Audio"
"Paperback"
"Hardcover"
"Hardcover"
$ cat notw.json | jq '.edition_information'
""
""
""
"Tenth Anniversary Edition"
```
And each entry has a different `book_id` field:
```
$ cat notw.json | jq '.book_id'
"17353642"
"18741780"
"12276953"
"34347493"
```
In our task, we want to summarize how all readers of the book feel about the book, regardless of the edition that they read.
Therefore, we'll need to find the reviews for each of these `book_id`s.

### Part 2.b: Computing the Join

To complete our join, we need to find all of the lines in the `goodreads_reviews_dedup.json.gz` file where the `book_id` field is one of the 4 `book_id`s above.
As mentioned earlier, `grep` is the tool of choice anytime we are filtering files on the command line,
so we need to write a `grep` command to complete this join.

One way of doing that would be to write 4 grep commands (one for each of the `book_id`s),
using output redirection to concatenate the results together.
```
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | grep '"book_id": "17353642"' >> reviews-notw.json
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | grep '"book_id": "18741780"' >> reviews-notw.json
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | grep '"book_id": "12276953"' >> reviews-notw.json
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | grep '"book_id": "34347493"' >> reviews-notw.json
```
But this is inefficient because each of these commands loops over the dataset separately,
and it's awkward to have to manually repeat all of these commands.
**You don't need to run the commands above.
The next paragraphs will introduce a more efficient method for getting the same answer.**

It is more efficient to take advantage of `grep`'s regular expression ability to combine all of our search criteria into a single regex, and a single call to `grep`.
We'll use regular expression [alternations](https://www.regular-expressions.info/alternation.html).
Recall that an alternation is expressed syntactically with the pipe operator `|` but has the semantic meaning of "or" inside of a regular expression.
Thus, the regular expression
```
"17353642"|"18741780"|"12276953"|"34347493"
```
semantically means: find the string `"17353642"` or `"18741780"` or `"12276953"` or `"34347493"`.
This is the regex that we want to use with grep.

The following command will extract all reviews with any of these four book ids.
```
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | grep -E '"17353642"|"18741780"|"12276953"|"34347493"' > reviews-notw.json
```
The file `reviews-notw.json` now contains the output of our join.

> **Note:**
> 
> Manually generating the regex, then copy/pasting it into our shell command is a bit awkward when the number of `book_id`s is large.
> In this note, we'll see how to automate that process.
>
> The following "simple" shell 1-liner generates the regex for finding `book_id`s:
> ```
> $ echo $(cat books-notw.json | jq '.book_id') | sed 's/ /|/g'
> "17353642"|"18741780"|"12276953"|"34347493"
> ```
> This is a rather cryptic incantation, and we'll break it down step-by-step to understand it.
> 
> 1. First we'll take a look at the command substitution `$( ... )`.
>   We've already seen that the `cat notw.json | jq '.book_id'` outputs the 4 `book_id`s each on their own line.
> 1. We pass that result to `echo` which replaces all the newlines with spaces.
>   This has the effect of putting all of the `book_id`s onto a single line.
> 1. Finally, `sed` places the alternation operator `|` between each `book_id`.
>
> It's okay if this command feels like magic to you right now.
> By the end of this semester, writing these 1-line shell commands should be easier for you than doing the copy/paste necessary to write out the regex manually :)
>
> Now, instead of manually copying pasting the regex into the command above,
> we can use command substitution:
> ```
> $ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | grep -E $(echo $(cat books-notw.json | jq '.book_id') | sed 's/ /|/g') > reviews-notw.json
> ```

### Part 2.c: Sanity Checking

We're not quite done.
We still need to sanity check our results.

<!--
> **Note:**
>
> One of the most common traps that students fall into when doing data analysis projects is not sanity checking your results.
That's even more common in big data projects because the results take so long to compute,
that when you finally get a number you're ready to just move onto the next phase of your project.
-->

A simple way to sanity check is to count the number of reviews we've found:
```
$ cat reviews-notw.json | wc -l
14
```

Hmmm... that's not a lot of reviews...

*The Name of the Wind* is a New York Times best selling book that has millions of copies sold.
14 reviews seems WAY too few.

What happened is that there are many more `book_id`s that correspond to *The Name of The Wind* that we didn't find.
It turns out that the `title` field inside of the `goodreads_books.json.gz` dataset is not guaranteed to be *equal* to the title of the book, like we assumed above.
Instead, the `title` field is only guaranteed to *begin* with the title of the book.
It is allowed to contain extra information about the book after the title.
Therefore, when searching for a book, we do not want to perform an exact match against the book title.

Recall that in Part 2.a above, we searched the file `boodreads_books.json.gz` with the following regex:
```
"title": "The Name of the Wind"
```
To find `title` fields that only begin with the title of our book, we make the simple change of removing the trailing `"` to get
```
"title": "The Name of the Wind
```
Now, if there is more information in the `title` field,
this new regex will still match.

With this regex, we will find many more than 4 JSON objects:
```
$ zcat /data-fast/goodreads/goodreads_books.json.gz | grep '"title": "The Name of the Wind' > books-notw-full.json
$ cat books-notw-full.json | wc -l
35
```
We can see that our new command captures several different titles that a human would "obviously" consider to be the same book:
```
$ cat books-notw-full.json | jq '.title'
```
And we can see that these all have many different format/edition combinations:
```
$ cat books-notw-full.json | jq '.format'
$ cat books-notw-full.json | jq '.edition_information'
```

> **Exercise:**
>
> Repeat the join process from Part 2.b with the new set of 35 `book_id`s we found above,
> and store the reviews in a file `reviews-notw-full.json`.
> (You may find the note at the end of Part 2.b particularly helpful.)
>
> Then do a sanity check of counting the number of reviews you found.
> You know you did everything correctly if you get a number a bit less than 6000.

<!--
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | grep -E $(echo $(cat books-notw-full.json | jq '.book_id') | sed 's/ /|/g') > reviews-notw-full.json
$ wc -l reviews-notw-full.json
5992
-->

You won't be able to complete the next section until you've completed the exercise above.

## Part 3: Generating the Review Summaries

*This section to be added later.*

<!--
The Mistral Large Language Models

Modern AIs like ChatGPT are more generally called *large language models* or *LLMs*.
In this section we will see how to programmatically use these LLMs.
Then in the next section we will combine these LLMs with the reviews extracted in the previous sections.

### Part 3.a: Using LLMs from the Command Line

There are many ways to interact with LLMs.
You've probably in the past used web interfaces like <https://chat.openai.com> or <https://bard.google.com/>.
These web interfaces are easy for human interaction, but hard for programs to use,
and so many API toolkits have been developed to interact with these systems from python.
These APIs are still awkward to use for a variety of reasons,
and so in this lab we will be using a command line interface called [llamafile](https://github.com/Mozilla-Ocho/llamafile).
Llamafile is a project developed by Mozilla (the non-profit best known for developing firefox).
The idea of llamafile is that every LLM can be packaged as a single, standalone executable file that can easily be combined with other unix processes using pipes.

**FIXME**
You can find more examples of cool uses of llamafiles at the [developer's webpage](https://justine.lol/oneliners/).

In unix, text is the universal interface, and the [unix philosophy](http://www.catb.org/~esr/writings/taoup/html/ch01s06.html) is that all programs should be written to accept text as input and write text as output.

### Part 3.a: Getting the Llamafile

Get started by using the `wget` command to download the Mistral LLM llamafile:

```bash
$ wget 'https://huggingface.co/jartine/Mistral-7B-Instruct-v0.2-llamafile/resolve/main/mistral-7b-instruct-v0.2.Q5_K_M.llamafile'
```

You should be averaging speeds over 150MB/s.
The Claremont Colleges have a very fast internet connection,
and your normal download speeds on your laptops are limited by the wifi bandwidth to only about 1MB/s.
The lambda server, however, connected by physical ethernet cables to the campus network and so is not limited by wifi bandwidth.
This is another reason to prefer working on remote computers whenever possible.

### Part 3.b: Running the LLamafile

Once the download completes, try running the llamafile with the command
```
$ ./mistral-7b-instruct-v0.2.Q5_K_M.llamafile
```
You should get a `Permission denied` error.
For security reasons, all files that are downloaded from the internet by default do not have execute permissions set.

In this case, we trust the downloaded executable file, and so we can manually give executable permissions with the `chmod` command:
```
$ chmod u+x mistral-7b-instruct-v0.2.Q5_K_M.llamafile
```
Now, when you run the file
```
$ ./mistral-7b-instruct-v0.2.Q5_K_M.llamafile
```
You should get a large amount of debugging output followed by the line
```
llama server listening at http://127.0.0.1:8080
```
By default, llamafile provides a web interface for interacting with the LLM.
But we won't be using that interface, and will instead use the command line interface.

Press `CTRL-C` to end the program and return to the command prompt.

We can pass the `-f` flag to our llamafile in order to specify that the input should come from a file instead of the web interface.
We will use the special file `/dev/stdin` to enable piping into the llamafile.
```
$ echo "[INST]Write 1 paragraph explaining why the shell is important for big data.[/INST]" | ./mistral-7b-instruct-v0.2.Q5_K_M.llamafile -f /dev/stdin
```
You should see a large amount of debug output, followed by en explanation of why the shell is important for big data.

> **Computational Note:**
>
> The command above is extremely compute intensive and will max out the lambda server's CPU usage.
> The command took me 30 seconds to complete when the lambda server was completely idle,
> and will take longer as the lambda server comes under load.
> The overall runtime will be proportional to the number of students currently running the command,
> so if 10 students are running the command,
> expect it to take about 300 seconds to complete.
> Subsequent steps for this lab do not depend on the output of the command above,
> so you can stop the command early (by pressing `CTRL-C`) if you'd like.

The string inside of the echo command above
```
[INST]Write 1 paragraph explaining why the shell is important for big data.[/INST]
```
is called the *prompt* for our large language model.
The prompt provides natural language instructions for what the model should do.
[The Mistral website](https://www.promptingguide.ai/models/mistral-7b#mistral-7b-instruct) has guidelines for how to write good prompts,
but this is still an active area of research.

### Part 2.c: Putting it all together

We're finally ready to generate the AI summary of our book reviews.
All we have to do is (1) write prompt telling the AI what to summarize,
and (2) use the shell to pass that to our LLM.

I recommend using the following prompt:
```
[INST]Write 1 short paragraph that combines all of the following book reviews into a single summary of the book.
The reviews are:
$(cat ./reviews-notw-full.json | head -n20 | jq '.review_text')
[/INST]
```
The purpose of the `head -n20` command above is purely computational.
Changing the 20 to a larger number will result in a better summary that incorporates more reviews,
but at the expense of a longer runtime.
With 20 reviews, the command takes me 4 minutes to run on an un-loaded llambdaserver.

## Submission
-->

<!--
$ echo "[INST]Write 1 short paragraph that combines all of the following book reviews into a single summary of the book. The reviews are: $(cat ./reviews-notw-full.json | head -n20 | jq '.review_text')[/INST]" | ./mistral-7b-instruct-v0.2.Q5_K_M.llamafile -f /dev/stdin

$ ./mistral-7b-instruct-v0.2.Q5_K_M.llamafile -f /dev/stdin <<EOF
[INST]Write 1 short paragraph that combines all of the following book reviews into a single summary of the book. The reviews are:
$(cat ./reviews-notw-full.json | head -n20 | jq '.review_text')
[/INST]"
EOF
-->

<!--
```
$ echo "[INST]This prompt contains several book reviews in JSON format. Write a short 1 paragraph summary that combines all of these reviews into a single summary of the book. $(cat ./notw.reviews.json | head -n20 | jq '.review_text')[/INST]"
```
Notice how I used command substitution to extract a subset of the reviews and 

```
$ echo "[INST]This prompt contains several book reviews in JSON format. Write a short 1 paragraph summary that combines all of these reviews into a single summary of the book. $(cat ./notw.reviews.json | tail -n10 )[/INST]" | ./mistral-7b-instruct-v0.2.Q5_K_M.llamafile -f /dev/stdin -c 0
$ ./mistral-7b-instruct-v0.2.Q5_K_M.llamafile -p "[INST]Write a short summary of the following book reviews: $(cat ./notw.reviews.json | jq '.review_text' | tail -n10 )[/INST]" -c 0
-->
