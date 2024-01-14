# AI Generated Book Reviews

**Overview:**

Amazon recently launched a new service where [AI models summarize product reviews](https://www.aboutamazon.com/news/amazon-ai/amazon-improves-customer-reviews-with-generative-ai).

In this lab, you will create your own version of this service.
In particular, you will use AI to summarize book reviews scraped from the website <https://www.goodreads.com/>.
This dataset contains all activity on this website between 2006-2017.
It's approximately 30GB of data.
15.7 million user reviews of 2.3 million books.

<!--
Dataset URL at https://mengtingwan.github.io/data/goodreads.html from 2017

https://datarepo.eng.ucsd.edu/mcauley_group/gdrive/goodreads/goodreads_reviews_dedup.json.gz
-->

**Learning Objectives:**

1. give you an overview of the bigdata course,
1. review how to work with csv and json files,
1. review basic python, terminal, and sql programming.

## Part 2: Generate Review Summaries

We will now shift gears and focus on generating our review summaries.
To do this, we will use the file `goodreads_reviews_dedup.json.gz` which contains 2.3 million user reviews of books.

## Exploring `goodreads_books.json.gz`

Whenever you're working with a new file,
it's always a good practice to first measure how large the file is:
```
$ du -h /data-fast/goodreads/goodreads_reviews_dedup.json.gz
5.1G	/data-fast/goodreads/goodreads_reviews_dedup.json.gz
```
In the first part of this lab, you completed an exercise to count the number of lines in this file.
You should have written a command that looks something like:
```
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | wc -l
15739967
```
That's 15.7 million reviews that we'll need to sort through.

Another good practice is to always visually inspect the files you're working with.
Recall that we saw before how to use the `zcat` and `head` commands to manually view the first few lines of a csv file.
The same commands will work to visualize a json file,
but the output can be a bit overwhelming.

Try running the following command:
```
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | head
```
You should get a huge wall of text that is difficult to read.

In order to understand this file better, we'll need to clean up the output a bit using two new techniques:

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
<!--
FIXME:

Another useful tool when working with json files is the `jq` command.
`jq` let's us extract individual fields from the json input.
-->

Notice that the title of the book is not stored in the JSON object.
Instead, the JSON object in `goodreads_reviews_dedup.json.gz` stores a field called `book_id`.
Another file `goodreads_books.json.gz` associates each `book_id` with information about the book, like the title, author, and publication year.
In order to figure out which book reviews correspond with which titles,
we will have to *join* the `goodreads_reviews_dedup.json.gz` and `goodreads_books.json.gz` files together.
Joining datasets is a notoriously difficult and time consuming process,
and we will spend a considerable amount of time discussing in this class how to do so correctly and efficiently.
We will use a simple, manually, and very inefficient process in this lab.

## Exporing `goodreads_books.json.gz`

The first step is to familiarize ourselves with the `goodreads_books.json.gz` file.

> **Exercise:**
> Recall that three good steps to familiarize yourself with a new file are: (1) check the size of a file, (2) count the number of lines in the file, and (3) and manually inspect that file.
> Write one line bash commands to complete each of these tasks.
> (Use the commands above as a guide.)

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
Our goal is to search through the `goodreads_books.json.gz` to find the `book_id` that corresponds to *The Name of the Wind*,
then we will search through `goodreads_reviews_dedup.json.gz` to find the corresponding reviews.

We can search through files in bash using the `grep` command.
`grep` takes a regular expression as a command line arg, and removes all lines in its input that don't satisfy that regular expression.
For example, the command 
```
$ zcat /data-fast/goodreads/goodreads_books.json.gz | grep '"title": "The Name of the Wind"'
```
will find all lines in the json file that contain the string `"The Name of the Wind"` anywhere on the line.

> **NOTE:**
>
> Pay close attention to the quotation marks in the previous paragraph.
> We pass the string `'"title": "The Name of the Wind"' to the `grep` command (with single quotes `'`),
> and it finds lines containing the string `"title": "The Name of the Wind"` (without single quoates `'`).
> 
> Recall that `grep` works only at the text level and does not understand the json semantics.
> Our query string above is designed to find the key `title` with the value `The Name of the Wind`.

This command takes a long time to run,
and we will need the output of this command several times.
To avoid re-running this command many times, we can save the output into a file using the output redirection operator '>'.
Run the command
```
$ zcat /data-fast/goodreads/goodreads_books.json.gz | grep '"title": "The Name of the Wind"' > notw.json
```
This will save the output into a file named `notw.json`.

Now let's count the number of lines in this output
```
$ cat notw.json | wc -l
4
```
It turns out that there are 4 different entries in `goodreads_books.json.gz` for the title `*The Name of the Wind*, each with their own `book_id`.
These 4 entries correspond to different editions of the book (there's a hardcover, a softcover, an audiobook, and a "Tenth Anniversary Edition" hardcover).

We will now introduce the `jq` command for extracting information out of json objects.
For example, we can extract the title from each of our books with the command
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

In our task, we want to summarize how all readers of the book feel about the book, regardless of edition.
So we want reviews of all 4 of these books.
To get these reviews, we'll need to extract the `book_id` field:
```
$ cat notw.json | jq '.book_id'
"17353642"
"18741780"
"12276953"
"34347493"
```

## Doing the Join

To complete our join, we need to find all of the lines in the `goodreads_reviews_dedup.json.gz` file where the `book_id` field is one of the 4 `book_id`s above.
`grep` is our tool of choice anytime we are filtering files on the command line,
so we need to write a `grep` command to complete this join.

One way of doing that would be to write 4 grep commands, one for each of the `book_id`s.
These commands would look like
```
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | grep '"book_id": "17353642"'
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | grep '"book_id": "18741780"'
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | grep '"book_id": "12276953"'
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | grep '"book_id": "34347493"'
...
```
But this is inefficient because each of these commands loops over the dataset separately.

Better is to take advantage of `grep`'s regular expression  ability to combine all of our search criteria into a single regex, and a single call to `grep`.
Recall that in 
```
$ zcat /data-fast/goodreads/goodreads_reviews_dedup.json.gz | grep -E $(echo $(cat notw.json | jq '.book_id') | tr ' ' '|')
```
The following incantation constructs that regex for us
```
$ echo $(cat notw.json | jq '.book_id') | tr ' ' '|'
"17353642"|"18741780"|"12276953"|"34347493"
```

### Part 2b: The Mistral Large Language Models

Modern AIs like ChatGPT are more generally called *large language models* or *LLMs*.
In this section we will see how to programmatically use these LLMs.
Then in the next section we will combine these LLMs with the reviews extracted in the previous sections.

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

Get started by using the `wget` command to download the Mistral LLM llamafile:

```bash
$ wget 'https://huggingface.co/jartine/Mistral-7B-Instruct-v0.2-llamafile/resolve/main/mistral-7b-instruct-v0.2.Q5_K_M.llamafile'
```

You should be averaging speeds over 150MB/s.
The Claremont Colleges have a very fast internet connection,
and your normal download speeds on your laptops are limited by the wifi bandwidth to only about 1MB/s.
The lambda server, however, connected by physical ethernet cables to the campus network and so is not limited by wifi bandwidth.
This is another reason to prefer working on remote computers whenever possible.

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

Press `^C` to end the program and return to the command prompt.

We can pass the `-f` flag to our llamafile in order to specify that the input should come from a file instead of the web interface.
We will again use the special `/dev/stdin` file to enable piping.
```
$ echo "[INST]Tell me about big data.[/INST]" | ./mistral-7b-instruct-v0.2.Q5_K_M.llamafile -f /dev/stdin -c 0
```

### Part 2c: Putting it all together

```
$ echo "[INST]This prompt contains several book reviews in JSON format. Write a short 1 paragraph summary that combines all of these reviews into a single summary of the book. $(cat ./notw.reviews.json | tail -n10 )[/INST]" | ./mistral-7b-instruct-v0.2.Q5_K_M.llamafile -f /dev/stdin -c 0
$ ./mistral-7b-instruct-v0.2.Q5_K_M.llamafile -p "[INST]Write a short summary of the following book reviews: $(cat ./notw.reviews.json | jq '.review_text' | tail -n10 )[/INST]" -c 0
