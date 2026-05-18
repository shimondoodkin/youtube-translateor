# User Intent

let's make an application using flutter. it will play YouTube videos.
currently in vertical mode on phone.
not sure about tablets.
it will work like this.

it will have a list of previously watched videos.
above it and URL to input new video url and ok button. but the main route to open the video is by sharing a video from YouTube to this application.
once it receives a new video it will download the video on a server extract audio, transcribe the video audio using Soniox async.
it will show processing to the user. why does it on server and poll the server for status. just in case if user closes the application it stops pulling when using a load the list again the lead list the loads with statuses of transcription. and user can I pull the list little over gesture or click refresh button you decide on the design.

once user opens an item in the list it shows in the top part, the video, there is a play button in the center of the video. it behaves like in YouTube it appears on click and disappear slowly, and when the paused it is visible,

 then it shows in the bottom part about 1/2 from bottom, it is slowly scrolling  wrapped words transcription like teleprompter.  on the black screen white text words in sans service probably font when the user clicks on the word it has an outline.  it is like a half transparent drawer with a maybe curved side corners on top. which will move over the video once user clicks on it it will nove to about 2/3 from bottom of the screen. inside this teleprompter scrolling pane there will be words. user can click on a word, also video pauses.   on the bottom third of the screen below the teleprompter, there will be translation pane.

in translation pane on the top there will be the word user clicked on the right of it play button which function same as the other play button.
text to speech will play it. tts goes through the server too to eleven labs api.

below the world there will be translated word and a play button that when playing it turns into stop and any other action interrupts the play.
and below it in small text
explanations in the translated language and an example sentence in the original language.
when clicking play on the translator word all the translation information is played the word of the explanation and an example sentence.

and that there will be probably also profile screen where user selects the target language.

user can see his subscription and click to redirect the website with the subscription. on the website there is subscription in PayPal. the subscription is gives you I don't know like 2 hours per day in a month so there is like monthly limit.

there will be postgres database with users table. and the maybe subscription purchases and addons hours table. you can figure out subscription yourself. user will login with Google account with a firebase but it is only used to make login easier in apps

it will be planned for iOS too but currently will be developed and tested for Android only.

make a simple website with subscription.
and make an API server

there will be probably a temp folder.

first write this text into a file called user intent.md

then think through this project
like enhance this prompt and the what to request in another file. this file is not plan yet. just think better what is good.
