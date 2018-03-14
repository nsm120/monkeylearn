---
title: MonkeyDo
author:
  name: Amanda Dobbyn
  email: amanda.e.dobbyn@gmail.com
  theme: yeti
output:
  html_document:
    keep_md: true
    toc: false
    theme: yeti
  github_document:
    toc: false
---





<br>

This is a story mostly about how I started contributing to the rOpenSci package [monkeylearn](https://github.com/ropensci/monkeylearn).

### Some Backstory

Things started at work, when I was looking around for an easy way to classify groups of products using R. I made the very clever first move of Googling "easy way to classify groups of products using R" and thanks to the magic of what I suppose used to be PageRank I landed upon a GitHub README for a package called monkeylearn. 

A quick `devtools::github_install("ropensci/monkeylearn")` and creation of an API key later this seemed like the package to fit my needs. I loved that it sported only two functions, `monkeylearn_classify()` and `monkeylearn_extract()`, which did exactly what they said on the tin. They accept a vector of texts and return a dataframe of classifications or keyword extractions, respectively.

<img src = "./monkeylearn_api.png" style="height: 450px">

For a bit of background, the `monkeylearn` package hooks into the [MonkeyLearn API](https://monkeylearn.com/api/), which uses natural language processing techniques to take a text input and hands back a vector of outputs (keyword extractions or classifications) along with metadata such as their confidence in relevance of the classification. There are a set of built-in "modules" (e.g., retail classifier, profanity extractor) but users can also create their own "custom" modules [^1] by supplying their own labeled training data.

I began using the package to attach classifications to around 70,000 texts. I soon discovered a major stumbling block for my particular use case: I could not send texts to the MonkeyLearn API in batches. This wasn't because the `monkeylearn_classify()` and `monkeylearn_extract()` functions themselves didn't accept multiple inputs. Instead, it was because they didn't explicitly *relate* inputs to outputs. This became a problem because inputs and outputs are not 1:1; if I send a vector of three texts for classification, my output dataframe might be 10 rows long. However, there was no way to know whether the first two or the first four output rows, for example, belonged to the first input text.

Here's an example of what I mean.



```r
texts <- c(
    "In a hole in the ground there lived a hobbit.",
    "It is a truth universally acknowledged, that a single man in possession of a good fortune, must be in want of a wife.",
    "When Mr. Bilbo Baggins of Bag End announced that he would shortly be celebrating his eleventy-first birthday with a party of special magnificence, there was much talk and excitement in Hobbiton.")

(texts_out <- monkeylearn_classify(texts) %>% knitr::kable())
```



 category_id   probability  label                text_md5                         
------------  ------------  -------------------  ---------------------------------
    18313280         0.071  Music                b48a6fe941a1bafee6af3b43a0467bbe 
    18313502         0.054  Music DVD's          b48a6fe941a1bafee6af3b43a0467bbe 
    18313524         0.553  See All Music DVDs   b48a6fe941a1bafee6af3b43a0467bbe 
    18314767         0.062  Books                af55421029d7236ca6ecbb2819e18137 
    18314954         0.047  Mystery & Suspense   af55421029d7236ca6ecbb2819e18137 
    18314957         0.102  Police Procedural    af55421029d7236ca6ecbb2819e18137 
    18313210         0.082  Party & Occasions    602f1ab2654b88f5c7f5c90e42d1ca7a 
    18313231         0.176  Party Supplies       602f1ab2654b88f5c7f5c90e42d1ca7a 
    18313235         0.134  Party Decorations    602f1ab2654b88f5c7f5c90e42d1ca7a 
    18313236         0.406  Decorations          602f1ab2654b88f5c7f5c90e42d1ca7a 

This works great if you don't care about classifying your inputs independently of one another. (Say, you're interested in classifying a whole chapter of a book.) In my case, though, my inputs were independent of one another and each had to be classified separately.


### Initial Workaround

My first approach to this problem was to simply treat each text as a seaparate call. I wrapped `monkeylearn_classify()` in a function that would send a vector of texts and return a dataframe relating the input in one column to the output in the others. Here is a simplified version of it, sans the error handling and other bells and whistles:



```r
initial_workaround <- function(df, col) {
  
  quo_col <- enquo(col)
  
  out <- df %>% 
    mutate(
      tags = NA_character_
    )
  
  for (i in 1:nrow(df)) {
    this_text <- df %>% select(!!quo_col) %>% slice(i) %>% as_vector()
    this_classification <- monkeylearn_classify(this_text) %>% select(-text_md5) %>% list()
    out[i, ]$tags <- this_classification
  }

  return(out)
}
```

Since `initial_workaround()` takes a dataframe as input rather than a vector, let's turn our sample into a tibble before feeding it in.


```r
texts_df <- tibble(texts)
```

And now we'll run the workaround:


```r
initial_out <- initial_workaround(texts_df, texts)

initial_out %>% knitr::kable()
```



texts                                                                                                                                                                                                tags                                                                                                                                   
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------  ---------------------------------------------------------------------------------------------------------------------------------------
In a hole in the ground there lived a hobbit.                                                                                                                                                        18313280, 18313502, 18313524, 0.071, 0.054, 0.553, Music, Music DVD's, See All Music DVDs                                              
It is a truth universally acknowledged, that a single man in possession of a good fortune, must be in want of a wife.                                                                                18314767, 18314954, 18314957, 0.062, 0.047, 0.102, Books, Mystery & Suspense, Police Procedural                                        
When Mr. Bilbo Baggins of Bag End announced that he would shortly be celebrating his eleventy-first birthday with a party of special magnificence, there was much talk and excitement in Hobbiton.   18313210, 18313231, 18313235, 18313236, 0.082, 0.176, 0.134, 0.406, Party & Occasions, Party Supplies , Party Decorations, Decorations 


We see that this retains the 1:1 relationship between input and output, but still allows the output list-col to be unnested. 


```r
(initial_out %>% unnest() %>% knitr::kable())
```



texts                                                                                                                                                                                                 category_id   probability  label              
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------  ------------  ------------  -------------------
In a hole in the ground there lived a hobbit.                                                                                                                                                            18313280         0.071  Music              
In a hole in the ground there lived a hobbit.                                                                                                                                                            18313502         0.054  Music DVD's        
In a hole in the ground there lived a hobbit.                                                                                                                                                            18313524         0.553  See All Music DVDs 
It is a truth universally acknowledged, that a single man in possession of a good fortune, must be in want of a wife.                                                                                    18314767         0.062  Books              
It is a truth universally acknowledged, that a single man in possession of a good fortune, must be in want of a wife.                                                                                    18314954         0.047  Mystery & Suspense 
It is a truth universally acknowledged, that a single man in possession of a good fortune, must be in want of a wife.                                                                                    18314957         0.102  Police Procedural  
When Mr. Bilbo Baggins of Bag End announced that he would shortly be celebrating his eleventy-first birthday with a party of special magnificence, there was much talk and excitement in Hobbiton.       18313210         0.082  Party & Occasions  
When Mr. Bilbo Baggins of Bag End announced that he would shortly be celebrating his eleventy-first birthday with a party of special magnificence, there was much talk and excitement in Hobbiton.       18313231         0.176  Party Supplies     
When Mr. Bilbo Baggins of Bag End announced that he would shortly be celebrating his eleventy-first birthday with a party of special magnificence, there was much talk and excitement in Hobbiton.       18313235         0.134  Party Decorations  
When Mr. Bilbo Baggins of Bag End announced that he would shortly be celebrating his eleventy-first birthday with a party of special magnificence, there was much talk and excitement in Hobbiton.       18313236         0.406  Decorations        


But, the catch: this appraoch was quite slow. The real bottleneck here isn't the for loop; it's that this requires a round trip to the MonkeyLearn API for each individual text. For just these three meager texts, let's see how long `initial_workaround()` takes to finish.


```r
system.time(initial_workaround(texts_df, texts))
```

```
##    user  system elapsed 
##   0.039   0.000   9.445
```

It was clear that even classifying my relatively small data was going to take a looong time 🙈. I updated the function to write each row out to an RDS file after it was classified inside the loop (with an addition along the lines of `write_rds(out[i, ], glue::glue("some_directory/{i}.rds"))`) so that I wouldn't have to rely on the funciton successfully finishing execution in one run. Still, I didn't like my options.

This classification job was intended to be run every night, and with an unknown amount of input text data coming in every day, I didn't want it to run for more than 24 hours one day and either a) prevent the next night's job from running or b) necessitate spinning up a second server to handle the next night's data.


### Diving In

Now that I'm starting to think 

<img src = "./theresgottabeabetterway.gif" style="margin-left: 10%">

I'm just about at the point where I have to start making myself useful.

I'd seen in the package docs and on the [MonkeyLearn FAQ](http://help.monkeylearn.com/frequently-asked-questions/queries/can-i-classify-or-extract-more-than-one-text-with-one-api-request) that batching up to 200 texts was possible[^2]. So, I decide to first look into the mechanics of how text batching is done in the `monkeylearn` package.

Was the MonkeyLearn API returning JSON that didn't relate each input individual and output? I sort of doubted it. You'd think that an API that was sent a JSON "array" of inputs would send back a hierarchical array to match. My huch was that either the package was concatenating the input before shooting it off to the API (which *would* save user on API queries) or rowbinding the output after it was returned. (The rowbinding itself would be fine if each input could somehow be related to its one or many outputs.)

So I fork the package repo and set about rummaging through the source code. Blissfully, everything is nicely commented and the code was quite readable. 

I step through `monkeylearn_classify()` in the debugger and narrow in on a call to what looks like a utility function: `monkeylearn_parse()`. I find it in [`utils.R`](https://github.com/ropensci/monkeylearn/blob/master/R/utils.R).

The lines in `monkeylearn_parse()` that matter for our purposes are:


```r
text <- httr::content(output, as = "text",
                        encoding = "UTF-8")
temp <- jsonlite::fromJSON(text)
if(length(temp$result[[1]]) != 0){
  results <- do.call("rbind", temp$result)
}
```


So this is where the rowbinding happens -- *after* the `fromJSON` call!  🎉

This is good news because it means that the MonkeyLearn API *is* sending differentiated outputs back in a nested JSON object. The package converts this to a list with `fromJSON` and only *then* is the rbinding applied.

I set about copy-pasting `monkeylearn_parse()` and doing a bit of surgery on it. I created `monkeylearn_parse_each()`, which skips the rbinding and retains the list structure of each output. That meant that inside an enclosing function, the output of `monkeylearn_parse_each()` can be turned into a nested tibble with each row corresponding to one input. That nested tibble can then be related to each corresponding element of the input vector. All that remained was to use create a new enclosing analog to `monkeylearn_classify()` that could use `monkeylearn_parse_each()`.

#### Thinking PR thoughts

At this point, I thought that such a function might be useful to some other people using the package so I started writing this new funciton with an eye toward making a pull request.

Since I'd found it useful to be able to pass in an input dataframe in `initial_workaround()`, I figured I'd retain that option. I wanted users to still be able to pass in a bare column name but the package seemed to be light on tidyverse functions unless there was no alternative, so I un-tidyeval'd the function (using `deparse(substitute())` instead of a quosure) and gave it the imaginative name...`monkeylearn_classify_df()`. The rest of the original code was so airtight I didn't have to change much more to get it working. 


A nice side effect of my plumbing through the guts of the package was that I caught a couple minor bugs (things like the remnants of a for loop remaining in what had been revamped into a while loop) and noticed where there could be some quick wins for improving the package.

After a few more checks I wrote up a [pull request](https://github.com/ropensci/monkeylearn/pull/23) and checked list of [package contributors](https://github.com/ropensci/monkeylearn/graphs/contributors) to see if I knew anyone. Far and away the main contributor was [Maëlle Salmon](http://www.masalmon.eu/)! I'd heard of her through the magic of #rstats Twitter and the R-Ladies Global Slack. A minute or two after submitting it I headed over to Slack to give her a heads up that a PR would be heading her way.

In what I would come to know as her usual cheerful, perpetually-on-top-of-it form, Maëlle had already seen it and liked the idea for the new function. 


### Continuing Work

To make a short story shorter, Maëlle asked me if I'd like to create the extractor counterpart to `monkeylearn_classify_df` and work on improving the existing functionality in general. I said yes, of course, and we began strategizing over rOpenSci Slack about tradeoffs like which package dependencies we were okay with taking on, whether to go the tidyeval or base route, what the best naming conventions for the new functions should be, etc. 

On the naming front, we decided to gracefully deprecate `monkeylearn_classify` and `monkeylearn_extract` as the newer functions could cover all of the functionality that the older guys did. I don't know much about cache invalidation, but the naming problem [was hard as usual](https://github.com/ropensci/monkeylearn/issues/24). We settled on naming their counterparts `monkey_classify` (which replaced the original `monkeylearn_classify_df`) and `monkey_extract`. 


#### gitflow

Early on in the process we started talking git conventions. Rather than both work off a development branch, I floated a structure that we typically follow at my company, where each ticket (or in this case, GitHub Issue) becomes its own branch off of dev. For instance, issue #33 becomes branch `T33` (T for ticket). This approach, I am told, stems from the "[gitflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)" philosophy which, as far as I understand it, is one of many ways to structure a git workflow that mostly doesn't end in tears.

Like most git strategies, the idea here is to make pull requests as bite-sized as possible; a PR can only be as big as the issue it's named from. An added benefit for me, at least, is that it keeps me from wandering off into other parts of the code that I notice could be improved without first documenting the point in a separate issue, and then creating a branch. At most one person is assinged to each ticket/issue, which minimizes merge conflicts. You also leave a nice paper trail because the branch name directly references the issue front and center in its title. This means you don't have to explicitly name the issue in the commit or rely on GitHub's (albeit awesome) keyword branch closing system[^3]. 

Finally, since the system is so tied to issues themselves, it encourages very frequent communication between collaborators. Since the issue must necessarily be made before the branch and the accompanying changes to the code, the other contributors have a chance to weigh in on the issue or the approach suggested in its comments before any code is written. In our case, it's certainly made frequent communication the path of least resistance. 

While this branch and PR-naming convention isn't particular to gitflow (to my knowledge), it did spark a short conversation on Twitter that I think is useful to have:

[Thomas Lin Pedersen](https://www.data-imaginist.com/) makes a good point on the topic:

<blockquote class="twitter-tweet" data-lang="en">
  <p lang="en" dir="ltr">
I prefer named PRs as it gives a quick overview over opened PRs. While cross referencing with open issues is possible it is very tedious when you try to get an overview
  </p>
&mdash; Thomas Lin Pedersen (@thomasp85) 
  <a href="https://twitter.com/thomasp85/status/970941530709155841?ref_src=twsrc%5Etfw">March 6, 2018</a>
</blockquote>


<!-- <blockquote class="twitter-tweet" data-lang="en"> -->
<!--     <p lang="en" dir="ltr"> -->
<!--     I prefer named PRs as it gives a quick overview over opened PRs. While cross referencing with open issues is possible it is very tedious when you try to get an overview -->
<!--     </p> -->
<!--     &mdash; Thomas Lin Pedersen (@thomasp85) -->
<!--     <a href="<a href="https://twitter.com/thomasp85/status/970941530709155841?ref_src=twsrc%5Etfw"> -->
<!--       March 6, 2018 -->
<!--     </a> -->
<!-- </blockquote> -->
<!-- <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> -->

<!-- ```{r echo=FALSE} -->
<!-- blogdown::shortcode('tweet', '970941530709155841') -->
<!-- ``` -->


This insight has me thinking that the best approach might be to explicitly name the issue number *and* give a description in the branch or PR title.

In any case, our current system of only referenging the issue number has worked out well for Maëlle and me thus far, but that certainly doesn't mean that our commit history couldn't be more readable by including more verbose descriptions in the branch name as well.

#### Main Improvements

As I mentioned, the package was so good to begin with it was difficult to find ways to improve it. Most of the subsequent work I did was to improve the new `monkey_` functions.

They got more informative messages about which batches are currently being processed and which texts those batches corresponsed to. Rather than discarding inputs such as empty strings that could not be sent to the API as the original `monkeylearn_` functions did, we now return return a row of NAs. This means that the output is always of the same dimensions as the input, and can be unnested with either an `unnest` flag,.


```r
text_w_empties <- c(
    "In a hole in the ground there lived a hobbit.",
    "It is a truth universally acknowledged, that a single man in possession of a good fortune, must be in want of a wife.",
    "",
    "When Mr. Bilbo Baggins of Bag End announced that he would shortly be celebrating his eleventy-first birthday with a party of special magnificence, there was much talk and excitement in Hobbiton.",
    " ")

(empties_out <- monkey_classify(text_w_empties, texts_per_req = 2, unnest = TRUE) %>% knitr::kable())
```

```
## The following indices were empty strings and could not be sent to the API: 3
##         They will still be included in the output.
```

```
## Processing batch 1 of 2 batches: texts 1 to 2
```

```
## Processing batch 2 of 2 batches: texts 2 to 3
```



req                                                                                                                                                                                                   category_id   probability  label              
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------  ------------  ------------  -------------------
In a hole in the ground there lived a hobbit.                                                                                                                                                            18313280         0.071  Music              
In a hole in the ground there lived a hobbit.                                                                                                                                                            18313502         0.054  Music DVD's        
In a hole in the ground there lived a hobbit.                                                                                                                                                            18313524         0.553  See All Music DVDs 
It is a truth universally acknowledged, that a single man in possession of a good fortune, must be in want of a wife.                                                                                    18314767         0.062  Books              
It is a truth universally acknowledged, that a single man in possession of a good fortune, must be in want of a wife.                                                                                    18314954         0.047  Mystery & Suspense 
It is a truth universally acknowledged, that a single man in possession of a good fortune, must be in want of a wife.                                                                                    18314957         0.102  Police Procedural  
                                                                                                                                                                                                               NA            NA  NA                 
When Mr. Bilbo Baggins of Bag End announced that he would shortly be celebrating his eleventy-first birthday with a party of special magnificence, there was much talk and excitement in Hobbiton.       18313210         0.082  Party & Occasions  
When Mr. Bilbo Baggins of Bag End announced that he would shortly be celebrating his eleventy-first birthday with a party of special magnificence, there was much talk and excitement in Hobbiton.       18313231         0.176  Party Supplies     
When Mr. Bilbo Baggins of Bag End announced that he would shortly be celebrating his eleventy-first birthday with a party of special magnificence, there was much talk and excitement in Hobbiton.       18313235         0.134  Party Decorations  
When Mr. Bilbo Baggins of Bag End announced that he would shortly be celebrating his eleventy-first birthday with a party of special magnificence, there was much talk and excitement in Hobbiton.       18313236         0.406  Decorations        
                                                                                                                                                                                                               NA            NA  NA                 

So even though the empty string inputs like in row 3, aren't sent to the API, we can see they're still included in the output dataframe and assigned the same column names as all of the other outputs. That means that even if `unnest` is set to FALSE, the output can still be unnested with `tidyr::unnest()` after the fact.


#### Tangent on developing functions in tandem

Something I've been thinking about while working on the twin functions `monkey_extract()` and `monkey_classify()` is what the best practice is for developing very similar functions in sync with one another. These two functions are different enough to have different default values (`monkey_extract()` has a default `extractor_id` while `monkey_classify()` has a default `classifier_id`) but are very similar in other regards.

As soon as you make a change to one function, should you immediately make the same change to the other? Or is it instead better to work on one function at a time, and, at some checkpoints then batch these changes over to the other function in a big copy-paste job? I've been tending toward the latter but it's seemed a little dangerous to me.

Since there are only two functions to worry about here, creating a function factory to handle them seemed like overkill, but might technically be the best practice. I'd love to hear people's thoughts on how they go about navigating this facet of package development.


### Last Thoughts

My work on the monkeylearn package so far has been rewarding to say the least. It's inspired me to be less of a consumer and more of an active contributor to open source.

Maëlle's been a fantastic mentor through and through, providing guidance in at least three languages -- English, [French](https://twitter.com/ma_salmon/status/971992354763649024), and R, despite the time difference and 👶(!). I couldn't be more stoked for future collaborations. *On y va*!

<img src = "./onward.gif" style="margin-left: 10%">

<br>
<br>

[^1]: Custom, to a point. As of this writing, two types of classifier models you can create use either Naive Bayes or Support Vector Machines, though you can specify other parameters such as `use_stemmer` and `strip_stopwords`.

[^2]: Batching doesn't save you on requests (sending 200 texts in a batch means you now have 200 fewer queries), but it does save you bigtime on speed.

[^3]: Keywords in commits don't automatically close issues until they're merged into master, and since we were working off of dev for quite a long time, if we relied on keywords to automatically close issues our Open issues list wouldn't accurately reflect the issues that we actually still had to address. Would be cool for GitHub to allow flags like maybe "fixes #33 --dev" could close issue #33 when the PR with that phrase in the commit was merged into dev. 

<div style="display:none;"> 
  <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
<div>
