---
published: true
---
## ListenTogether

March 2020 Covid hit us. Our day to day lives changed, our routines simplified to activities at home. With more free time on my hands, I began to think of a coding challenge for myself. I've been curious about [React Native](https://reactnative.dev/) - I'm always onboard with simplfying development, and having to code once and deploy on Android and iOS is too much enticing for me. 

After some brainstorming I decided to go with an app which would enable Spotify users to listen together and also chat. 

### Architecture
At the foundation, we need to leverage [Spotify's APIs](https://developer.spotify.com/documentation/web-api/). The area of interest are the mobile SDKs (for Android and iOS) which lets us control the installed Spotify App on the user's device. Here's some quick bullet points on what the SDK offers:
- Auth 
- Music player controls (play, skip, set mode)
- Music player metadata
- Music player events
Here's the [javadoc](https://spotify.github.io/android-sdk/app-remote-lib/docs/) for reference.

Since I'm using React Native I started to explore for tools which can help development. Here's what I found for React Native components:
- [react-native-gifted-chat](https://github.com/FaridSafi/react-native-gifted-chat)
- [native-base](https://nativebase.io/)
- [react-native-ui-lib](https://github.com/wix/react-native-ui-lib)

As for other dependencies I found [react-native-spotify-remote](https://github.com/cjam/react-native-spotify-remote/issues) by [cjam](https://github.com/cjam). Huge props to him for starting this! It is a react native wrapper around the Spotify Mobile SDKs. He even gives an example project to test out the capabilities. 

Here's a brief overview of the App's architecture:
![ListenTogether_Architecture.png]({{site.baseurl}}/_posts/ListenTogether_Architecture.png)



As you can see I'm using Firebase's [Realtime Database](https://firebase.google.com/docs/database/) as my backend DB and [Cloud Functions](https://firebase.google.com/docs/functions) for remote logic. It is my first time using Firebase, and I have to say I'm quite impressed with the available features. For any project that is starting out, trying to get their MVP - Firebase is definitely a smart choice.

That's enough for today, I'll share more on the details of my app in the future. Good evening!
