# SatGPT has been updated

Back in 2023, when talking to a computer in plain English was still a party trick, I built a small toy called SatGPT. You typed something like "find Sentinel-2 images over Seattle from July 2024" and it went and found them: the satellite scenes, their footprints on a map, a plot of how cloudy each one was, and a viewer where you could flip between true colour and a dozen vegetation and water indices. Then I put it down, and the ground under it quietly moved for three years.

Today it woke up. You can poke at it here: **[kevinlalli.me/satgpt-app](https://kevinlalli.me/satgpt-app/)**. It is a recording of one real conversation, replayed, with all the knobs still live: pick a date, pick NDVI or NBR or any of the others, change the colour map, drag the stretch, hide the clouds. Nothing runs on a server, there is no model behind the chat, and it will not ask you for a key. Type something anyway; it has a polite answer ready.

A few things from the afternoon, for the curious:

- The 2023 code was fine. What had drifted was everything around it: a retired model name, libraries that stopped shipping wheels for the Intel Python this Mac still carries, and one dependency that now wants pandas 2. Thirty-three pins later, it ran.
- The first public tunnel returned nothing but 404s. The culprit was my own laptop: a config file for a different website was quietly redirecting every quick tunnel to a port where nothing lives.
- Given a real key, the model sent half a bounding box, `-122.485,`, twice, identically. The tool now asks for four numbers instead of one string, and newer models stopped improvising.
- Landsat imagery sits in a bucket that charges the reader. SatGPT now says so instead of going quiet.
- A plan to run the whole thing inside your browser got as far as a working spike before the honest answer arrived: the image pipeline will not fit in WebAssembly. The recording you can play with is what replaced it, and it is the better demo.

How the afternoon went, one ask at a time:

<!--FX-->

If you would rather run the real thing, with a live model and your own key, the code and instructions are at [github.com/lalligagger/satgpt-app](https://github.com/lalligagger/satgpt-app), and the original command-line version at [github.com/lalligagger/satgpt](https://github.com/lalligagger/satgpt) works again with current shell-gpt. Both are toys. Good ones, I think.
