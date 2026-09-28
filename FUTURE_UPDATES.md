# Future Updates

Notes on potential features. Not committed to any of these — just a running list.

| Idea | Notes |
|---|---|
| Export to sheet music via LilyPond | Convert the custom notation into real LilyPond source and render it. |
| Chord quality shorthand (sus, minor, etc.) | Dedicated syntax for chord qualities instead of voicing every note by hand with octave marks. |
| Scale-degree chords (ii, iii, IV, etc.) | Support without changing/overloading the Chord Progression field. |
| Inversions | Notation for voicing a chord in a specific inversion. |
| Interval notation | Express notes/chords as intervals rather than fixed lane positions. |
| Better repeat syntax | `\repeat N { }` works but is clunky — look at how Chord Player handles repeats for a cleaner model. Like not needing to fill in the entire bar is a really cool feature of chord player|
| Read Chord Player Files | Have claude learn how to read chord player files and convert them into this format. |
| Learn more about Chord Player | How does it work with scales? Do I want conditions? Do I want this to play sound? |
| Ties. | So yeah that seems important |
| Bass 1 2 3 instead of only B1 BF etc. | Stranger is broken right now |
|Sustain. | Should the default be that notes are sustained? It looks kinda clunky to have the notes show the full duration instead of just showing when the note is played. Maybe this should be a setting? |