Hey @Karim @Leonard — could I get a token bump to finish one submission?

I'm at 12.5 tokens and one gate away. Everything else is green: Fair task,
Solvable 4/10, No cheating, No environment blockers, Minimum runs 10/10,
Difficulty 40% (Medium), Long-horizon. Description 3/3, Solution 3/3,
Agent runs 3/3.

The only red is **No false positives** — 3 of the 4 passing runs were
flagged. I've already found and fixed the cause locally: the test patch I
uploaded was missing the guard that rejects an ordinary save on a trashed
record, which is exactly the "trashed-save rejection" the panel named. The
fix is written and verified — reference solution runs 170/0, and replaying
all 10 stored agent solutions against it leaves 3 solves, so Solvable and
Difficulty both still hold. I also closed the Auto Review's T3/T4 HIGH in
the same patch at no cost to the solves.

So it's a test-patch swap plus one more round of checks and rollouts to
submit. Would really appreciate the tokens 🙏

Draft: https://shipd.ai/quests/olympus/challenges/kh7f7qjjz9h7ps6ph52n5wcaas8cybht
