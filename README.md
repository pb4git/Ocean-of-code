
# Ocean of Code
A Post-Mortem by pb4
Codingame challenge - March/April 2020  

# Introduction:

  

Ocean of Code is a multiplayer AI contest proposed on the Codingame platform from Friday the 20th of March to Monday the 20th of April 2020. It is based on the board game Captain Sonar and requires a delicate balancing act between being active and detectable with the submarine under your control while at the same time staying as stealthy as possible.

  

At the beginning, I was hooked on the challenge of implementing a proper system to track the opponent. In the early days, this was enough to climb all the way up the leaderboard. From that moment on, my fate was clear: I was going to spend one month trying to improve this AI…

  

# First versions and game analysis

  

Over the duration of the contest, I have written 8 major versions of my AI. Here is the breakdown of how they behaved, how they influenced my vision of the correct strategy and hence how I prioritized features for my final version.

The **first version** was purely reactive and based on simple sequential  greedy rules:
-   Move to the closest cells, stay in the biggest chamber   
-   Shoot the opponent if the damage probability is sufficient
-   Place a mine randomly any time I can    
-   Silence 0 if the opponent can shoot me
-   Trigger mines if the probability is sufficient
    
The **second and third versions** were able to *group move and torpedo actions together* if the opponent could be brought in range with a single movement. However, *this move-to-shoot logic was susceptible to close off a large area of the map*.

The **fourth version** *prioritized stealthness over shooting*. Games were long, but would often be lost when the map is filled with the submarine's previous path: remaining hidden proves impossible when space is cramped, and I would lose in torpedo fights. At some point, *it is important to balance stealthness with aggressiveness*.

By this time, moving with the sole objective to stay stealthy was a declining strategy as most players started to use mines efficiently. Movement was now a difficult trade-off between:
- Keeping long chambers available (tron-like)
- Staying stealthy
- Avoiding mines
- Approaching the opponent to shoot

My **fifth and sixth version** were (failed) attempts at integrating the four criteria above into greedy rule-based logic in one single decision step. I *failed at balancing* correctly all aspects together.

The **seventh version** was acceptable, it balanced movement correctly and had a strong late-game but *often lost during forced engagements where the opponent had taken the initiative*.

At this point, I was ready to write my **eighth and final version** based on the strategy below.

 # Strategy
As of writing this PM, I have come to the conclusion that captain sonar must be played as two mini-games in one.
1) A space-filling game where you must claim the most surface area to out-survive the opponent in the late game
2) A waiting game where you must **at all point in time be ready** if the opponent finds and engages your submarine

Obviously, this assumes decent fighting logic as soon as both submarines are close to each other and revealed.

While point #2 may seem obvious, its direct implication is that **torpedo and silence charges must be conserved at all times**. Hence, **it is of utmost importance to use Silence as sparingly as possible**.  The only valid reasons to use Silence would be to avoid the opponent's mine field, or preemptively evade an unfavorable engagement where the opponent will shoot you and you can not retaliate. Even if a submarine gets the first torpedo shot, it will be at a disadvantage if it needs to charge both Torpedo and Silence at the same time.

This approach introduces a new compromise to make:
> "I can Silence right now, the opponent has a medium chance to hit me.
> Should I silence away ?"

# Final version

  As explained above, the objective of this final version is to balance correctly the four actions Torpedo / Move / Silence / Surface and allow fine-tuned compromise between aggressive and defensive behaviors.
## Evaluation framework
In order to have a unified evaluation, all actions are evaluated in the framework of total discounted damage taken from or done to the opponent.
- For a torpedo, it is simply the damage probability given the opponent presence map `damageProbability*0.95^0`
- For a surface, we take a known damage when surface happens and assume we'll need to surface again after `pathSize` turns : ``1*(0.95^turn + 0.95^(turn + pathSize)``
- For space-filling, `0.95^spaceLeft` : we will need to surface when all cells are filled in
- For a mine, the submarine takes `mineDamage*0.95^turn` damage when walking through a mine field. See below how mineDamage is calculated.

## Overall structure
The objective of the search is to evaluate nearly exhaustively all Surface / Move / Torpedo / Silence combinations.
The search is split in three consecutive steps:
1) For all Surface/Silence/Move combinations, evaluate the future damage I'll take from navigating the opponent's mine field and from my own surface
2) For all valid combinations from step 1, interleave a torpedo between all actions and evaluate the damage dealt to the opponent.
3) For all valid combinations from step 2, evaluate the opponent retaliation (ie: best torpedo) if he's allowed to move only, move/silence or surface/move/silence



## Step 1: Handling navigation through a minefield

The objective is to construct a list associating movement orders at turn 0 to future damage taken. To do this, for all initial Surface/Silence/Move combination, a Dijkstra search is launched looking for the cheapest way to move 5 tiles deep.

     Move at turn 0                           Future damage taken   Best move continuation
    {MOVE N SILENCE},                         0.055                 MOVE E | MOVE E | MOVE S
    {SURFACE,MOVE N SILENCE},                 1.055                 MOVE E | MOVE E | MOVE S
    {MOVE E SILENCE},                         0.045                 MOVE E | MOVE S | MOVE S
    {SURFACE,MOVE E SILENCE},                 1.045                 SURFACE | MOVE E | MOVE S
    {MOVE S SILENCE},                         invalid               MOVE N | MOVE E | MOVE S
    {SURFACE,MOVE S SILENCE},                 1.000                 ...
    {MOVE W SILENCE},                         0.333                 ...
    {SURFACE,MOVE W SILENCE},                 1.333                 ...
    {MOVE N SILENCE,SILENCE N 0},             0.055                 ...
    {SURFACE,MOVE N SILENCE,SILENCE N 0},     1.055                 ...
    {MOVE N SILENCE,SILENCE N 1},             invalid               ...
    {SURFACE,MOVE N SILENCE,SILENCE N 1}      invalid               ...
    ...   (144 lines)                         ...                   ...

  
Calculation for "future damage taken" is shown below.
Other players have explained in detail how it is possible to generate a mine presence probability map such as the following one:

        0   0   0   0   0   0
        0   X   0   0   0   0
        0   0   0   25  0   0
        0   12  37  0   25  0
        12  12  12  37  0   0
        0   12  12  0   0   0
        0   0   0   0   0   0

The future damage taken is evaluated assuming all mines are "auto-triggered" if the submarine is within range.
As an example, assuming the submarine start in position X on the diagram above:


    MOVE S  : trigger all mines marked 'a', this is (12+37)*0.95^0 damage
        0   0   0   0   0   0
        a   X   a   0   0   0
        a   a   a   25  0   0
        a   a   a   0   25  0
        12  12  12  37  0   0
        0   12  12  0   0   0
        0   0   0   0   0   0
        
    MOVE S  : trigger all mines marked 'b', this is (12+12+12)*0.95^1 more damage
        0   0   0   0   0   0
        a   X   a   0   0   0
        a   a   a   25  0   0
        a   a   a   0   25  0
        b   b   b  37  0   0
        0   12  12  0   0   0
        0   0   0   0   0   0
        
    MOVE E  : trigger all mines marked 'c', this is (37+25)*0.95^2 more damage
        0   0   0   0   0   0
        a   X   a   0   0   0
        a   a   a   c   0   0
        a   a   a   c   25  0
        b   b   b   c   0   0
        0   12  12  0   0   0
        0   0   0   0   0   0
        
    MOVE N  : trigger all mines marked 'd', this is (0)*0.95^3 more damage
        0   0   0   0   0   0
        a   X   a   d   0   0
        a   a   a   c   0   0
        a   a   a   c   25  0
        b   b   b   c   0   0
        0   12  12  0   0   0
        0   0   0   0   0   0

    SURFACE  : this is (1)*0.95^4 more damage
        0   0   0   0   0   0
        a   X   a   d   0   0
        a   a   a   c   0   0
        a   a   a   c   25  0
        b   b   b   c   0   0
        0   12  12  0   0   0
        0   0   0   0   0   0


## Step 2

Try to interleave a torpedo between all action and evaluate the probability to damage the opponent.

    Move at turn 0                           Future damage taken   Damage dealt to opp
    {MOVE N SILENCE},                        0.055                 0.0
    {TORPEDO,MOVE N SILENCE},                0.055                 0.8
    {MOVE N SILENCE,TORPEDO},                0.055                 1.5
    {SURFACE,MOVE N SILENCE},                1.055                 0.0
    {SURFACE,TORPEDO,MOVE N SILENCE},        1.055                 0.8
    {SURFACE,MOVE N SILENCE,TORPEDO},        1.055                 1.3
    ...   (568 lines)                        ...                   ...

## Step 3

A torpedo sent might reveal the position of the submarine. The probability of being damaged by the opponent is calculated assuming there is an equal chance for the opponent's presence in all positions of his presence map and the opponent will play aggressively (i.e.: move and torpedo if in range, even if he enters a mine field).

    Move at turn 0                           Future damage taken   Damage dealt to opp   Retaliation by opp
    {MOVE N SILENCE},                        0.055                 0.0                   1.0
    {MOVE N SILENCE,TORPEDO},                0.055                 1.5                   2.0
    {TORPEDO,MOVE N SILENCE},                0.055                 0.8                   2.0
    {SURFACE,MOVE N SILENCE},                1.055                 0.0                   1.5
    {SURFACE,TORPEDO,MOVE N SILENCE},        1.055                 0.8                   0.0
    {SURFACE,MOVE N SILENCE,TORPEDO},        1.055                 1.3                   0.0
    ...   (568 lines)                        ...                   ...                   ...

My retaliation to the opponent's torpedo is also evaluated to find situations where it is possible to wait for a better torpedo on the next turn.

## Final action chosen
In the end, the action chosen is the one that minimizes `futureDamageTakenFromMines - damageDealtToOpponent + damageReceivedFromOpponentRetaliation`.

Some key advantages of this search are:
- In extreme cases, the search remains well-behaved and insensitive to edge-cases:
--- If the opponent position is known with 100% certainty, the algorithm behaves like a **3-ply deep minimax**, for efficient fighting.
--- If the opponent position is not known, the algorithm behaves like a **5-ply deep space-filling-mine-avoiding** algorithm
--- In between, the search is able to balance both requirements
- It is possible to **balance aggressiveness, mine evasion, torpedo avoidance and silence usage**
- Silence may be tuned to "jump" over mine fields and evade incoming torpedoes: this fits the strategic objective to minimize silence usage overall
- Ability to find **natively all types of combinations** within the same framework, and to **combine objectives** such as using surface + silence away to both avoid a torpedo and not enter a minefield


## Tweaks

In no particular order, a large amount of tweaks were added to tune the evaluation function.

- The silence cost is largely reduced if a torpedo was launched
- Favor Move after Torpedo to charge faster
(((not finished)))
