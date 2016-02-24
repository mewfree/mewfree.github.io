---
layout: post
title:  "(H)Ledger: Double-Entry Accounting Using Haskell and the Terminal (Part 1)"
date:   2016-02-23 21:43:00
categories: update
---
When I moved back to Montréal in 2015 for my internship I had to open a local bank account to receive my paycheck.
I wanted to open an account at Tangerine as they offer no fee banking (I remembered them from their great advertising
campaign in the Montréal Métro a while back) but I needed a cheque from another Canadian banking institution.
I tried going to their "Café" but apparently they're not too keen on foreign passports and driver's licenses.

I ended up going to a local Bank of Montreal (BMO) branch on the Mont-Royal Avenue during the weekend because
it was the only bank open on a Saturday afternoon in the neighborhood. I was still a student at the time
so I got away with some kind of a basic banking package (a chequing account, a savings account, a credit card)
with no fees.

After some time, I used my BMO account to open a Tangerine account as it was my initial goal. Everything was
great but I now had a very mundane problem: how do I keep track of my money on a single interface?

I've heard of Mint (which supports Canadian banks 😁) but I wasn't excited about the idea of a
private company having access to my different bank accounts and storing my different credentials on their own servers.
You Need A Budget (YNAB) seemed okay but I had no intention of paying for such a tool.

But recently, this video made an appearance on my YouTube subscriptions feed:
<iframe width="560" height="315" src="https://www.youtube.com/embed/cjoCNRpLanY" frameborder="0" allowfullscreen></iframe>

I was amazed! This was exactly what I needed! I could do everything that I wanted, from the comfort of my terminal,
and everything would be self-hosted! I installed ledger-cli with a simple `brew install ledger`, installed
the Vim plugin [vim-ledger](https://github.com/ledger/vim-ledger) (as I don't use Emacs, sorry rms) and I was good to go!

Ledger uses a principle that was familiar to me thanks to my business school background: double-entry accounting.
If you don't know what it is, either watch the video above for a super quick intro or [make some Google searches](http://lmgtfy.com/?q=double+entry+accounting).
Also, it stores everything in **plain text** meaning it can be human readable, easily back-upped and exported, etc...

My next step was to read the [Ledger's manual](http://ledger-cli.org/3.0/doc/ledger3.html) to get my first journal going.
Everything was going well, I entered some transactions manually but I got tired really quickly. There's no way that I would
manually enter those dozens of monthly ATM withdrawals, online and offline credit card transactions, bill payments...
One of the reasons why I wanted to use a system of this kind was to analyze how I spend my money so I was really
interested in having historical transactions data from years ago.

So my consensus in my head at that moment was "Let's write a script for that!". But then I logged in my bank
account and remembered about all the security measures I'd have to go through to do anything programatically.
There's gotta be a solution... My French bank Monabanq allows me to export my historical data to a CSV file:
![Monabanq export](../.../../../../../assets/monabanq.png)

Great! But other banks are not as nice. BMO allows you to export data as CSV, but only for the last two months?!
Why?! And the web interface only allows you to display transactions month by month... I'll be able to use
the export function for my future transactions, but I'll need to do some dirty work for the past ones.

Tangerine is a little bit nicer. They allow you to list all transactions, but the HTML is paginated. Hopefully
there's a "print" version that lists everything on one page. And the rendering is clean enough that some
copy pasting will easily convert that in a CSV file.

I remembered ledger's CSV importing capabilities after my initial Googling. I tried to dive more
into it and stumbled upon the "ledger-likes" community. One of those Ledger "port" is [hledger](http://hledger.org/),
written in Haskell. It tries to be compatible with ledger files and has great [CSV importing functionalities](http://hledger.org/how-to-read-csv-files.html).
You can basically write a rule file that defines how each CSV from each account has to be imported. No need
for full blown scripting! Amazing! But if I'll ever want some custom advanced stuff later, I know I'll be
able to leverage Haskell's power and write my own `.hs` scripts. Exciting.

Let the hacking begin!
I'm starting with BMO. I'm able to create a simple CSV rule for hledger really quickly:

    fields  date, , description, amount-out, amount-in

    currency CAD

    account1 assets:chequing BMO

I then copy & paste
all my monthly data from the web interface to a spreadsheet. The spreadsheet does a good job of
understanding the data format. I convert dates to a sane YYYY-MM-DD format, remove the confusing $ signs on
transactions (is it USD? is it CAD? my chequing account is in CAD but I have a USD savings account) because
I'd rather enter those manually (currency is set directly in the rule file, account by account). Download as CSV.
`hledger -f chequingBMO.csv print`. hledger understands everything. Excellent.

    2015/11/02 VIDEOTRON SENC BPY/FAC
        expenses:unknown         CAD56.90
        assets:chequing BMO     CAD-56.90

    2015/11/02 BURGER DE VILLE
        expenses:unknown         CAD12.65
        assets:chequing BMO     CAD-12.65

    2015/11/03 PAYPAL PTE LTD MSP/DIV
        expenses:unknown         CAD53.94
        assets:chequing BMO     CAD-53.94

    2015/11/05 COMPTOIR 21
        expenses:unknown         CAD13.69
        assets:chequing BMO     CAD-13.69

Mh. You see those `expenses:unknown`? I wish hledger would be able to classify at least the most common
ones. Let's write a rule for the example above. Vidéotron is my cellphone carrier, Burger de Ville and
Comptoir 21 are excellent restaurants in Mile End, Montréal, and I think everybody knows about PayPal.

The rule file now looks like this:

    fields  date, , description, amount-out, amount-in

    currency CAD

    account1 assets:chequing BMO

    if ~ VIDEOTRON
      account2 expenses:bills:cellphone

    if ~ BURGER DE VILLE
      account2 expenses:restaurants:BGV

    if ~ COMPTOIR 21
      account2 expenses:restaurants:comptoir 21

    if ~ PAYPAL
      account2 expenses:misc:paypal

And here is how hledger interpreted it:

    2015/11/02 VIDEOTRON SENC BPY/FAC
	expenses:bills:cellphone      CAD56.90
	assets:chequing BMO          CAD-56.90

    2015/11/02 BURGER DE VILLE
	expenses:restaurants:BGV      CAD12.65
	assets:chequing BMO          CAD-12.65

    2015/11/03 PAYPAL PTE LTD MSP/DIV
	expenses:misc:paypal      CAD53.94
	assets:chequing BMO      CAD-53.94

    2015/11/05 COMPTOIR 21
	expenses:restaurants:comptoir 21      CAD13.69
	assets:chequing BMO                  CAD-13.69

This looks muuuucchhh better 🎉

Before setting more rules and importing transaction from other accounts, let's have some fun:

`hledger -f chequingBMO.csv balance expenses:restaurants` should tell me how much I spent on eating out recently.

	       CAD246.09  expenses:restaurants
	       CAD131.77    BGV
	       CAD114.32    comptoir 21
    --------------------
	       CAD246.09

Wow. And that's only with two restaurants mapped and data from my debit card only (that I barely use compared to my
credit card!).

On that depressing note, I decide to call it night and continue my financial hacking later.

See you again for part 2 soon!
