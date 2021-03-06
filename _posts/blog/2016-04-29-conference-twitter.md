---
layout: media
title: 'Playing with Twitter Data'
categories: blog
date: 2016-04-29
tags: [rstats, networks, openscience]
comments: true
image:
   feature: blog/twitter/main.jpg
   teaser: blog/twitter/mentionNet.jpg
---

Last Friday, the [Institute for Social Sciences](http://socialsciences.ucdavis.edu) hosted a great one-day conference on various aspects of the reproducability crisis, [*Making Social Science Transparent*](http://socialsciences.ucdavis.edu/impact/iss-conferences/making-social-science-transparent/iss-conference-confronts-transparency-and-replication-crisis). It was the first time I've done much tweeting during an event like this, and while it felt a little silly, it was also fun, it was nice to hear what was resonating with other people at the event, and I'm psyched to stay connected to other participants on Twitter.

It also gave me an excuse to learn to scrape and analyze Twitter data. And doing that pushed me to setup RMarkdown rendering on this Jekyll site. I'm pretty psyched about both.

I was surprised by how easy it was to get and manage Twitter data. The `twitteR` package is awesome. If you want to try this out yourself, the first thing to do is [register your "app" with Twitter](https://apps.twitter.com/), which just takes a couple minutes. 

Now let's see if we can find anything interesting...

## Get the data

Start by loading some libraries and authenticating myself to Twitter. The arguments to `setup_twitter_oath` are access tokens that Twitter gives you when you register your app and are just strings that I've defined elsewhere so that you can't see them. That plus `searchTwitter` and `twListToDF` are the only functions you need to get and organize tweets around a particular subject. Super simple.


{% highlight r %}
lapply(c('twitteR', 'dplyr', 'ggplot2', 'lubridate', 'network', 'sna', 'qdap', 'tm'),
       library, character.only = TRUE)
theme_set(new = theme_bw())
source('../../R/twitterAuth.R')
set.seed(95616)

# setup_twitter_oauth(my_key, my_secret, my_access_token, my_access_secret)
# tw = searchTwitter('#MSST2016', n = 1e4, since = '2016-04-01')
# saveRDS(tw, '../../R/MSST_Tweets.RDS')
tw = readRDS('../../R/MSST_Tweets.RDS')
d = twListToDF(tw)
{% endhighlight %}

## When do people tweet?

We get 133 original tweets and 177 retweets. There were a few announcements in the days leading up to the conference, and a few retweets after, but the action was concentrated on the day of the event.

People compose original tweets during panels, not at breaks and especially not during lunch. Retweeting abates less during the breaks, perhaps due to people not at the conference.


{% highlight r %}
# Put in local time
d$created = with_tz(d$created, 'America/Los_Angeles')

timeDist = ggplot(d, aes(created)) + 
    geom_density(aes(fill = isRetweet), alpha = .5) +
    scale_fill_discrete(guide = 'none') +
    xlab('All tweets')

# Zoom in on conference day
dayOf = filter(d, mday(created) == 22)
timeDistDayOf = ggplot(dayOf, aes(created)) + 
    geom_density(aes(fill = isRetweet), adjust = .25, alpha = .5) +
    theme(legend.justification = c(1, 1), legend.position = c(1, 1)) +
    xlab('Day-of tweets')
cowplot::plot_grid(timeDist, timeDistDayOf)
{% endhighlight %}

![plot of chunk time](/assets/Rfig/conference-twitter//time-1.svg)

## What platforms are people using?

Mostly what we would expect, but some bots got in on the action too.


{% highlight r %}
par(mar = c(3, 3, 3, 2))
d$statusSource = substr(d$statusSource, 
                        regexpr('>', d$statusSource) + 1, 
                        regexpr('</a>', d$statusSource) - 1)
dotchart(sort(table(d$statusSource)))
mtext('Number of tweets posted by platform')
{% endhighlight %}

![plot of chunk platforms](/assets/Rfig/conference-twitter//platforms-1.svg)

## Emotional valence of tweets

Let's look at the content of the tweets, first over the course of the day. The first panel, *Defining the Issues: Research on Replication, Reproducibility, and Transparency* had some hard truths in it -- Brad Jones predicted it would be the "sky is falling" panel -- and we see that reflected in the smoother going below zero for one of only two times in the day during that panel. The second panel was a call to action: we heard about the [Open Science Framework](https://osf.io/) from [Katie Corker](https://twitter.com/katiecorker) (OSF is doing two [workshops on campus](http://datascience.ucdavis.edu/Events.html#OpenScience) May 4) and [Data Carpentry](http://www.datacarpentry.org/) from [Tracy Teal](https://twitter.com/tracykteal), as well as some ideas about what individuals and labs can do on their own. That, it seems, was the emotional high point of the day, at least until someone had a couple glasses of wine at the reception and then closed out the day with an exuberant tweet. 


{% highlight r %}
# Split into retweets and original tweets
sp = split(d, d$isRetweet)
orig = sp[['FALSE']]
# Extract the retweets and pull the original author's screenname
rt = mutate(sp[['TRUE']], sender = substr(text, 5, regexpr(':', text) - 1))

pol = 
    lapply(orig$text, function(txt) {
        # strip sentence enders so each tweet is analyzed as a sentence,
        # and +'s which muck up regex
        gsub('(\\.|!|\\?)\\s+|(\\++)', ' ', txt) %>%
            # strip URLs
            gsub(' http[^[:blank:]]+', '', .) %>%
            # calculate polarity
            polarity()
    })
orig$emotionalValence = sapply(pol, function(x) x$all$polarity)

# As reality check, what are the most and least positive tweets
orig$text[which.max(orig$emotionalValence)]
{% endhighlight %}



{% highlight text %}
## [1] "Hey, this Open Science Framework sounds like a great way to  collaborate openly! Where do I sign up? Here: https://t.co/9oAClb0hCP #MSST2016"
{% endhighlight %}



{% highlight r %}
orig$text[which.min(orig$emotionalValence)]
{% endhighlight %}



{% highlight text %}
## [1] "1 Replications are boring 2 replications are attack 3 reputations will suffer 4 only easy ones will be done 5 bad studies are bad #MSST2016"
{% endhighlight %}



{% highlight r %}
# How does emotionalValence change over the day?
filter(orig, mday(created) == 22) %>%
    ggplot(aes(created, emotionalValence)) +
    geom_point() + 
    geom_smooth(span = .5)
{% endhighlight %}

![plot of chunk polTime](/assets/Rfig/conference-twitter//polTime-1.svg)

## Do happier tweets get retweeted more?

My guess was that more emotionally-positive tweets (greater emotionalValence scores) would be retweeted more -- people gravitate to happy messages, right? -- but it looks like tweets with less-emotional content (closer to zero valence) get retweeted more. I wonder if it's generally true that academics prefer and retweet emotionally neutral messages, and how that compares to the population at large. If it is generally true for academics but not others, there's an important science-outreach lesson here.


{% highlight r %}
ggplot(orig, aes(x = emotionalValence, y = retweetCount)) +
    geom_point(position = 'jitter') +
    geom_smooth()
{% endhighlight %}

![plot of chunk unnamed-chunk-1](/assets/Rfig/conference-twitter//unnamed-chunk-1-1.svg)

## Emotional content

Now let's look at the content of what was tweeted, parsed by the emotional valence of the tweet. This is a super naive analysis -- I'm using `qdap`'s `polarity` function straight out of the box to examine the emotional valence of each tweet. One nice thing about that function is that it returns the positive and negative words found in each text, which allows me to A) tabulate the most-used positively and negatively valenced words, and B) to strip those words from positive and negative tweets to see what is being talked about in positive and negative ways, without interference from the emotionally-charged words themselves.

Here is the frequency of usage in the conference tweets of words that `qdap` (using the sentiment dictionary of Hu & Liu, 2004) identifies as positively or negatively valenced.


{% highlight r %}
polWordTables = 
    sapply(pol, function(p) {
        words = c(positiveWords = paste(p[[1]]$pos.words[[1]], collapse = ' '), 
                  negativeWords = paste(p[[1]]$neg.words[[1]], collapse = ' '))
        gsub('-', '', words)  # Get rid of nothing found's "-"
    }) %>%
    apply(1, paste, collapse = ' ') %>% 
    stripWhitespace() %>% 
    strsplit(' ') %>%
    sapply(table)

par(mfrow = c(1, 2))
invisible(
    lapply(1:2, function(i) {
    dotchart(sort(polWordTables[[i]]), cex = .8)
    mtext(names(polWordTables)[i])
    }))
{% endhighlight %}

![plot of chunk polWords](/assets/Rfig/conference-twitter//polWords-1.svg)

## Emotionally associated non-emotional words

No text analysis would be complete without a wordcloud. Here I've classified tweets by their emotionalValence score and removed the words that score on emotionally valence, to sort the content that was being talked about in emotionally different ways. Mixed success, I think -- there's not a lot of data here, and the emotional-words list could use some tuning.


{% highlight r %}
polSplit = split(orig, sign(orig$emotionalValence))
polText = sapply(polSplit, function(df) {
    paste(tolower(df$text), collapse = ' ') %>%
        gsub(' (http|@)[^[:blank:]]+', '', .) %>%
        gsub('[[:punct:]]', '', .)
    }) %>%
    structure(names = c('negative', 'neutral', 'positive'))

# remove emotive words
polText['negative'] = removeWords(polText['negative'], names(polWordTables$negativeWords))
polText['positive'] = removeWords(polText['positive'], names(polWordTables$positiveWords))

# Make a corpus by valence and a wordcloud from it
corp = make_corpus(polText)
col3 = RColorBrewer::brewer.pal(3, 'Paired') # Define some pretty colors, mostly for later
wordcloud::comparison.cloud(as.matrix(TermDocumentMatrix(corp)), 
                            max.words = 100, min.freq = 2, random.order=FALSE, 
                            rot.per = 0, colors = col3, vfont = c("sans serif", "plain"))
{% endhighlight %}

![plot of chunk wordcloud](/assets/Rfig/conference-twitter//wordcloud-1.svg)

## Who's retweeting whom?

Twitter's API doesn't tell us where a retweeter saw the tweet that they retweeted; an edge always goes from the original author of the tweet to the retweeter, so we can't follow the diffusion of a tweet. But, we can get a sense of who is being retweeted, and we see a core of individuals engaging in a conversation at the center of the graph. Nodes are sized to their total degree (retweeting and being retweeted), and edge-width is proportional to the number of retweets between that pair. Labeled nodes are those that were retweeted at least once.


{% highlight r %}
# Adjust retweets to create an edgelist for network
el = as.data.frame(cbind(sender = tolower(rt$sender), 
                         receiver = tolower(rt$screenName)))
el = count(el, sender, receiver) 
rtnet = network(el, matrix.type = 'edgelist', directed = TRUE, 
                ignore.eval = FALSE, names.eval = 'num')

# Get names of only those who were retweeted to keep labeling reasonable
vlabs = rtnet %v% 'vertex.names'
vlabs[degree(rtnet, cmode = 'outdegree') == 0] = NA

par(mar = c(0, 0, 3, 0))
plot(rtnet, label = vlabs, label.pos = 5, label.cex = .8, 
     vertex.cex = log(degree(rtnet)) + .5, vertex.col = col3[1],
     edge.lwd = 'num', edge.col = 'gray70', main = '#MSST2016 Retweet Network')
{% endhighlight %}

![plot of chunk retweetNet](/assets/Rfig/conference-twitter//retweetNet-1.svg)


## Who's metioning whom?

It's almost exclusively speakers and the host institute that get mentioned, though a few outside players get some shout-outs, especially the Open Science Framework, which was discussed in detail by [Katie Corker](https://twitter.com/katiecorker). Unsurprisingly, the host, UCDavisISS, does a lot of mentioning and gets mentioned quite a bit. I think it's interesting that graph fractures almost perfectly between the host and the speakers: there is very little cross-talk between those who are mentioning and being mentioned by the host, and those who are mentioning and being mentioned by the speakers.

Edges originate at the tweeter and point to the mentioned; nodes are scaled to number of mentions.


{% highlight r %}
# Extract who is mentioned in each tweet. 
# Someone has probably written a function to do this, but it's a fun regex problem.
mentioned = 
    lapply(orig$text, function(tx) {
        matches = gregexpr('@[^([:blank:]|[:punct:])]+', tx)[[1]]
        sapply(seq_along(matches), function(i) 
            substr(tx, matches[i] + 1, matches[i] + attr(matches, 'match.length')[i] - 1))
    })
# Make an edge from the tweeter to the mentioned, for each mention
mentionEL = 
    lapply(seq_along(orig$text), function(i) {
        # If the tweet didn't have a mention, don't make edges
        if(mentioned[[i]] == '')  
            return(NULL)
        # Otherwise, loop over each person mentioned, make an edge, and rbind them
        lapply(mentioned[[i]], function(m)
            c(sender = orig$screenName[i], receiver = m)) %>%
            do.call(rbind, .) %>% as.data.frame()
    }) %>% 
    do.call(rbind, .) %>%
    count(tolower(sender), tolower(receiver))

# Make the network
mentionNet = network(mentionEL, matrix.type = 'edgelist', directed = TRUE, 
                ignore.eval = FALSE, names.eval = 'num')

# Color speakers and the host
vCol = rep(col3[3], network.size(mentionNet))
speakers = c('duncantl', 'rlucas11', 'cristobalyoung5', 'katiecorker', 
             'mcxfrank', 'tracykteal', 'siminevazire', 'jwpatty', 
             'kramtrak', 'phylogenomics', 'donandrewmoore')
vCol[(mentionNet %v% 'vertex.names') %in% speakers] = col3[1]
vCol[mentionNet %v% 'vertex.names' == 'ucdavisiss'] = col3[2]

plot(mentionNet, displaylabels = TRUE, label.pos = 5, label.cex = .8, 
     vertex.cex = degree(mentionNet, cmode = 'indegree'), vertex.col = vCol,
     edge.lwd = 'num', edge.col = 'gray70', main = '#MSST2016 Mention Network')
legend(x = 'bottomleft', legend = c('Speaker', 'Host', 'Other'), 
       pt.bg = col3, pch = 21, pt.cex = 1.5, bty = 'n')
{% endhighlight %}

![plot of chunk mentionNet](/assets/Rfig/conference-twitter//mentionNet-1.svg)

Thanks for reading. It really was a great conference; if you'd like some real information on what went down here is [Ben Hinshaw's summary of the event](http://socialsciences.ucdavis.edu/impact/iss-conferences/making-social-science-transparent/iss-conference-confronts-transparency-and-replication-crisis). 
