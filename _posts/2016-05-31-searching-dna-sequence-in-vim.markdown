---
layout: post
title:  "Searching DNA sequence in Vim"
date:   2016-05-31 14:41:24 +0100
categories: vim dna
---

I love the text editor Vim. Back when I started doing some Bioinformatics work I had a feeling that I should be using a better text editor than pico or whatever it was at the time. I'd heard of vim, so I gave it a go. Starting out with vim is a bewildering experience where any stray key press could accidentally destroy all your work. The twenty minutes or so learning how to actually produce text (and undo mistakes!) pays huge dividends however. I wrote my whole PhD thesis in vim and Latex in a terminal window! I'm still learning and still being impressed by how powerful vim is, and how much you can do with it - [see here for some good examples](http://whileimautomaton.net/2008/11/vimm3/operator).

Since most sequence data arrives as a text file of some sort, I often end up opening it in vim and doing some preliminary sanity checks, reformatting or even analysis using some quick vim commands. For example, when someone emails sequences copy-pasted from an excel file that need to be converted to fasta to blast them.

```
Sequence_1  ATGGCCATAGGCA
Sequence_2  TTAGACCGTACCA
```

As any vim wizard knows, all you need here is a quick `<Ctrl-V>GI><ESC>:%s/\s\+/^M/g<ENTER>` and there's your fasta format! Searching with highlighting also tends to be useful to look for expected bits of sequence and quickly see their surrounding context. It's this that prompted my latest vim level-up. Vim is not biology-aware so I often end up having to run two searches - one for the sequence of interest and one for its [reverse complement](https://en.wikipedia.org/wiki/Complementarity_%28molecular_biology%29), in case we have a mixture of sequences from both strands in the file. This involves a potentially error-prone manual conversion or copy-paste into some other reverse-complementing tool, so I figured there should be some way to extend vim to do this. All I would need to do is convert my search string automatically into a regex that searched for my original string or its reverse-complement, something like this:

```
GTTC => GTTC|GAAC 
```

These thoughts came with a certain amount of trepidation, however, as vims reputation for inscrutability is as legendary as its power (see my fasta conversion "macro" above). Fortunately a bit of searching found [this useful PDF](mattmargolis.net/scripting_vim_with_ruby.pdf), suggesting that you could script Vim plugins in ruby! Sounds great, I could use Bioruby, or just knock up my own reverse-complementing function very easily, a bit like this.

{% highlight ruby %}
def rcsearch(seq)
  return 0 unless seq.match(/\A[GATCN]+\z/i)
  revcomp = {
    "A" => "T",
    "G" => "C",
    "T" => "A",
    "C" => "G",
    }
  rcseq = seq.each_char.map{|a| revcomp[a.upcase]}.reverse.join
  seq + '\|' + rcseq #Escape the OR pipe for vim regexp syntax
end
{% endhighlight %}

The fun starts when you try to call this from Vim:

{% highlight vim %}
exec expand("rubyfile <sfile>:p:h/rcsearch.rb")
function! Rcsearch(seq)
  ruby <<EOF
    seq = VIM::evaluate('a:seq')
    pattern = BioVim.rcsearch(seq)
    VIM.command("let pattern = \'#{pattern}\'")
EOF
  if pattern  "uh oh...
    let @/ = pattern
    normal! n
  else
    echom "Invalid sequence - please use only GATCN."
  endif
endfunction
{% endhighlight %}

The first line sources the ruby file, via the vim `rubyfile` function, containing the function (`<sfile>:p:h` returns the directory of the current file). Next we have a vim function definition. I highly recommend [Learn Vimscript the Hard Way](http://learnvimscriptthehardway.stevelosh.com) for all the info on vimscripts quirks - the exclamation mark and capital letter in the definition here are not really optional! There are only a couple of ways to to interact with ruby from vim and vice versa (:help ruby). One is the `rubyfile` method, or the `ruby` method used here to execute inline ruby code. I used the `evaluate` function in the ruby VIM module to send the value of the passed in argument (which has to be scoped with the leading a:, thanks vim) to ruby. Then I can call my function (which I put into a ruby module called Biovim) on it and use one of the other VIM module methods to set a vim variable to the result. Looks a bit clunky, maybe there's a better way - but the module only has four methods so options seemed limited!

From now on it's vimscript all the way and the craziness really starts. We have a pattern that includes an OR operator and our reverse complemented sequence. To actually do the search, we can set vim's search register @/, which usually stores the last searched pattern, to our new regexp. Then by programatically hitting 'n' we go to the 'next' occurence of this match, just as if this were interactive vim. This works fine, you even get highlighting just as if you did a normal search! But there's an ugly error if the ruby function fails, so to cut down on that I added some input validation, with a return of 0 if you try to feed it non-sequence characters. If `pattern` gets successfully set to a string, we will search for it, otherwise it's 0 (falsey) so we will have a helpful error message. Except...vim helpfully coerces strings to 0 in conditions!! I still have no idea why you would ever want this. (Learn Vimscript the Hard Way is particularly sardonic in this chapter). The only way I could find to fix it is:

{% highlight vim %}
  if pattern != '0'
{% endhighlight %}

Now the *string* zero gets coerced to an integer 0 if comparing with an integer, but left as a (non-equal) string if `pattern` is a string! Ruby and vimscript are truly at the opposite ends of the readability and intention conveying spectrum, but for me they make a good match! Vim doesn't really have a good unit testing framework, so ruby fills a big hole there. Although there are only basic methods available for interacting between the two languages, it seems to do the job. Already getting some use out of my search function and plotting some further functionality...

I used [Pathogen](https://github.com/tpope/vim-pathogen) which makes installation and development really easy, and added a binding to a command to make calling the function easier. This could then be mapped to a key for further convenience; but since spare keys are at a premium in vim this is best left to the user.

{% highlight vim %}
command -nargs=1 Biorc :call Rcsearch(<f-args>)
{% endhighlight %}
