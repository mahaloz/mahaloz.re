---
title: "Tasteless 21: Godot Game Hacking in Tasteless Shores"
layout: post
tags: [ctf, reversing, game-hacking]
feature_image: /assets/images/ctf/tasteless-21/feature_image.png
description: "A series of game hacking challenges hosted in Tasteless 21. Decompiling, modifying, and recompiling Godot scripts to abuse client-side computations for a multiplayer game. Fly hacks, no-damage, super speed, and rng abuse."
---
# Introduction
This weekend we played Tasteless 21 and they had a great series of challenges. This year they spent a painstaking amount of time making a 3D game called `Tasteless Shores`. It was big enough to be it's own category in the game, hosting a total of 6 challenges in the same game. My teammates and I completed 4/6, and it was super fun. In this writeup I'll try to mostly focus on how to do the game hacks that made solving the challenges possible. 

## Collaborators:
- [@clasm](https://github.com/WGibbs15)
- @lucas_baizer
- @lbshine

![]({{ site.baseurl}}/assets/images/ctf/tasteless-21/challenges.png)

# Challenge Description:

The entire challenge had its own page for the CTF. It had 3 files associated with it.
The client/server game binary, and its resources:
> Linux
Download: https://s3.eu-central-1.amazonaws.com/tstlss.tasteless.eu/tasteless-shores.x86_64 and https://s3.eu-central-1.amazonaws.com/tstlss.tasteless.eu/tasteless-shores.pck
>
> Run ./tasteless-shores.x86_64 in the folder where the tasteless-shores.pck exists

And the "master server" that ran the game server:

> Linux: https://s3.eu-central-1.amazonaws.com/tstlss.tasteless.eu/server
> 
> Run local setup:
> 
> Masterserver: ./server
> 
> ameserver: ./tasteless-shores.x86_64 server
> 
> Game: ./tasteless-shores.x86_64 local

They also provided a different binary for each OS. They had MacOS, Windows, and Linux. You can find all files relevant to my solve in my [ctf_files](https://github.com/mahaloz/ctf_files/tree/main/ctfs/tasteless-21/tasteless-shores) on GitHub.

# Overview of components

At first, we were baited into analyzing the binaries provided, because we thought that the `.pck` file was just a resource pack. We spent around 2 hours looking at the binaries before we decided to actually look more into the `.pck` file, which was critical to this entire challenge set. For reference, the game binary is **HUGE**:

```bash
▶ du -h tasteless-shores.x86_64
38M	tasteless-shores.x86_64
```

It was insane. IDA could not load it fast enough. From a meta-game perspective, it was strange that they gave us a binary per-OS. This is not normal if you were supposed to reverse the binary, since that would change the binary per-player potentially. This made us start thinking that maybe the `.pck` was more important than we thought. We looked at the output of the binary again:

```bash
▶ ./tasteless-shores.x86_64
Godot Engine v3.3.3.stable.official.b973f997f - https://godotengine.org
```

Which lead us to this link about [godot and GDScripts](https://docs.godotengine.org/en/stable/getting_started/workflow/export/exporting_pcks.html). Yes, this `.pck` was actually was more than just resource pack, so much so that it could contain full game logic. You could run the game binary in both client and server mode, so this made even more sense.

Lastly, the other `server` binary the organizers provided was for managing multiple game servers and connection routing. Think of it as a middle-man, or [Bunjee Server](https://docs.godotengine.org/en/stable/getting_started/workflow/export/exporting_pcks.html), like in Minecraft.

# Game Logic

Funny enough, the rest of this challenge has nothing to do with the binaries above. The only thing that mattered for the rest of the challenges was the `.pck` file. We found a [decompiler](https://github.com/bruvzg/gdsdecomp) for GDScripts pretty fast from google. After using the decompielr feature, we had a full godot project:

```bash
▶ tree -L 1
.
├── assets
├── default_env.tres
├── export_presets.cfg
├── game
├── icon.png
├── icon.png.import
├── project.godot
└── scenes
```

With most game logic things in `game`:

```bash
▶ tree ./game
./game
├── game.gd
├── items
│   ├── fishing_rod.gd
│   ├── item.gd
│   ├── items.gd
│   ├── melee_weapon.gd
│   ├── range_weapon.gd
│   └── weapon.gd
├── net
│   ├── client2.gd
│   └── server2.gd
├── player
│   ├── player.gd
│   ├── player_controller.gd
│   ├── spring_arm.gd
│   ├── water.gd
│   └── waves.gd
├── screen
│   ├── character.gd
│   ├── credits.gd
│   ├── load.gd
│   └── login.gd
├── ui
│   ├── center_notification.gd
│   ├── chat_container.gd
│   ├── compass.gd
│   ├── compass_item.gd
│   ├── inventory.gd
│   ├── minimap.gd
│   ├── player_ui.gd
│   ├── ui.gd
│   └── world_label.gd
└── unit
    ├── enemy.gd
    ├── npc.gd
    ├── skeleton.gd
    ├── unit.gd
    ├── unit_remote.gd
    └── unit_spawner.gd
```

As you can probably guess, `server2.gd` is run on the server. `client2.gd` is run on the client (your local machine)... This means we can modify anything in client (and anything it loads that is not the `Server`).


# Speed Hacks POC

To prove we could _actually_ modify the client, we first started with a simple proof-of-concept by making speed hacks. We modified the player controller, which interfaced with game to give you movement, here:

## player_conroller.gd:11
```js
var moveSpeed:float = 2.0
var jumpForce:float = 5.0
var boatSpeed:float = 25.0
var currentSpeed:float = moveSpeed
var vel:Vector3 = Vector3()
var paused = false

func _ready():
	if true:
		moveSpeed = 5.0
		jumpForce = 10.0
		boatSpeed = 50.0
```

We changed all the `moveSpeed` to 10. We recompiled with the earlier mentioned [decompiler chain](https://github.com/bruvzg/gdsdecomp) This allowed us to _zoom_ around the spawn island at top speeds. It looked really hacky as we speed glitched everywhere.


# Jump Hacks

While we were hacking speed, we thought we should have jump hacks as well. We modified the same file. We set the `jumpForce` to `100`. This is what it looked like:

![yeeet]({{ site.baseurl}}/assets/images/ctf/tasteless-21/jump_hacks.png)
Me, flying hundreds of blocks from one jump.


# Fly Hacks

As one more fun thing, we added fly hacks (really just the lack of gravity). As is the theme, we modified the player controller:

## player_controller:78
```js
func _physics_process(delta):
	vel.x = 0
	vel.z = 0
	
	var input = Vector3()
    
	vel.y += - 9.8 * delta
```

We modified the velocity and y so that we only ever went up, so now velocity would be greater than 0. We no longer fall after jumping. 


![yeeet]({{ site.baseurl}}/assets/images/ctf/tasteless-21/fly_hacks.png)
Me, looking down on the peasants who cant fly.


# Flag Hacks (challenge: Boat)

We decided it was time to get some flags, so we first wanted the flag from the fisherman. Looking at the code, we saw how the fisherman gave you the boat to get to the island with the flag:

## boat.gd:10
```js
func interact():
	if "FLAG_BOAT" in Client.player.marker:
		Ui.ask("That is some real leet fishing skillz you got. You have a boat now to enter the water.")
	else :
		Ui.ask("I've been a fisher myself, young lad...\n\nBut to get a boat, you need to prove yourself.\n\nThere is fish in the west, but you gotta fish in the Lake'o'despair to prove yourself being worth it.")
		Client.interact(111)
```

The fisher checked for the `FLAG_BOAT`, which you could only get if you caught a fish in some fishing area:

## fishing_rod.gd:29
```js
func fish():
	Client.player_controller.paused = false
	for area in $FishArea.get_overlapping_areas():
		if area.has_method("lake'o'despair") or true:
			Ui.show_note("Now I am a true fisher")
			Client.start_fish(area.call("lake'o'despair"))
		else :
			Ui.show_note("Only small fish here...")
		return
```

`start_fish` was supposed to give you the `FLAG_BOAT`, but it was impossible to run it, since it errors-out each time you run it. We did not have the patience to _reallY_ figure out why, but we knew it had something to do with fishing in all the fishing areas. 

Instead, we wanted to directly modify our players flags, which was checked in `player.marker`. In `player.gd`, you can actually see where the initial markers are, which we observed over wireshark connections as well:

## player.gd:1
```js
extends "res://game/unit/unit.gd"

const Game = preload("res://game/game.gd")
const Unit = preload("res://game/unit/unit.gd")

signal water_entered
signal water_exited

onready  var boatMesh:MeshInstance = $Boat

var boat = false
var drowning = false
var marker = {
	"FLAG_EYES":true, 
	"FLAG_BOAT":true,
}
```

On the last line there, we added the `FLAG_BOAT` as seen, which was the flag the fisher checks for.

Now there was only one more thing, and thats' the fact that the chest is not created unless you **actually** fished once:

## server2.gd:157
```js
func _handleFish(pid):
	if not has_node("player_" + str(pid)):
		prints("error, unknown player fish", pid)
		return 
	var node = get_node("player_" + str(pid))
	spawn_chest(node, "FLAG_BOAT", flags["FLAG_BOAT"].global_transform.origin)
```

This code is hit once you do `start_fish` on the client from the fishing rod item. This helped us understand the framework, since the server gets this from fish request:

## client2.gd:511
```js
func start_fish(target):
	socket.put_u8(MsgClientFish)
```

Remember, we control the client. We modify what happens when you attack to allow us start a fishing event, because we are wack:

```js
func start_attack(target):
	socket.put_u8(MsgClientFish)
```

So now you just punch once, it activates a fish event and spawns the chest, then you use your hacks to walk over the water and grab the flag from the chest.

![flagggg]({{ site.baseurl}}/assets/images/ctf/tasteless-21/flag_hacks.png)
Me, getting that flag.

At this point we conclude we have the ability to send arbitrary event requests. We can request to fish by attacking. We can request to do literally anything by overriding attacking as our trigger. 


# Challenge: Skull Island

Using our knowledge of _stuff_, we realized the flag for the Skull Island is in the _eye_ (go figure). We used our fly hacks to propel/god ascend to the eye and grab the flag. We also had to make sure we kept the `FLAG_EYES` in our player `markers`. 

![]({{ site.baseurl}}/assets/images/ctf/tasteless-21/skull_island.png)


# TP Hacks (challenge: Conch)

Using our event request knowledge from earlier, we decided it was time to stop flying around the map and instead time to start teleporting around the map. To make this possible, we used the same client request overrides found in the boat challenge. Instead of overriding attacking, this time we decided to override how using the chat worked. Each time a message was sent from the user, `chat(whisper, msg)` was called. We introduced a handler:

## client2.gd: 574
```js
func chat(whisper, msg):
    _msg_handler(msg)
```

The handler, and its dependencies was added as well to client:

```js
func _teleport_to_coor(x, y, z):
	player.global_transform.origin.x = x
	player.global_transform.origin.y = y
	player.global_transform.origin.z = z

func _msg_handler(msg):
	if msg.begins_with("/tp"):
		var m_arr = msg.split(" ")
		var x = float(m_arr[1])
		var y = float(m_arr[2])
		var z = float(m_arr[3])
		var out_msg = "TP(X,Y,Z): "+msg
		_teleport_to_coor(x,y,z)
		print(out_msg)
```

Now we could just use the chat to teleport:

![]({{ site.baseurl}}/assets/images/ctf/tasteless-21/tp_hacks.png)

Now this is only useful if you actually know where you want to teleport, and lucky for us, there is a challenge which needs this functionality.

In the level `Conch` you are tasked with finding a hiding rabbit, which is requested of you in the `quest_fish.gd`. Once you get the conch, a special event occurs that places the rabbit you are looking for at a random coordinate on the map:

## conch.gd:10
```js
func _init():
	rabbit = Vector3(rand_range( - 100, 100), rand_range( - 100, 100), rand_range( - 100, 100))
	

const rabbit_distance = 0.1

func use(collider, from):
	if OS.get_ticks_msec() - lastAttackTime < attackRate * 1000:
		return false
	lastAttackTime = OS.get_ticks_msec()

	Server.conch(from, from.global_transform.origin.distance_to(rabbit))
	if from.global_transform.origin.distance_to(rabbit) < rabbit_distance:
		Server.spawn_chest(from, "FLAG_CONCH", rabbit)

	return true
```

That init function is run every time you get the conch. The conch can also be used to get you the _Euclidean Distance_ to the rabbit. It's passed to the `conch` function in the same file, but will just print `Close` or `Far` in a hot an cold way. We modify it to print useful data:

## conch.gd:28
```js
static func conch(distance, player):
	var x = player.global_transform.origin.x
	var y = player.global_transform.origin.y 
	var z = player.global_transform.origin.z
	
	var notif = String(distance) + ": (" + String(x) + "," + String(y) + "," + String(z) + ")"
	Ui.show_note(notif)
```

This will, on each use, get our current 3d coordinates and print the distance from our point to that point in 3d space. With this information, we can actually [triangulate](https://en.wikipedia.org/wiki/Triangulation) the 3d position to 1 point estimate just py trial and error.

![]({{ site.baseurl}}/assets/images/ctf/tasteless-21/locate_hacks.png)
Lucas, locating the rabbit

Getting within a single digit of the rabbit is actually not good enough. The check sees if you are within `0.1` distance of the rabbit, which means the only real way you can get that close is by teleporting to its exact location. Using a common method of finding the [intersections of spheres in a 3d space](https://math.stackexchange.com/questions/562240/how-to-find-the-intersection-of-three-spheres-full-solutions), we can get a decent point guess from just sampling three points in the 3d space. We used this [stackoverflow code](https://stackoverflow.com/questions/1406375/finding-intersection-points-between-3-spheres) to find the intersection of three spheres:

```python
import numpy                                             
from numpy import sqrt, dot, cross                       
from numpy.linalg import norm                            

def trilaterate(P1,P2,P3,r1,r2,r3):                      
    temp1 = P2-P1                                        
    e_x = temp1/norm(temp1)                              
    temp2 = P3-P1                                        
    i = dot(e_x,temp2)                                   
    temp3 = temp2 - i*e_x                                
    e_y = temp3/norm(temp3)                              
    e_z = cross(e_x,e_y)                                 
    d = norm(P2-P1)                                      
    j = dot(e_y,temp2)                                   
    x = (r1*r1 - r2*r2 + d*d) / (2*d)                    
    y = (r1*r1 - r3*r3 -2*i*x + i*i + j*j) / (2*j)       
    temp4 = r1*r1 - x*x - y*y                            
    if temp4<0:                                          
        raise Exception("The three spheres do not intersect!");
    z = sqrt(temp4)                                      
    p_12_a = P1 + x*e_x + y*e_y + z*e_z                  
    p_12_b = P1 + x*e_x + y*e_y - z*e_z                  
    return p_12_a,p_12_b                       
```

We simply made a estimated triangle around the closest points, got their distances (which is the radi), and used the points in this equation. This returned to us a point. Using the earlier mentioned `/tp` command we introduced, we teleported to the exact point of the rabbit and got the flag. 

# RNG Hacks (challenge: Thrybrush)

The last flag we had time to get in this CTF was the `Thrybrush` challenge. Right from the get-go we found the npc that was responsible for this challenge on Melee Island. His code can be found in the `guybrush.gd` file. He implements a chat method that allows you to talk to him:

## guybrush.gd:10
```js
func chat(from, msg):
	if not (from.remote_id in players):
		return 
	
	var bw = players[from.remote_id]
	var next_try = bw.dig()
	var insult = insults.keys()[next_try]
	if msg == insults[insult]:
		bw.distance -= 1
	else :
		bw.distance += 1

	if bw.distance <= 0:
		Server.spawn_chest(from, "FLAG_BIGWHOOP", Server.flags["FLAG_BIGWHOOP"].global_transform.origin)
		Server.chat(remote_id, from.remote_id, "you are faster than your shadow it seems!")
	else :
		Server.chat(remote_id, from.remote_id, insult)

func interact():
	Ui.ask("he speaks an ancient language...")
```

Currently, its impossible to talk to him, so we need to implement the `whisper` variable in the cold `chat` method found in client:

```js
func chat(whisper, msg):
	var should_whisper = _msg_handler(msg)
	if should_whisper:
		whisper = 31337
		var m_arr = msg.split(" ")
		var key = int(m_arr[1])
		var insult = insults.keys()[key]
		msg = insults[insult]
	socket.put_u8(MsgClientChat)
	socket.put_u64(whisper)
	socket.put_u8(msg.length())
	socket.put_data(msg.to_utf8())
```

Ignoring the use of the `insults` dict, which we will explain soon, we can now send him messages and receive insults back from him. Now onto how to get the flag. Looking at the earlier snippet starting from line 10 in `guybrush.gd`, we can see that the way you get the flag has to do with you predicting his next message:

## guybrush.gd:14
```js
var bw = players[from.remote_id]
	var next_try = bw.dig()
	var insult = insults.keys()[next_try]
	if msg == insults[insult]:
		bw.distance -= 1
	else :
		bw.distance += 1
```

The objective is to guess his next message, so that `bw.distance` is reduced. You want it to equal 0. Here is the list of messages he can say, and how he generates a random order of insults:

## guybrush.gd:32
```js
ar insults = {
"You fight like a dairy Farmer!":"How appropriate. You fight like a cow!", 
"This is the END for you, you gutter crawling cur!":"And I've got a little TIP for you, get the POINT?", 
"I've spoken with apes more polite than you!":"I'm glad to hear you attended your family reunion!", 
# [truncated]
}

var players = {}

class BigWhoop:
	var pos = Vector3.ZERO
	var rotation = 0as int
	var distance = 10as int

	func _init():
		pos.x = randi() % 255
		pos.y = randi() % 255
		pos.z = randi() % 255

	func dig():
		var hint = (int(pos.x) ^ ((int(pos.x) << 4)) & 255) & 255
		pos.x = pos.y
		pos.y = pos.z
		pos.z = rotation
		rotation = int(pos.z) ^ hint ^ (int(pos.z) >> 1) ^ ((int(hint) << 1) & 255)
		return rotation % 64
```

He has 64 different insults. On his initialization he creates 3 random bytes `(x,y,z)` which are used as the type of IV for this rng algorithm. Its a recursive type algorithm that creates a new state based on the last state, creating a state chain. Each state outputs a `rotation % 64`, which is the lookup index for the insults. Each time you ask for an insult, it progresses to the next state. At face value, a 1/64 guess is not that bad, but remember that each time you get the guess wrong, you MUST get it right 2 times in a row to account for that loss. At a minimum you need `10` correct guesses to win based on the initial `distance` value. That would be a `1/1152921504606846976` chance of winning. That's no good. So, let's conceptualize how this algorithm is flawed.

Here is a visual of what happens:

```
X = (x, y, z, rotation)

dig(X0) -> X1
        dig(X1) -> X2
            ...
                dig(Xn-1) -> Xn
```

We want to know `X0`, given `X1` ...  `Xn`. From the init, it's clear that `X0` is legitimately 3 bytes of randomness, but if we track all the possible `X0` that will generate `X1` that will generate `X2`, it will highly reduce the set since all the `X0` that does not lead to `X2` will be filtered out. We can expand this idea to `Xn` by generating the full set of `X0` and tracking which `X0` make it to `Xn`. To be concise, we care about the `X0` that reaches `Xn`.

After some trial and error, it became clear that after `n = 8`, there are only two possible points that `X0` could be. This is a 50% chance to have a seed that will generate only 100% correct guesses. That's really good, since we can actually try both `X0` guesses against the npc. Here is the python code we wrote to solve for `X0`, called the seed, and generate the next sequences given 8 observations:

```python
# truncated file 

def dig(x, y, z, rot):
    hint = (x ^ (x << 4) & 0xff) & 0xff
    x = y
    y = z
    z = rot
    rot = z ^ hint ^ (z >> 1) ^ ((hint << 1) & 0xff)

    return rot % 64, (x, y, z, rot)

def get_nth_dig(x, y, z, rot, n):
    output = []
    for _ in range(n):
        out, point = dig(x, y, z, rot)
        x, y, z, rot = point
        output.append(out)

    return output

def solve(obs):
    # you need approximately 8 rounds to have a 50/50 chance
    round_sets = {i: {} for i in range(len(obs))}

    # initialize round 0 with a full 3 byte range
    rot = 0
    solns = {}
    for x in tqdm(range(0xff)):
        for y in range(0xff):
            for z in range(0xff):
                out, point = dig(x, y, z, rot)
                if out == obs[0] and point[2] == 0:
                    # nextround -> thisround
                    solns[point] = (x, y, z, rot)
    round_sets[0] = solns

    # run as many rounds as observations left
    for i in range(1, len(obs)):
        solns = {}
        for soln in tqdm(round_sets[i-1]):
            out, point = dig(*soln)
            if out == obs[i]:
                solns[point] = soln
        round_sets[i] = solns

    # use a backwards chain to verify the seed is real
    for k in tqdm(round_sets[len(round_sets)-1]):
        last_round = k
        for i in range(len(round_sets)-1,-1,1):
            if last_round in round_sets[i]:
                last_round = round_sets[i][last_round]
        else:
            print(f"[+] FOUND SEED: {last_round}")
            yield last_round


if __name__ == "__main__":
    observations = [
        "Heaven preserve me! You look like something that's died!",
        "This is the END for you, you gutter crawling cur!",
        "Killing you would be justifiable homicide!",
        "My tongue is sharper than any sword.",
        "My skills with a sword are highly venerated!",
        "My attacks have left entire islands depopulated!",
        "You're the ugliest monster ever created!",
        "I've heard you are a contemptible sneak.",
    ]
    idx_obs = []
    for obs in observations:
        idx_obs.append(reverse_lookup_insult(obs))

    print("[+] Inverting and composing init state:")
    print(idx_obs)

    print("[+] Starting solve...")
    init_vals = solve(idx_obs)

    print("[+] Getting next sequence...")
    for init_val in init_vals:
        print("[+] Extended Sequence: ")
        print(get_nth_dig(*init_val, len(idx_obs)*2 + 30))

    # convert_seq_to_words(get_nth_dig(*init_val, len(idx_obs)*2 + 11)[len(idx_obs)+1:])
    import ipdb; ipdb.set_trace()
```

As a fast breakdown, the real magic happens in the `solve` function, that takes a series of insults you got from the npc. You need him to insult you 8 times, as said before. The solve starts by generating the full set of possible `X0`, then uses each round of observed conversation to reduce the set to only `X0` that can generate things. Lastly, in a nice confusing recursion-like loop, we use a dict for each round to do a reverse loopkup. If we make it all the way down to round 0 dict, that means the point is a possible seed. 

Here is the output of this script for the dialog interaction we recorded:

```python
▶ python3 solve_insults.py 
[+] Inverting and composing init state:
[47, 1, 36, 22, 60, 49, 37, 13]
[+] Starting solve...
[+] Getting next sequence...
[!] FOUND SEED: (124, 241, 101, 77)
[+] Extended Sequence: 
[47, 27, 41, 58, 6, 24, 31, 62, 43, 22, 12, 8, 33, 43, 10, 55, 31, 29, 13, 2, 18, 12, 45, 61, 53, 27, 17, 14, 6, 56, 7, 22, 55, 4, 63, 26, 30, 61, 50, 37, 21, 40, 10, 16, 55, 20]
[!] FOUND SEED: (252, 241, 101, 77)
[+] Extended Sequence: 
[47, 27, 9, 10, 14, 20, 21, 33, 3, 62, 14, 58, 50, 9, 63, 46, 15, 3, 19, 8, 45, 14, 12, 50, 12, 56, 16, 14, 29, 27, 6, 23, 43, 35, 56, 45, 54, 24, 60, 5, 29, 59, 34, 12, 61, 30]
```

After this, it was as simple as inputing the numbers we knew was next in the sequence, due to the earlier done `chat` override that would replace the number with the insult from the insults dict. It takes about 18 insults (after the 8 observed) to win. Just like the Skull Island challenge, you must have `FLAG_BIGWHOOP` in your `player.markers` to be able to spawn the chest, so use the earlier hack for flags. Find the full solve script [here](https://github.com/mahaloz/ctf_files/blob/main/ctfs/tasteless-21/tasteless-shores/solve_insults.py).

# Thanks where thanks is due

I need to give a huge shutout to both Lucas and Wil. For the majority of the game I had to continually send them my game patches and they had to run it since I had no work setup. They also were a huge part in solving these levels. As usual, this solve was only possible with friends :). 


# Conclusion

This was a really fun challenge, and a classic example of game hacking. I did wish there was more binary related stuff, but I know its hard to dump a lot of time into making a game engine. I also wish we had more time to solve part 5 and 6 of this challenge set, since I'm sure it must have been great. We got 10th, gg. 

![]({{ site.baseurl}}/assets/images/ctf/tasteless-21/scoreboard.png)



