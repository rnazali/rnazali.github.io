---
layout: post
title:  "Creating my first game for Game Off 2023"
date:   2023-11-30 10:15:27 +0700
---

I found out that GitHub is holding annual game jam called **Game Off**, where people try to create a game
on the month of November for a certain predetermined theme.

Personally, I think one month is a very short time to make a game that excel in all key components: 
playable, reliable (no game-breaking bug), fun, immersive.

I've always been interested in GameDev, so I thought I'd give it a try.

Since I've mostly used python in the past years, I picked Godot Engine. 
The open-source license is also a plus, as basically I can own both the game engine and the game I've created, royalty-free.

This year's [Game Off 2023](https://itch.io/jam/game-off-2023){:target="_blank"} theme is 'SCALE'.

Naturally, I'm not aiming to be the #1 developer, but rather to demonstrate 
whether I could create & finish something that are playable, with optimization and immersion deliberately left behind.

In around 16 days, I [coded](https://github.com/rahmatnazali/colonite){:target="_blank"} a simple unit simulation with a generic finite state machine.
The unit can switch its state according to the surrounding situation, and we (the user) basically just watching them going against each other.

In Godot, a state machine implementation looks like this:

```gdscript
extends Node
class_name StateMachine

@export var enabled: bool = true
@export var initial_state: GenericState

var current_state: GenericState
var states: Dictionary = {}

func _ready():
    for child in get_children():
    if child is GenericState:
        states[child.name.to_lower()] = child
        child.transition.connect(on_child_transitioned)
    else:
        push_warning('StateMachine contains child which is not `State`: ', child)

    if initial_state != null:
        initial_state.enter()
        current_state = initial_state
    else:
        push_warning('The initial_state` variable is not assigned,')

func _process(delta):
    if enabled and current_state != null:
    current_state.update(delta)

func _physics_process(delta):
    if enabled and current_state != null:
        current_state.physics_update(delta)

func on_child_transitioned(source_state: GenericState, new_state_name: StringName):
    if current_state != source_state:
        return

    var new_state: GenericState = states.get(new_state_name.to_lower())
    if new_state == null:
        push_warning('Called transition to state "' + new_state_name + '" state that does not exist in the `states` dictionary.')
        return
	
    if current_state != null:
        current_state.exit()
	
    new_state.enter()
    current_state = new_state
```

<br>

To give some gamification, I added resource to spawn the unit and declare a goal to get rid all the other team, and that's it.

It's playable in browser, Windows, and Linux platform:

<iframe frameborder="0" src="https://itch.io/embed/2379186?linkback=true" width="552" height="167"></iframe>

Going for a solo indie game developer may be a nice idea, even though I know the path would be harsh.
May this very first project lays the stepping stone towards it.
