# The Wepwawet standard

*What an app's API needs so it can be used by voice.*

Version 2, October 2026. Written from taking five real apps through by voice (two chore apps, a
history app, a calendar of religious celebrations, and the weather brain) with Wepwawet Veil, the
house intent router. Version 2 adds the actions a person asks for (expectation 14), how long an app's
data stays good (15), what sound an app takes in (16), an app's spoken name (1), and asking an app whether a request is its own (11). Each expectation below came up because an app didn't meet it, and says what
went wrong. The router follows this paper, but the paper stands on its own: it's for anyone
building an app that people will talk to through a voice assistant.

## What it is, and why it matters

Wepwawet extends OpenAPI; it doesn't replace it. An app keeps the API document it already has and
adds one section, `x-wepwawet`, that tells a voice system what the app is, what its things are
called and which words ask for which command. OpenAPI allows sections whose names start with `x-`,
so tools that don't know this one ignore it.

That matters because of where a request goes. A voice system that can read an app's own
description can answer from the house's apps directly: quickly, on the house's own network, with
no code written for that app. Only the questions that really need one go on to a large model in
the cloud (Gemini, Google and the like). Less goes out, answers come back faster, and the heavy
model is kept for the heavy questions.

## Why voice is different

A person typing into an app sees the screen: which chore, which project, whether it worked. A
person talking to an assistant sees nothing. The words come through speech-to-text, which mishears
("add a chore" as "at a chore", "test" as "task"), and the assistant has to turn them into one exact
API call, then say truthfully what happened. So a voice-friendly API has to let the assistant:

- find out what the app is and what it offers, without code written for that app;
- match a spoken name to exactly one of the app's things, or know it can't;
- make the change in one clear step, and know for certain it happened;
- notice when the app has changed underneath it.

## The expectations

Each has a **why** (what we ran into) and a **what to do**.

### 1. Describe yourself

**Why.** Without a description, every app needs hand-written glue in the voice system: which of its
screens matter, what it can do for the house, where its people and projects are listed.

**What to do.** Add an `x-wepwawet` section to your OpenAPI (or Swagger) document (the format is
below). OpenAPI allows sections whose names start with `x-`; tools that don't know this one ignore it.
Give the name the app is called by out loud in `spoken:` (a chore app called "Vikki"), which can
differ from its title.

### 2. Say your version, and keep it current

**Why.** An update can change what a command does even when its name and fields stay the same. A
voice system that knows your version can say "the chore app changed from 0.1 to 0.2: look at its
commands" before something breaks in someone's kitchen.

**What to do.** Put your real software version in the spec's `info: version:` and change it with
every release. A framework's default (FastAPI's `0.1.0`) tells nobody anything.

### 3. List your things, each with an id and a name

**Why.** People speak names ("the gutters", "Mara"); APIs want ids. The assistant needs a list to
match against, and the people in your app must be mapped to the people in the house.

**What to do.** A `GET` for each kind of thing a person would name (people, projects, items), each
item with a stable id and its human name. Say in `x-wepwawet: lists:` which command lists which kind.

### 4. Put your things in groups, and say which

**Why.** Two things can have the same name once small words are dropped ("Mow Lawn" and "Mow the
lawn"). The only fair way out by voice is to ask, and the question needs something to tell them
apart: "the one in Voice Tech, or the one in House?"

**What to do.** Give each thing a group a person would recognize (a project, a list, a room) in the
same answer that gives its name.

### 5. Let a change be partial

**Why.** A spoken change is small ("move it to Saturday"). An API that wants the whole object sent
back makes the assistant read it, copy every field and send it all, and any field it doesn't know
about can break the change (the chore app fails if a chore's labels aren't sent back).

**What to do.** Accept just the fields that change (`PATCH`, or a command per change like
`/chores/{id}/dueDate`).

### 6. Don't require the thing's current state to change it

**Why.** The chore app refuses to move a chore unless the request carries the chore's `updatedAt`,
its last-changed time. A voice request never has it; the assistant must read the chore first, and
a copy even a minute old is refused.

**What to do.** Don't require it. If you need protection against two changes at once, make it
optional, or say plainly in the spec which field is needed and where to get it.

### 7. Keep your addresses stable

**Why.** The chore app answered a create at `/chores` by redirecting to `/chores/`. The redirect
dropped the request, nothing was created, and the assistant very nearly told someone it had been.

**What to do.** Answer at the address your spec lists. If you must redirect, never for a request
that changes something.

### 8. Prove a change happened

**Why.** An assistant must never say "done" unless the app said so. A success that's only "no
error" isn't proof: the redirect in 7 returned no error.

**What to do.** Answer a change with what was changed (the new item with its id, the new due date).

### 9. Say truthfully what can be undone, and make it work

**Why.** "Undo that" is one of the most natural things to say. The chore app's spec promises it can
undo a completion or a skip within 5 minutes; asked to undo a skip made seconds earlier, it answers
400. The assistant can only be honest if the spec is.

**What to do.** List exactly what your undo reverses, and test it through the API, not only your
own screens.

### 10. Give errors a person can hear

**Why.** When something can't be done, the assistant should say why in a sentence ("the chore app
wouldn't let me undo that"), not go quiet or guess.

**What to do.** A short, plain reason with every error, in words, beside the code.

### 11. Take the words as said, if you understand words

**Why.** Some apps are better at their own subject than any general voice system: a weather app
knows "Saturday", "tonight" and "in London" better than a router does.

**What to do.** Offer one door that takes the request as spoken (and the earlier requests, for
"what about Sunday?") and answers with a status and a sentence to speak. Say so in `x-wepwawet:
answers:`, and mark the voice map and clean-up as not applying.

A voice system that hasn't sent a request your way may still ask whether it's yours at all, before
it goes to a cloud model. If your door can tell, take a parameter that says "only tell me whether
it's yours", answer `not_mine` when the words aren't about your subject, and name the parameter in
`answers: claim:`. Left out, a voice system sends you only what it has already decided is yours.

### 12. Get the house's people from the voice system

**Why.** Each app names the same person its own way: a speaker-ID app by its profile ("Chief"), a
chore app by its member ("Chief"), the people around them by what they call them ("Dad"). An app
that keeps its own copy of the household goes stale the day someone is added or renamed.

**What to do.** Don't keep a copy: ask. The voice system keeps the one record of who's who, a row
per person and a column per app, each cell that person's name in that app. An app asks for its own
column, with its own key, and gets each person's name in the app beside the house's name for them;
it never sees another app's column. Say in `x-wepwawet: lists:` which of your lists holds people
(`house: people`), so the voice system knows which column is yours.

### 13. Name each command's starter and its fields

**Why.** A voice request opens with a few words that say what to do ("add a chore", "mark it done",
"move it to"), then gives details. Without the app saying which words open which command, and which
of its fields are needed, every app needs a hand-written voice map and hand-written field names:
the chore app calls a chore's date `nextDueDate` and its person `assignedTo`, the next app something
else, and the voice system can't know which detail to ask for when one is missing.

**What to do.** In `x-wepwawet: commands:`, give each command a person would call by voice its
starter (the words a request opens with) and its fields, required and optional, each under a
standard name the voice system knows (below), beside your own field's name. *(Draft: the standard
names are settled as more apps are taken through.)*

A command reached only through the door of expectation 11 needs none of this: the app reads the
words itself. The starter says which of an app's commands is meant, never which app: that stays
the voice system's own job (its topic words).

### 14. Declare what a person asks for

**Why.** A command is what the app can do; what a person asks for is often a few commands in a row.
"Rename the gutters to clean the gutters" means: list the chores, find the one meant, ask for the new
name if it wasn't said, read the chore as it is now, send it back with the new name, and say so.
Without a way to say that, every voice system builds it by hand for every app. Taking five apps
through, 20 of the 21 things built by hand had one of four shapes.

**What to do.** In `x-wepwawet: actions:`, give each thing a person asks for its starter and its
shape, from its parts:

| Shape | Parts | For |
|---|---|---|
| change | `target`, `value`, `send` (if the app wants the whole item back), `command`, `says` | rename, change a date |
| act | `target`, `confirm`, `command`, `says` | delete, mark done |
| read | `target`, `says` (the item's fields said back) | "when is it due?" |
| list | `from`, `pick` (by a field, by date: the next one, the day said), `says` | "what's next?", "who was born today?" |

- `target` is one item of a list, found by what was said: the voice system matches it, asks which
  when two fit, and checks first when it isn't sure.
- `confirm: {always: true}` asks first even when it's sure. Use it for anything that can't be undone.
- `value: read_back:` says a value back before it's sent ("Rename Get to Me?").
- Anything that fits no shape stays the voice system's own: the household builds it.

An app that has no section can still be described: the voice system writes a **compliant copy** of
the app's document with the section filled in, and keeps it beside the app's own, which is never
changed. `source:` says who wrote it and from which version.

### 15. Say how long your data stays good

**Why.** Some data changes the moment someone acts on it (a chore list), some once a day (what
happened on this day in history), some once a year (a calendar of celebrations), and some by the
minute (the weather). A voice system that doesn't know holds old data, or asks again for no reason.

**What to do.** Say it in `x-wepwawet: data:`:

- `kind: record`: kept by the app; read it live, never hold a copy.
- `kind: transient`, with `pace: slow` (it changes once a day or less, up to a year) or `pace:
  fast`, and `hold_seconds`: how long a copy stays good.

### 16. Say what sound you take in

**Why.** An app that listens takes in more than speech. The sound around it can be told apart by
type, and an app that says which types it handles lets the voice system send it the right sound and
leave the rest.

**What to do.** If your app takes in sound, say so in `x-wepwawet: hears:`, with its frequency range
and the acoustic types it handles, by these standard names:

- **Temporal:** `continuous` (steady: machinery, HVAC, constant traffic), `intermittent` (drops to
  the background and restarts: passing trains, overhead flights), `impulsive` (sudden, sharp, loud: a
  door slamming), `low_frequency` (persistent rumbles and hums, typically under 100 Hz).
- **Propagation path:** `airborne` (through the air: voices, barking, stereos), `structural` (through
  floors, walls and pipes: footsteps upstairs, plumbing).
- **Spectral color:** `white` (equal energy at every audible frequency, 20 Hz to 20 kHz), `pink`
  (3 dB less per octave: heavy rain, waterfalls), `brown` or `red` (6 dB less per octave: distant
  thunder), and `blue`, `violet` and `gray` (higher frequencies, or adjusted for the ear).

## The `x-wepwawet` section

Every part, shown once (one app wouldn't have them all: a speech app has no events and no `/ask`
door):

```yaml
x-wepwawet:
  version: 2                  # this standard's version
  title: Hearing              # how the voice system names the app (else the spec's info: title:)
  spoken: Hearing             # what it's called out loud (expectation 1)
  role: speech                # what it is: speech | answers | app
  data: {kind: record}        # how long its data stays good (expectation 15)
  hears:                      # the sound it takes in (expectation 16)
    range: human voice        #   its frequency range
    temporal: [continuous, intermittent]
    path: [airborne]
  tabs:                       # which parts apply (true) or not (false); left out: true
    commands: true            #   its own commands, from this document
    voice_map: false          #   things said to it, each calling one of its commands
    clean_up: false           #   words stripped from what was said before it's called
    lists: true               #   its lists (people, projects...)
    options: true             #   its own knobs
  auth:                       # how a program sends its key (else the spec's securitySchemes)
    in: header
    name: Authorization
    prefix: "Bearer "
  offers:                     # what it can do for the house, each a switch
    who_spoke:
      title: Who spoke
      description: Names the speaker of each request (speaker ID).
      ways:                   # how it can be had; the first is the usual one
        wyoming: in the audio path
        api: its API
        sensor: a sensor it sets in Home Assistant
  lists:                      # its lists (expectation 3), each with where its things come from
    people:
      from: GET /profiles     #   the command that lists them
      id: name                #   the field that is an item's id
      name: name              #   the field that is an item's name
      house: people           #   its items are people in the house (expectation 12)
    events:
      from: GET /events
      id: key
      name: title
      weight:                 #   how much a thing matters (a draft)
        field: rank           #     your field
        higher: true          #     higher matters more (false: 1 is the top)
        from: 3               #     worth saying from here up
        words: rank_label     #     your field of each level's own words ("Solemnity")
  answers:                    # only for an app that takes the words as said (expectation 11)
    door: GET /ask
    said: text
    context: context
    claim: claim              #   optional: the parameter that asks "is this yours at all?" (not_mine if not)
  commands:                   # each command a person calls by voice (expectation 13; a draft)
    POST /items:
      starts: [add, create, new]   # the words a request for it opens with
      required:               # standard name: your field
        name: title
      optional:
        when: dueDate         #   a date or time
        who: assignee         #   a person in the house (expectation 12)
        group: listId         #   its group (expectation 4)
        where: room           #   a place
  actions:                    # what a person asks for (expectation 14)
    rename:
      starts: [rename]
      target: {from: GET /items, find: {field: title, until: [to]}}
      value: {need: new_name, after: [to], question: "What should I call it?"}
      send: {whole: "GET /items/{id}", set: {title: new_name}}
      command: "POST /items/{id}"
      says: {lines: ["{item} is now {new_name}."]}
```

- A part marked `false` is shown as not applying, never hidden, so every app's layout is the same.
  The voice system believes it: how people phrase a question is the voice system's own job (its
  topic words), one step before any app.
- Offers are off until the household switches them on; a way is picked when there's more than one.
- The standard names (a draft): `name`, `when`, `who`, `group`, `where`, and `weight`. A field
  with none of them keeps your own name; the voice system asks for a missing required field by its
  standard name ("When is it due?"), or by your field's name when it has none.
- `weight` (a draft): many apps carry how much a thing matters (a calendar's rank, a chore's
  priority, an alert's severity, a rating). By voice it decides what's said first and what's said
  at all: the main one first when several fit, only what matters enough, a list in order. The
  number alone doesn't say which way it runs or where it starts to matter, so say both, and give
  each level's own words, so the voice system says "a solemnity", never "rank 6".

## An example: the weather brain

[The weather brain](https://github.com/coderpigsfly/weather_brain) answers questions about the weather and the sun for one house. It's the simplest
kind of app: it understands words itself (expectation 11), so the voice system hands it the request
exactly as heard and speaks its answer.

**Its section** (added to its own `/openapi.json`):

```yaml
x-wepwawet:
  version: 2
  title: Weather brain
  spoken: the weather
  role: answers
  data: {kind: transient, pace: fast}
  tabs: {commands: true, voice_map: false, clean_up: false, lists: false, options: false}
  answers: {door: GET /ask, said: text, context: context, claim: claim}
  commands:
    GET /weather:
      starts: ["what's the weather", will it rain, "how's the weather", forecast]
      optional: {when: when}
    GET /sun:
      starts: [when is sunset, when is sunrise, when does it get dark, when does it get light, how much daylight]
      required: {ask: ask}
      optional: {when: when}
```

**What that tells the voice system:**

- `role: answers` and `answers:` mean: send it the words, don't pick a command. There's one door,
  `GET /ask`, taking `text` (the request as said) and `context` (earlier requests, newest first).
- `claim: claim`: a voice system that hasn't routed a request to the brain can ask
  `GET /ask?text=...&claim=true`, and the brain answers `{"status": "not_mine", "reason": "not_weather"}` unless the words
  are about the weather or the sun ("Tell me a joke" isn't; "Is it nice out?" is).
- `voice_map: false` and `clean_up: false`: there's nothing to map words onto and nothing to strip;
  the brain reads "Do I need an umbrella on Saturday?" itself.
- `lists: false`, `options: false`: no people or projects to match, no knobs.
- `data:` the weather changes by the hour: a voice system may hold an answer only briefly.
- `commands:` is for a voice system that calls commands directly. `when` is a standard name; which
  sun time is asked (`ask`) isn't one, so it keeps the brain's own name.

**A request, start to finish.** Someone asks "When does the sun set?":

1. The voice system's own topic words decide this is about the weather (it knows "sunset", and,
   since speech-to-text splits it, "sun set").
2. It calls `GET /ask?text=When does the sun set?`.
3. The brain answers with a status and a sentence:

   ```json
   {"status": "answered", "speech": "The sun sets at 6:15 this evening."}
   ```

   Or, for something it can't do, still with a sentence, so nothing is made up elsewhere:

   ```json
   {"status": "out_of_reach", "reason": "too_far", "speech": "I can only see seven days out."}
   ```

   Or `{"status": "not_mine"}` when the words weren't really for it ("what's a dark color"), so the
   request goes on.

**What it took to comply:** it used to show FastAPI's default version, `0.1.0` (expectation 2), and
had its town, timezone, forecast grid and climate normals written into its code. Now its version is
set in one place and shown in its spec, and everything about the house is in one file,
`house.json`, that never leaves the house; a new install starts from `house.example.json`.

## An example: a chore app's compliant copy

A chore app with no section of its own, as the voice system wrote it (expectation 14). The app's own
document is untouched; this section sits in the copy beside it. Of the chore app's four things built
by hand, all four are actions here.

```yaml
x-wepwawet:
  version: 2
  title: Chores
  spoken: Vikki
  source: {written_by: Wepwawet Veil, from: "Chores v2.7.1", pushed_back: [delete, find, rename, rename_project]}
  data: {kind: record}
  lists:
    people: {from: "GET /projects/{id}/projectusers", id: id, name: name, house: people}
    project: {from: GET /projects, id: id, name: title}
  actions:
    rename:
      starts: [rename, change the name of]
      target: {from: GET /tasks, find: {field: title, until: [to, as], last: true}}
      confirm: {}
      value: {need: new_name, after: [to, as], question: "What should I call it?",
              read_back: {always: true, question: "Rename {item} to {new_name}?"}}
      send: {whole: "GET /tasks/{id}", set: {title: new_name}}
      command: "POST /tasks/{id}"
      says: {lines: ["{item} is now {new_name}."]}
    delete:
      starts: [delete]
      target: {from: GET /tasks, find: {field: title, last: true}}
      confirm: {always: true, question: "It can't be undone. Delete {item}?"}
      command: "DELETE /tasks/{id}"
      says: {lines: ["{item} is deleted."]}
    find:
      starts: [find, when is, "what's the due date"]
      target: {from: GET /tasks, find: {field: title, last: true}}
      confirm: {}
      says: {lines: ["{item} is due {due}."], fields: {due: due_date}, empty_lines: ["{item} has no due date."]}
```

**What it took:** nothing from the app. The voice system read its rules for the app (the things
the household had built by hand) and wrote each one that had a shape as an action. Across five apps
taken through, 20 of 21 had one; the one left over says a single item at random.

## Apps that follow it

- **[The weather brain](https://github.com/coderpigsfly/weather_brain)** (answers, by its `/ask` door).
- **A speech app** (speech: speech-to-text and who spoke), from its version with `x-wepwawet`.
- **A chore app, a history app and a calendar**, by compliant copies their voice system wrote
  (expectation 14).

An app without the section still works with Wepwawet Veil: every part stays open and the household
fills it in by hand, and Veil writes a compliant copy from what the household built. The chore app, the history app and the calendar were taken through that way;
the chore app's shortfalls are the examples in expectations 5 to 10.

## Later versions

Kept here as the standard grows: what an app needs from the house (its location, its timezone).
The kinds of detail a voice system owns (a person, a date, a project, a place) and which of an
app's fields each fills became expectation 13 (a draft).

---

The Wepwawet standard © 2026 coderpigsfly, licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/): share and adapt it, with credit.
