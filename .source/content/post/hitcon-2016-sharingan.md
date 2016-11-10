+++
date = "2016-10-23T20:49:57-07:00"
title = "HITCON 2016 - Sharingan"
draft = false

+++

It is no fair play.  Can you win?

<!--more-->

# Problem

> Sharingan  
> 38 Teams solved.
>
> Description  
> nc 52.197.160.186 31337  
> [sharingan.rb](https://s3-ap-northeast-1.amazonaws.com/hitcon2016qual/sharingan.rb_20a3d2497ad0292d72b4796754c5585523b3e985)  
>
> Hint  
> None

# Solution

I started by playing around with the program locally.  It turns out to be a Go AI with very interesting rule: As soon
as the white stone (me) tries to capture black stone (AI), so called "Isanagi" is activated.  As a result of "Isanagi",
that last capturing white stone turns black!  Other than this unfair rule, it is a very basic AI that plays pure
[Mirror Go](https://en.wikipedia.org/wiki/Mirror_go).  If there's no way to mirror the last move, it randomly pick a
valid move.

At first I tried to apply some well known mirror go refutations but because of the Isanagi rule, they don't seem to work.
It looks like there's no easy way to beat this simple AI unless there's fundamental flaw in the program.

And there is.

```ruby
def main
  brd = Array.new(N){Array.new(N){EMPTY}}
  who = 0
  loop do
    color = color_of who
    case who
    when 0 then x, y = ai(brd, color)
    when 1 then x, y = player(brd, color)
    else 
    end
    next if x.nil?
    brd[x][y] = color
    if who == 1 and will_capture(brd, x, y, color)
      puts "Isanagi activated!"
      brd[x][y] = exchg(brd[x][y])
    end
    N.times do |a|
      N.times do |b|
        next if brd[a][b] == EMPTY || brd[a][b] == color
        capture!(brd, a, b) if not alive?(brd, a, b, Set.new)
      end
    end
    @last_x, @last_y = x, y
    who ^= 1
    show(brd) and flag if win(brd) # <<< checks at every iteration
  end
end
```

It turns out the program checks the winning condition *at every iteration*.  It checks if there're more white stones
than black stones right after the white stone's move, before the mirror happens.  This means if I can capture and remove
two (because black takes first move) black stones in one move, it will show me the flag!

That's easy enough.  Since AI simply mirror my move, I can pre-capture an area with three holes, put two black stones
into them, and fill the last hole with white stone.  The Isanagi rule will turn the last white stone into black.  Because
it is pre-captured, all three black stones will be removed.  In the last move, one white stone and two black stones were
removed, temporarily leaving more white stones than black stones, and prints the flag.

So the final solution:

```bash
$ print "0 1\n1 1\n2 1\n3 1\n3 0\n18 18\n17 18\n2 0" | nc 52.197.160.186 31337
```

