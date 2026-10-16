# SatGPT updated

In 2023, with Ivan D'Ortenzio and David H., I built SatGPT, a demo of what a language model could do for satellite imagery, and entered it in Anaconda's inaugural Data App Contest, where it took [first place](https://www.linkedin.com/posts/philippjfr_introducing-satgpt-activity-7093224123950084096-Z-LB). You typed a request in plain English, "find Sentinel-2 images over Seattle from July 2024", and it found the scenes, drew their footprints on a map, plotted the cloud cover of each one, and opened a viewer where you could switch between true colour and a dozen vegetation and water indices. Then I set it aside, and the libraries and models under it kept moving for three years.

This week it runs again, and there is a version you can try without installing anything: **[kevinlalli.me/satgpt-app](https://kevinlalli.me/satgpt-app/)**. It is a recording of one real conversation with the app, replayed, with the viewer's controls left live: choose a date, choose NDVI or NBR or any of the other indices, change the colour map, adjust the stretch, mask the clouds. Nothing runs on a server, there is no model behind the chat, and it will not ask for a key. If you type something of your own it explains that it is a recording and lists what it can show.

A few notes from the afternoon, for the curious:

- The 2023 code needed no changes to run. What had moved was everything around it: a retired model name, libraries that no longer ship wheels for the Intel Python this Mac still carries, and one dependency that now requires pandas 2. Thirty-three version pins later, it started.
- The first public tunnel returned nothing but 404s. The cause was on my own laptop: a configuration file for a different website was redirecting every quick tunnel to a port where nothing was listening.
- Given a real key, the model sent half a bounding box, `-122.485,`, twice, identically. The search tool now takes four numbers instead of one free-text string, and newer models stopped improvising.
- Landsat imagery lives in a bucket that charges the reader, so SatGPT now says so instead of failing silently. Sentinel-2 is free and loads.
- A plan to run the whole app inside the browser got as far as a working prototype before the honest answer arrived: the image pipeline does not fit in WebAssembly. The recording replaced it, and it is the better demo.

How the afternoon went, one ask at a time:

<!--FX-->

To run the real thing, with a live model and your own key, the code and instructions are at [github.com/lalligagger/satgpt-app](https://github.com/lalligagger/satgpt-app). The original command-line version, at [github.com/lalligagger/satgpt](https://github.com/lalligagger/satgpt), works again with current shell-gpt.
