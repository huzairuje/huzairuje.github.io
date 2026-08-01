---
layout: post
title: "Learning Programming Again with Scratch 3: A Backend Engineer's Mind-Blowing Reset"
date: 2026-07-29 10:30:00 +0700
categories: programming education
tags: [scratch3, programming, education, game-development, mental-models, backend]
description: "A backend engineer returns to programming through Scratch 3, builds a jumping game, and rediscovers events, state, concurrency, debugging, and the joy of immediate feedback."
---

After years of writing backend systems, programming can start to feel like infrastructure around the thing you actually wanted to build. Before one useful behavior appears, there is a repository to clone, a runtime to install, a database to configure, an API contract to discuss, and a deployment pipeline to satisfy.

Then I opened Scratch 3.

I dragged a block that said `when space key pressed`, attached two small loops, clicked the green flag, and watched a character jump. There was no build step. The program was visible, the state was visible, and the result was immediate.

That experience was more than nostalgic. It was mind-blowing.

Scratch is often introduced as programming for children, especially ages 8 to 16, but [people of all ages use it](https://scratch.mit.edu/help/about/). For an experienced programmer, its simplicity is not a limitation. It is a way to remove years of accumulated tooling and meet the essential ideas again: events, state, loops, conditions, messages, timing, and feedback.

I used the Scratch Team's [How to Make a Jumping Game in Scratch](https://www.youtube.com/watch?v=1jHvXakt1qw&t=46s) tutorial as my way back in. The project is small: a character jumps over moving obstacles and earns points. Underneath that simple game is a surprisingly rich programming model.

<!--more-->

## Why learn programming again?

An experienced programmer does not need to relearn what a loop is. But we may need to remember what learning a loop feels like.

In backend work, a loop is buried inside a consumer, scheduler, retry policy, database scan, or library call. We reason through logs and metrics after the behavior has happened. In Scratch, a loop is a colored shape on the screen and its effect happens in front of us. Change `repeat 10` to `repeat 20`, click once, and the character jumps twice as high.

That tight loop changes how we think:

1. Make a small prediction.
2. Change one thing.
3. Run it immediately.
4. Observe what actually happened.
5. Adjust the model in our head.

This is programming before ceremony. It is also excellent education. Scratch describes its purpose as helping young people think creatively, reason systematically, and work collaboratively. Its tutorials are intentionally open-ended starting points, designed for learners to experiment and express their own ideas rather than merely reproduce a finished product.

## The jumping game: build the smallest playable idea

Open the [Scratch project editor](https://scratch.mit.edu/projects/editor/), then keep the goal deliberately small:

> Press Space to jump over an obstacle. Touching it ends the game. Passing it earns one point.

The official tutorial divides that into five behaviors:

- make a character jump;
- make an obstacle move;
- stop the game after a collision;
- add more obstacles;
- keep score.

This order matters. Each step produces something visible and testable. A learner never has to hold the entire game in their head before seeing a result.

### 1. Make the character jump

Choose a sprite and a backdrop. The tutorial uses a chick and a blue-sky scene, but the code does not care whether the character is a chick, a dinosaur, or a flying database mascot.

The first script is conceptually:

```text
when [space] key pressed
start sound [Chirp]
repeat (10)
  change y by (10)
end
repeat (10)
  change y by (-10)
end
```

The stage is a coordinate system. Increasing `y` moves the sprite up; decreasing it moves the sprite down. Two loops create a simple jump without introducing velocity, acceleration, or a physics engine.

For a production game, this motion is crude. For a first mental model, it is perfect. Input causes a sequence of state changes, and the screen renders the new state.

Before running it, ask a learner to predict:

- What will happen if `10` becomes `20`?
- Which number changes the height?
- Which number changes how smooth the movement looks?
- What happens if the second `-10` becomes `-5`?

Those questions turn block copying into reasoning.

### 2. Make the obstacle move

Add a second sprite for the obstacle. Place it on the right side, then glide it toward the left:

```text
when green flag clicked
forever
  go to x: (right edge) y: (ground level)
  glide (2) seconds to x: (left edge) y: (ground level)
end
```

The character does not move forward. The world moves toward the character. That small design choice creates the illusion of running while keeping the player's logic simple.

Now two scripts are alive at the same time. The character waits for keyboard input while the obstacle continuously crosses the stage. Scratch has introduced concurrency without first demanding a lecture about threads.

### 3. Stop on collision

The obstacle also watches for contact:

```text
when green flag clicked
forever
  if <touching [character]?> then
    stop [all]
  end
end
```

This is a complete rule:

```text
event + current state + condition -> outcome
```

The game does not need a central controller asking every object for an update. The obstacle owns the behavior related to its collision. That is compact, but it also introduces an important debugging lesson: behavior is distributed across sprites.

### 4. Add another obstacle

Duplicate the obstacle, then delay its first appearance:

```text
when green flag clicked
hide
wait (1.5) seconds
show
forever
  go to x: (right edge) y: (ground level)
  glide (2) seconds to x: (left edge) y: (ground level)
end
```

The duplicate reuses working behavior. The wait changes its phase so both obstacles do not overlap forever.

This is the moment the toy becomes a game. Difficulty emerges from the interaction of a few simple processes, not from one complicated algorithm.

### 5. Keep score

Create a `score` variable. Reset it when the green flag starts the game, then add one each time an obstacle completes a safe pass:

```text
when green flag clicked
set [score] to (0)

forever
  move obstacle across the stage
  change [score] by (1)
end
```

Now the project has a goal, a failure condition, and feedback. That is enough game.

Do not add menus, levels, accounts, a leaderboard, or a reusable game framework yet. Play the small version first.

## The mental model underneath Scratch 3

The biggest surprise for me was that Scratch did not hide programming. It made the runtime visible.

Here is the model I now use:

| Scratch concept | Mental model | Familiar backend idea |
|---|---|---|
| Stage | The world where state becomes visible | Running environment |
| Sprite | An object with state and behavior | Actor or service |
| Script | A behavior attached to an event | Event handler |
| Green flag | Start or reset the system | Process boot |
| `forever` | Keep this behavior alive | Worker loop |
| Variable | Mutable state | In-memory field or counter |
| `if` + sensing | Observe state and decide | Guard or business rule |
| Broadcast | Announce that something happened | Pub/sub event |
| Clone | Create another active instance | Spawn an actor |

These are analogies, not exact equivalences. A Scratch sprite is not a network service, and a broadcast is not Kafka. But the comparison reveals why the environment feels familiar to a backend engineer.

### Scratch is event-first

Many beginner programming exercises start at line one and run downward. Scratch starts with hats:

```text
when green flag clicked
when space key pressed
when I receive [game over]
when this sprite clicked
```

Nothing happens until an event activates a script.

That resembles real backend systems more than a sequence of arithmetic exercises does. A request arrives. A message is consumed. A timer fires. A process starts. The handler changes state or produces another event.

Scratch lets a learner build that model by making a chick jump.

### Concurrency is drawn, not hidden

Several Scratch scripts can run independently:

- the character waits for Space;
- obstacle one glides across the stage;
- obstacle two waits before starting;
- a collision loop checks whether the player lost;
- a score script updates progress.

Their relative timing changes the game. If the delay is wrong, obstacles overlap. If collision sensing is attached to a behavior that stops running, the game no longer ends. If two scripts reset the same variable, the score behaves strangely.

Backend engineers know these as coordination and state-ownership problems. Scratch makes them visible without stack traces, goroutine dumps, or distributed tracing.

### Each sprite owns part of the truth

In a conventional backend, we often search for the service or database that owns a piece of state. Scratch creates the same question in miniature:

- Does the character own jumping?
- Does the obstacle own collision?
- Who resets the score?
- Who decides that the game has ended?

There is no type system forcing a design. The stage simply shows whether the design works. This makes Scratch useful for teaching responsibility and decomposition before introducing classes, packages, or architectural diagrams.

### Time is part of the program

Backend code often treats time as an unpleasant external dependency. Scratch places it in the middle of the canvas:

```text
wait (1.5) seconds
glide (2) seconds
repeat (10)
```

A child changing `1.5` to `0.5` is doing load and difficulty tuning. A backend engineer changing it sees scheduling, ordering, and throughput in miniature.

## Why Scratch can humble an experienced backend engineer

Text code gives us powerful tools for controlling complexity. It also lets us hide behind abstractions.

Scratch removes many of those hiding places. There is no interface to name before the behavior exists. No dependency injection container. No generic repository. You drag an event, add a condition, mutate state, and watch whether the idea works.

That directness exposes several habits:

- We often reach for architecture before proving the interaction.
- We confuse knowledge of syntax with understanding of behavior.
- We tolerate long feedback cycles because they are normal in our jobs.
- We forget that visible progress is part of good learning.

The mind-blowing part is not that Scratch can imitate "real" programming. Scratch is real programming. The game has input, concurrent behaviors, mutable state, collision rules, failure, scoring, timing, and sound. What it removes is incidental complexity.

## How to use the tutorial for education

Watching the video and copying every block will produce a game. It will not automatically produce understanding.

A better learning sequence is:

### First pass: copy and finish

Follow the tutorial once. Pause often, reproduce the blocks, and play after every step. The goal is confidence and a complete feedback loop, not memorization.

### Second pass: rebuild without the video

Start a blank project and recreate the three essential rules:

1. Space makes the character jump.
2. An obstacle moves toward the character.
3. Touching the obstacle ends the game.

Getting stuck is useful here. It shows which part of the mental model is missing.

### Third pass: change one rule

Do not add a large feature. Change one variable and explain the result:

- make the jump higher;
- make obstacles faster;
- randomize the waiting time;
- lose one point on collision instead of stopping;
- use a broadcast to show a `Game Over` message;
- add a second key for a different action.

The modification is where ownership begins. The learner is no longer reconstructing someone else's program.

### Fourth pass: teach it

Ask the learner to explain:

- What starts each script?
- Which scripts run at the same time?
- Where does the score live?
- What condition ends the game?
- What would break if the obstacle stopped looping?

If they can answer while pointing at the blocks, they understand the program well enough to change it.

For a class or workshop, let learners work in pairs and rotate who controls the mouse. Ask for predictions before giving corrections. A bug is not a failed lesson; it is the moment the learner's mental model meets evidence.

## What I would build next—and what I would not

After the tutorial, I would add only one of these:

- random delay between obstacles;
- a gradually increasing obstacle speed;
- a `Game Over` broadcast and restart button;
- a high-score variable;
- a custom character and sound.

Each extension deepens an idea already present. Random delay teaches unpredictability. Increasing speed connects score to difficulty. Broadcasting separates the collision from the game-over presentation. A restart makes initialization and state reset impossible to ignore.

I would not begin by translating the project into JavaScript, adding realistic gravity, or designing a reusable engine. Those may be good later projects, but they move attention away from the reason Scratch works: a thought can become visible before motivation disappears.

## Programming with the lights turned on

Scratch 3 gave me an unexpected feeling: programming with the lights turned on.

Every active object was on the stage. Every behavior was attached to a visible sprite. Every timing decision could be changed with one number. When the model in my head was wrong, the game demonstrated it immediately.

For a new learner, that creates confidence. For an educator, it creates opportunities to ask better questions. For an experienced backend programmer, it strips away familiarity and reveals the computational ideas underneath our frameworks.

Build the jumping game. Follow the [video tutorial](https://www.youtube.com/watch?v=1jHvXakt1qw&t=46s) once, then close it and rebuild the idea from memory. When the obstacle moves, the character jumps, and the score changes, pause before adding anything else.

That small moment is the whole craft: imagine a behavior, express it precisely, observe reality, and learn from the difference.

Sometimes the best way to become a better programmer is to become a beginner again.

## Resources

- [How to Make a Jumping Game in Scratch — Scratch Team](https://www.youtube.com/watch?v=1jHvXakt1qw&t=46s)
- [Jumping Game — Scratch Foundation Learning Library](https://www.scratchfoundation.org/learn/learning-library/jumping-game)
- [Scratch project editor](https://scratch.mit.edu/projects/editor/)
- [About Scratch](https://scratch.mit.edu/help/about/)
- [How Scratch 3.0 tutorials support learning](https://sip.scratch.mit.edu/tutorials/?eventId=1a27b925-9525-46ab-904e-f8005f21b566)
