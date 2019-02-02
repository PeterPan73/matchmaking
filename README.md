# PROJECT
Create a matchmaking service given hypothetical player information.

# INSTALLING
npm install package.json

# RUNNING TESTS
npm test

# STARTING THE SERVER
node app/server.js

## A complete client matchmaking example
If you run:

node app/client.js

While this is running, many clients will attempt to perform all the requests necessary of a complete matchmaking example.

Otherwise, a full example is: 

1. Start the server in another tab

2. Run this curl command which simulates starting the search
  curl -d '{"id":"1", "mmr":"0"}' -H "Content-Type: application/json" -X POST http://localhost:8099/start

3. Run the same command but with a different ID (simulates another client searching):
  curl -d '{"id":"2", "mmr":"0"}' -H "Content-Type: application/json" -X POST http://localhost:8099/start

4. Poll the server and ask if you've been paired with anyone yet. If so, you'll get your opponent's ID:
  curl -d '{"id":"2", "mmr":"0"}' -H "Content-Type: application/json" -X POST http://localhost:8099/status

  Note: At this point you've been matched and you would be redirected to a lobby that only you and your opponent have access to.
      The actual redirecting to a match lobby isn't implemented.

5. Tell the server you have your opponent's information and that the server can forget about you
  curl -d '{"id":"2", "mmr":"0"}' -H "Content-Type: application/json" -X POST http://localhost:8099/update

  Note: After you run ^ you will be able to restart matchmaking.

### MY NOTES ABOUT MATCHMAKING

-Matchmaking is about bringing two equally (or psuedo-equally) skilled players together.
-The skill of a player is set by MMR (Matchmaking rating), which is a positive numerical ranking. This ranking is calculated with an algorithm by the game developer or publisher.
-After every match, a player's MMR is updated based on the results of the match and other factors.
-A lot of the MMR algorithms assume player skill is normally distributed.
  With this assumption ^ I can calculate the expected win probabilities of two players.

### WIN PROBABILITY CALCULATION

The probability that a player (p1) will beat an opponent (p2) is:

P (p1 > p2| s1, s2) = CDF((s1 - s2) / sqrt(s1.sigma**2 + s2.sigma**2))

where:
  s1 & s2 are skill estimates;
  CDF is a gaussian cumulative density function
  the denominator is the square root of the sum of variances of the two players.

Source: https://www.microsoft.com/en-us/research/wp-content/uploads/2007/01/NIPS2006_0688.pdf

Note: In extreme cases, high variance in skill or low variance in skill leads to bad results.

# IMPLEMENTATION
Creating bins for skill ranges is how I implemented the matchmaking service

The pseudo-code for a binned matchmaker would be:
  - Create bin ranges for the entire MMR range of your players (ie 0-10,000); Each bin range gets its own queue.
    You can tailor your bin ranges to have equi-player-pool-sized buckets, based on MMR frequency info about your players.

  - When a player decides to search, lookup their bin based on their MMR info and retrieve the queue

  - If there are no players in the queue, add them to the top of the queue
    - Repeatedly poll the bin pool until a new player is added (or keep a websocket connection alive and wait to be updated)

  - Else pick the player at the top of the queue and dequeue that player
    - Tell both clients a match is made

  -As the user, update the server you have your opponent info / lobby info
    - Server "forgets" about user and that user can now start over again matchmaking

Apart from the intial bin-range creation, this doesn't require calculations to be made. There's no iterating over players to calculate win probabilities, and if the player pool is a queue, dequeue and enqueue both are of O(1) complexity. The bins can also easily be stored in memory (you only need the player ID and queue structure). The player ID is usually just the Steam ID which is a 32 bit number.

There are problems with this implementation, however. If you have a player with a very well defined MMR, that it at either edge of a bin range, then that player will either win or lose with a high probability.

For example:

Consider a bin with MMR ranges of [950-1000], and a player with 950 MMR or 1000 MMR and 1 standard deviation.
Take an example opponent of 975 with standard deviation of 30.
The 950 player will lose an estimated 80% of the time. The 1000 player will win 80% of the time.
In the most extreme example, the 1000 player will win 100% if playing against the 950 player.

