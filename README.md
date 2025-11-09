# Ramosa

A small, opinionated node-based dialogue editor for games. It's built for branching dialogues with conditions, traits, and with  a straightforward import/export flow. The primary goal is making branching dialogues simpler to visualize and to reason about (compared to, say, writing raw JSON).

It's not general-purpose since it was built for a tailored use case, but it's pretty extensible even though it's simple.

Each node represents a single line of NPC dialogue, and each node has zero or more player choices. Choices can optionally specify the following dialogue node, traits (e.g. what the user-picked option represents in the player, e.g. aloofness, kindness, etc.), or conditions (e.g. whether the player has found a her glasses).

![Dialogue tool usage example](/public/demo.png "Demo of dialogue tool")

