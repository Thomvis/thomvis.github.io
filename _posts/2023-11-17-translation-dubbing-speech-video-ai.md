---
layout: post
title: "Automated translation & dubbing of speech in video using AI"
---

I wanted to watch [a video about robots](https://www.youtube.com/watch?v=8WZoVJIV9V0) with my 5-year-old son. One problem: the video was in English, which he doesn't understand (yet). I could have translated it on the fly as we watched, but since it was evening and he was already in bed, I had time to devise a technical solution.

How hard can it be to automate this? This thought wouldn't have come up if it wasn't for the current AI boom. Seemingly every day, a new AI-powered tool emerges. These tools make tasks, once considered difficult, suddenly very possible. I would just have to tie these tools together to achieve what I wanted. And even the tying together I wouldn't have to do, GPT-4 can do that for me.

I identified the following steps towards reaching the desired result:

1. transcribe the video's speech using automatic speech recognition (ASR)
2. translate the transcription from English to Dutch
3. turn the translated text into speech (TTS)
4. separate the video's audio into background and speech
5. concatenate and mix translated speech and background sounds for resulting video

I used [whisper.cpp](https://github.com/ggerganov/whisper.cpp) to turn the video's audio into a json with all speech text & timing info. The [tinydiarize model](https://github.com/ggerganov/whisper.cpp#speaker-segmentation-via-tinydiarize-experimental) (`ggml-small.en-tdrz.bin`) gave me better results because each speech segment in the output is a speaker turn instead of more granular sentence fragments. While the latter might be better when generating subtitles, having whole speaker turns was convenient for the next step.

I used GPT-4 to translate each speaker turn from English to Dutch. Each speaker turn was a separate request to the OpenAI [Chat Completions API](https://platform.openai.com/docs/guides/text-generation/chat-completions-api). I instructed GPT-4 to keep the translations the same length to fit the video, but I'm not convinced this approach made a significant difference. A more sophisticated approach would be to translate the complete transcription in one go (or in big chunks at least), because it would give GPT-4 more context. Another option would be to use a more specialized translation API (such as [DeepL](https://www.deepl.com/pro-api)), but you wouldn't be able to instruct it to return translations of similar length.

To turn the translated speaker turns into speech, I used the OpenAI [Create speech API](https://platform.openai.com/docs/api-reference/audio/createSpeech). A separate request is made for each turn, resulting in a (short) mp3 for each turn. Initially, I had hoped to use different voices for different speakers in the video, but whisper.cpp [doesn't support](https://github.com/akashmjn/tinydiarize/tree/main#gotchas) speaker clustering.

Having only the generated speech as the only audio in the resulting video would be boring and perhaps even unsettling. I knew I wanted to mix in the original audio somehow. I initially lowered the volume of the original audio tenfold and put the generated speech over it. That's similar to the approach commonly taken on TV (only without the [ducking](https://en.wikipedia.org/wiki/Ducking)). I took it one step further with Meta's [demucs](https://github.com/facebookresearch/demucs). With it I separated the speech from all other sounds in the original video's audio track. This was by far the slowest step of all, but it makes for a more impressive end result.

To combine the generated speech mp3 files and the extracted "karaoke" track with the video I used [ffmpeg, the ultimate Swiss Army knife](https://www.caseyliss.com/2017/8/17/ffmpeg). It was able to extract, convert, concatenate and mix whatever I threw at it. It emitted unintelligible errors while still providing a good result. I did not question it's ways. When the generated speech would exceed the corresponding segment in the video, it is sped up to a max of 1.25x. Another approach could be using the `speed` parameter of the TTS API.

I tied all this together in a simple (in the sense of: hacked together to get the job done) shell script. You can find it at the bottom of this post (or in [this Gist](https://gist.github.com/Thomvis/474e16941959ece4d9d579c3f6e8c706)). It's not really built to be reusable, but it might still prove to be insightful.

Both for shell scripting and ffmpeg invocations, GPT-4 was super useful. Although most of its responses included a mistake or two, it significantly saved time by getting me 80% of the way there. I don't _want_ to become a shell or ffmpeg expert, and thanks to GPT-4 I don't have to.

All in all this took the better part of an evening to build. And while the end result is far from perfect, my son and I enjoyed watching it the next day. You can too (speech starts at 0:22):

<iframe width="630" height="355" src="https://www.youtube.com/embed/aNrS15XlkCk?si=XVjP5ADL_DYDkyAZ" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

## Script
The simple, hacked-togehter-to-get-the-job-done, shell script:

<script src="https://gist.github.com/Thomvis/474e16941959ece4d9d579c3f6e8c706.js"></script>